# Optimize DataInput Value Storage and Loading for Large Models

## Problem

Model `15207` exposed a scaling problem in the current `DataInputs` design.

The issue is not that one indicator has all 700MB by itself. The failing case we are focusing on is:

> one model/scenario has many `DataInputs` across many blocks/indicators, and together they add up to hundreds of MB.

For model `15207`, scenario `15180`, the old full-model load pulled:

- `224` DataInput records
- about `774 MB` of `DataInputs.data_values`
- about `6.46M` JSON value rows

The current short-term fix scopes `/outputs/v2` so one block no longer loads unrelated model inputs. That fixes the immediate “load every block’s inputs for one block” problem.

But there is still a deeper storage/design problem: `DataInputs.data_values` is one giant JSON string per indicator/scenario. That makes large real input data hard to batch, stream, index, filter, and load efficiently.

## Current Design

`DataInputs` currently acts as both:

1. metadata/header table
2. value storage table

Current shape:

```text
DataInputs
- id
- indicator_id
- scenario_id
- type
- dimensions
- data_values
- input_adjustment
- date_created
```

In practice:

one indicator + one scenario = one `DataInputs` row

Example:

Block: P4W Data
Indicator: PL Data
Scenario: Baseline
DataInputs row: 499153

`data_values` is a giant JSON string containing many value rows.

`dimensions` only tells us the shape:
This input is by Cost Centre, Branch, Accounts, Time

`data_values` stores the actual values:

```json
{
  "40048": "Admin",
  "40064": "Apr-26",
  "40078": "Beyond Corporate",
  "40055": "Profit Costs",
  "value": 0
}
```

### Current Query Limitation

Today we can batch by `DataInput` rows:

```sql
SELECT
  di.id,
  di.indicator_id,
  di.dimensions,
  di.data_values
FROM "DataInputs" di
WHERE di.scenario_id = $1
  AND di.indicator_id = ANY($2);
```

That means we can do:

- Batch 1: 25 DataInput rows
- Batch 2: 25 DataInput rows
- Batch 3: 25 DataInput rows

But every batch still returns JSON blobs.
The backend still has to:

- receive JSON strings
- parse JSON strings
- create Python dict/list objects
- build dataframe/calc payload

So this helps memory pressure a bit, but it does not solve the main design limitation.

### What Current Design Cannot Do Efficiently

Because actual values are hidden inside `data_values`, the database cannot naturally:

- batch by individual value rows
- stream 50k value rows at a time
- index individual values
- filter by dimension item before loading
- build dataframe/Arrow/Polars-style columns directly from DB rows
- avoid JSON string parsing
- avoid large JSONB aggregation limits

In simple terms:

Postgres sees one big blob, not millions of normal rows.
So it cannot cleanly do:

- Give me the first 50,000 value rows.
- Then the next 50,000.
- Then the next 50,000.

It can only return the full blob for each `DataInputs` row.

## Proposed Redesign

Keep `DataInputs` as metadata only.

```text
DataInputs
- id
- indicator_id
- scenario_id
- type
- dimensions
- input_adjustment
- date_created
```

Move actual values into a new row-based table.

```text
DataInputValues
- id
- data_input_id
- scenario_id
- time_item_id
- coord_1_item_id
- coord_2_item_id
- coord_3_item_id
- coord_4_item_id
- value
```

`DataInputValues.data_input_id` links back to `DataInputs.id`.
`DataInputs.dimensions` explains what the coordinate columns mean.

Example:

`DataInputs.dimensions = [Branch, Team, Employee]`

Then:

- `coord_1_item_id` = Branch item
- `coord_2_item_id` = Team item
- `coord_3_item_id` = Employee item

Example value row:

- `data_input_id`: `502422`
- `scenario_id`: `15180`
- `time_item_id`: `Apr-26`
- `coord_1_item_id`: `Branch A`
- `coord_2_item_id`: `Team X`
- `coord_3_item_id`: `Employee Mia`
- `value`: `120`

## Improved Query Flow

Instead of returning JSON blobs, we query actual values as rows.

```sql
SELECT
  id,
  data_input_id,
  scenario_id,
  time_item_id,
  coord_1_item_id,
  coord_2_item_id,
  coord_3_item_id,
  coord_4_item_id,
  value
FROM "DataInputValues"
WHERE scenario_id = $1
  AND data_input_id = ANY($2)
  AND id > $3
ORDER BY id
LIMIT 50000;
```

This gives real value-level batching.

Example:

- Batch 1: 50,000 value rows
- Batch 2: next 50,000 value rows
- Batch 3: next 50,000 value rows

This works even if all periods are needed and even if all values are non-zero.

## Why This Is Better

The new design lets us:

- load only required `DataInput` IDs
- batch actual values, not JSON blobs
- stream large inputs safely
- index by scenario/input/time/coordinate
- avoid Postgres JSONB aggregation limits
- avoid giant Python JSON parsing
- build calculation payloads from row/column data directly
- support sparse storage where missing means zero
- support large models even when the data is genuinely large

## Areas That Need Updating

