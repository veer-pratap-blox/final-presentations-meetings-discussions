# Optimize Large-Model Loading by Scoping DataInput and Metadata Queries

## Problem

While analysing model `15207`, we found several large-model performance issues in the current loading architecture.

Model `15207` makes the problem very visible because it has large dimensions and very large `DataInput.data_values`, but the underlying issue is not specific to this model. The same patterns can affect other large or complex models as they grow.

The main issue is that several active APIs load more data than the page or action actually needs.

The most serious case is:

`POST /block/{block_id}/outputs/v2`

When opening the Revenue Planning block output in model `15207`, the backend creates `ModelMetadataCacheV4` before calculation. That cache currently loads all `DataInputs` for the selected model and scenario, instead of only loading the inputs needed for the requested block.

For model `15207`, this means a Revenue Planning output request loads unrelated large inputs from other blocks.

From the analysis:

- Revenue Planning itself is around `1.32 MB`
- P4W Data is around `182 MB`
- Utilization is around `577 MB`

So a single Revenue Planning request can pull hundreds of MB of unrelated input payloads before calculation starts.

This explains why the request fails before Rust/omni-calc can return useful timing. The Rust timing report is empty because the failure happens during metadata/data loading, before calculation finishes.

There are also related metadata-loading problems in the model overview and Dashboard V2 setup flows. These pages return simple-looking metadata, but the backend builds those responses by walking ORM relationships or by making repeated per-dimension calls.

This ticket should fix the architecture pattern, not just patch model `15207`.

---

## Affected Active App Flows

### 1. Model Overview Page Load

**Frontend**

`ModelOverviewPage/index.tsx` dispatches `FetchModelBlocks`.

**API**

`GET /model/{model_id}`

**Backend**

`Model.get -> ModelsModel.json()`

**Current Issue**

`ModelsModel.json()` builds the response by walking ORM relationships for:

- Dimensions
- Categories
- Blocks
- Block indicators
- Driver checks
- Model category data

The response is a summary, but the backend performs relationship-based work to build it.

---

### 2. Dashboard V2 Setup / Page Load

**Frontend**

`useDashboardData` dispatches `DashboardSliceV2.FetchModelBlocks`.

**APIs**

- `GET /model/{model_id}/blocks`
- `GET /model/{model_id}/dimensions`
- `GET /dimension/{dimension_id}/items`

**Backend**

- `BlockList.get -> BlocksModel.json()`
- `DimensionList.get`
- `DimensionItemList.get`

**Current Issue**

Dashboard V2 uses `/model/{model_id}/blocks`, which calls `block.json(scenario_id)` for every block.

Inside `block.json()`, the code loops through indicators and accesses `ind.input_id` to read input dimensions.

Because `ind.input_id` is a normal SQLAlchemy relationship to `DataInputModel`, the row includes `data_values`, even though Dashboard setup only needs metadata like dimensions.

Dashboard V2 also calls `/dimension/{dimension_id}/items` once per dimension. The backend already has a bulk endpoint, so this should be batched.

---

### 3. Plan / Builder Block Output Load

**Frontend**

`fetchAnalysisSectionDataVersion2 / FetchBlockOutputsV2`

**API**

`POST /block/{block_id}/outputs/v2`

**Backend**

`BlockKPIRouter -> BlockKPINewV4Rust -> ModelMetadataCacheV4`

**Current Issue**

`ModelMetadataCacheV4` query 10 builds one large JSONB object containing `DataInput.data_values` for every indicator in every block of the selected model/scenario.

Python then parses that huge JSON object and stores it in `_data_inputs_cache`.

Individual indicators read from that cache later, but the expensive whole-model load has already happened.

**Current query behaviour**

```sql
WHERE b.model_id = $1
AND di.scenario_id = $2
```

This scopes by model and scenario, but not by requested block or required indicators.

---

## Out of Scope

### `/model/{model_id}/blocks-indicators`

This route is not part of the current frontend flow.

The current frontend breakpoint has this route commented out and uses `/model/{model_id}/blocks` instead.

It can be optimized separately if it becomes active again, but it should not be the main target for this ticket.

### `/model/{model_id}/blocks_full`

This route is also not part of the current active initial page/dashboard flow and should not be used for initial page load.

---

## Required Redesign

The core fix is to make data loading match app intent.

- Opening the model overview should load model summary only.
- Opening Dashboard setup should load dashboard metadata only.
- Opening one block output should load only the data required to calculate that block.
- Large `DataInput.data_values` should only be loaded after the backend knows which indicators are required.

### `/outputs/v2` Redesign

Create a scoped output metadata path for block output calculation.

Do not use a single metadata cache path that always loads all model/scenario `DataInputs`.

#### New Flow

1. Receive `POST /block/{block_id}/outputs/v2`
2. Find block and check permission
3. Load lightweight model/scenario metadata
4. Build dependency graph for the requested block
5. Resolve required source blocks and required input indicator IDs
6. Load `DataInputs` only for those required indicator IDs and selected scenario
7. Build Rust/omni-calc payload
8. Run Rust/omni-calc
9. Build pivot/output response
10. Return the same response shape expected by the frontend

### Replace Whole-Model JSONB Aggregation

Replace the current whole-model JSONB aggregation with a scoped row query.

**Current Pattern**

```sql
WHERE b.model_id = $1
AND di.scenario_id = $2
```

**Improved Pattern**