### Metadata and output calculation

- `modelAPI/services/model_metadata_cache_v4.py`
- `modelAPI/services/model_metadata_cache.py`
- `modelAPI/resources/block_kpi_v4_rust.py`
- `modelAPI/resources/block_kpi_v4.py`
- `modelAPI/resources/block_kpi_v3.py`
- `modelAPI/resources/new_block_kpi.py`
- `modelAPI/calc_engine/rust_bridge.py`
- Rust input loading under `modelAPI/omni-calc/src/engine/exec/steps/input.rs`

These should stop depending on `DataInputs.data_values` blobs and load values through a row-based value service.

### Data input APIs and editing

- `modelAPI/models/data_inputs.py`
- `modelAPI/resources/data_inputs.py`
- `modelAPI/calc_engine/data_inputs.py`
- `modelAPI/calc_engine/data_inputs_v3.py`
- `modelAPI/validators/data_input.py`

These currently read, parse, mutate, and save `data_values` JSON. They need to read/write `DataInputValues` rows instead.

Important: dummy zero grids should be display-only. They should not be persisted as dense zero rows.

### Import and forecast write paths

- `modelAPI/services/forecast_writer.py`
- `modelAPI/services/forecast_import_service.py`
- `modelAPI/services/import_sync.py`
- `modelAPI/services/actuals_import_writer.py`
- `modelAPI/calc_engine/actuals.py`

These should write/upsert/delete value rows instead of rebuilding a giant JSON blob.

### Copy and scenario flows

- `modelAPI/models/data_inputs.py`
- `modelAPI/models/model_scenario.py`
- `modelAPI/models/indicators.py`
- `modelAPI/scripts/sync_model_missing_from_source_db.py`

Copying an input, indicator, scenario, or model must copy both:

- `DataInputs` metadata row
- `DataInputValues` value rows

### Dimension rename/delete flows

- `modelAPI/resources/dimensions.py`

These currently inspect/update `data_values` JSON. With row-based values, dimension item changes should update coordinate item IDs or rely on IDs so string rewrites are not needed.

### Dashboard/model metadata

- `modelAPI/models/blocks.py`
- `modelAPI/models/models.py`
- `modelAPI/repositories/blocks.py`
- `modelAPI/repositories/indicators.py`

Metadata routes should continue using lightweight metadata only and must not accidentally reload raw values.

## Suggested New Service Layer

Add a small repository/service boundary so callers do not touch storage details directly.

Example methods:

- `get_data_input_metadata_for_indicators(scenario_id, indicator_ids)`
- `stream_values_for_data_inputs(scenario_id, data_input_ids, batch_size=50000)`
- `replace_values(data_input_id, rows)`
- `upsert_values(data_input_id, rows)`
- `delete_values_for_scope(data_input_id, scenario_id, filters)`
- `copy_values_to_scenario(source_scenario_id, target_scenario_id)`
- `copy_values_to_indicator(source_indicator_id, target_indicator_id)`
- `materialize_display_grid(data_input_id)`

This keeps the redesign contained and reduces regression risk.

## Migration Plan

1. Add `DataInputValues` table.
2. Backfill from existing `DataInputs.data_values`.
3. Store only non-zero rows where missing-value-as-zero is valid.
4. Verify row counts, non-zero counts, and calculation output parity.
5. Add dual-read support behind a feature flag.
6. Move calculation/output reads to `DataInputValues`.
7. Move edit/import/save/copy flows to `DataInputValues`.
8. Keep old `data_values` temporarily for compatibility.
9. Remove or deprecate old blob reads after validation.

## Acceptance Criteria

- `/outputs/v2` can calculate large models without loading unrelated inputs.
- Large required input sets can be loaded in value-row batches.
- Backend no longer depends on giant JSONB aggregation for `DataInput` values.
- Calculation results remain unchanged.
- Missing value rows are treated as zero where valid.
- Forecast/import/edit/save flows write to the new value table.
- Copy scenario/model/indicator flows copy values correctly.
- Dashboard/model metadata routes do not load raw value rows.
- Existing API response shapes remain compatible or have a planned migration path.
- Tests cover calculation parity, sparse values, imports, forecast writes, copy flows, and dimension changes.

## Testing Required

- Compare output results before/after migration for model `15207`.
- Verify block output where only a small subset of model inputs is required.
- Verify block output where many/all inputs are required.
- Verify imports still write correct values.
- Verify clear-before-import deletes the correct value rows.
- Verify forecast edits update only changed values.
- Verify scenario copy/model copy preserves values.
- Verify missing rows calculate as zero.
- Verify dashboard/model metadata does not load value rows.
- Verify performance with batch sizes such as 10k, 50k, and 100k rows.

## Summary

The current short-term fix stops loading all model inputs for one block.
This ticket is the longer-term storage redesign.
The main change is:

- Before: `DataInputs` row contains one giant `data_values` JSON blob.
- After: `DataInputs` stores metadata.
- After: `DataInputValues` stores actual values as queryable rows.

That gives us proper batching, indexing, streaming, and safer calculation loading for large models.