```sql
WHERE di.scenario_id = $1
AND di.indicator_id = ANY($2)
```

Instead of asking Postgres to build one giant JSONB object for the whole model, return normal rows for the required indicators and build the cache map in Python.

**Example Loading Contract**

```python
def load_data_inputs_for_indicators(self, indicator_ids: list[int]) -> None:
    rows = fetch_data_input_rows(self.scenario_id, indicator_ids)
    grouped = {}

    for row in rows:
        grouped.setdefault(str(row["indicator_id"]), []).append(dict(row))

    self._data_inputs_cache.update(grouped)
```

This keeps the existing `get_data_inputs_for_indicator()` style usable, but the cache contains only scoped data.

### Dashboard Metadata Redesign

Keep `/model/{model_id}/blocks` compatible, or introduce a dedicated endpoint:

`/model/{model_id}/dashboard-blocks`

The response should include only the metadata Dashboard V2 needs:

- Blocks
- Indicators
- Block dimensions
- Connected dimensions
- Indicator input dimensions
- Basic dimension metadata

It must not load `DataInput.data_values`.

For indicator input dimensions, fetch only:

- `indicator_id`
- `dimensions`
- `scenario_id`

Do not use `ind.input_id` inside `block.json()` for Dashboard metadata.

Instead, use a batched query that reads only the required columns.

### Dashboard Dimension Item Redesign

Replace repeated per-dimension calls:

```text
GET /dimension/{dimension_id}/items
GET /dimension/{dimension_id}/items
GET /dimension/{dimension_id}/items
...
```

With the existing bulk endpoint:

```text
GET /dimensions/items/bulk?dimension_ids=1,2,3&scenario_id=15180
```

The frontend already has:

`apiService.get_dimension_items_bulk()`

So `DashboardSliceV2` should use that instead of `Promise.all` over `get_dimension_items()`.

### Model Overview Redesign

Replace `ModelsModel.json()` relationship walking with batched summary queries.

`GET /model/{model_id}` should keep the same response shape, but build it from grouped queries:

- Model core row
- Dimensions
- Categories with block counts
- Blocks
- Indicator names by block
- Driver flags by block
- Model category map
- Base scenario
- Time properties

**Example Grouped Queries**

```sql
SELECT model_category_id, COUNT(*)
FROM "Blocks"
WHERE model_id = $1
GROUP BY model_category_id;
```

```sql
SELECT block_id, array_agg(name ORDER BY position) AS indicators
FROM "Indicators"
WHERE block_id = ANY($1)
GROUP BY block_id;
```

```sql
SELECT bd.block_id, true AS has_driver_dimension
FROM "BlockDimensions" bd
JOIN "Dimensions" d ON d.id = bd.dimension_id
WHERE bd.block_id = ANY($1)
AND d.driver_page = true;
```

This avoids walking ORM relationships block by block and category by category.

---

## Acceptance Criteria

### `/outputs/v2`

- `POST /block/42405/outputs/v2` does not load `DataInputs` for unrelated blocks.
- A Revenue Planning request does not load `P4W Data.data_values` unless `P4W Data` indicators are part of the dependency graph.
- A Revenue Planning request does not load `Utilization.data_values` unless `Utilization` indicators are part of the dependency graph.
- Rust/omni-calc timing is present when calculation runs.
- The frontend response shape remains compatible.

Add timing around:

- Lightweight metadata load
- Dependency scope build
- Scoped `DataInputs` load
- Rust payload build
- Rust execution
- Pivot response build

### Dashboard V2

- Dashboard setup loads block metadata without querying or returning `DataInput.data_values`.
- Dashboard setup loads dimension items through the bulk endpoint.
- Block, indicator, dimension, connected dimension, and filter selectors keep existing behaviour.
- Connected dimension names and indicator names still resolve correctly.

### Model Overview

- `GET /model/15207` returns the same model overview data shape.
- The response is built from batched/grouped queries instead of ORM relationship walking.
- It does not load `DataInput.data_values`.
- Category counts, indicator names, driver flags, block metadata, model category data, base scenario, and time properties remain correct.

---

## Testing

Add tests or instrumentation to prove:

- Revenue Planning `/outputs/v2` only loads required `DataInput` indicator IDs.
- Unrelated `P4W Data` and `Utilization` inputs are not loaded for Revenue Planning unless the dependency graph requires them.
- Dashboard block metadata does not load `DataInput.data_values`.
- Dashboard bulk dimension item loading returns the same normalized data as current per-dimension calls.
- `GET /model/{id}` summary response matches the existing response shape.
- Rust timing appears after scoped metadata/data loading succeeds.

---

## Expected Impact

This issue was discovered through model `15207`, but the improvement applies across large models, complex models, and models that grow over time.

For model `15207`, the biggest improvement is removing unrelated `DataInput.data_values` from a single block output request.

The Revenue Planning request should no longer pull hundreds of MB of unrelated `P4W Data` and `Utilization` payloads before calculation.

For other models, this reduces the chance that block output, dashboard setup, or model overview loading becomes slower as more blocks, dimensions, indicators, scenarios, and input values are added.

Dashboard V2 setup should avoid loading raw input values and reduce many dimension-item calls into one bulk request.

Model overview should avoid relationship-walking work and build the same summary response from batched queries.

Overall, the backend should stop loading the whole library when the user only asks for one chapter. Each page/API should load only the data required for that specific app action.
