# CFO Demo Jira Bug Report 

## Issue Inventory

| Ticket | Priority | Area | Summary |
| --- | --- | --- | --- |
| CFO-BUG-001 | P0 | Model settings | Saved start/end period and duration can reload incorrectly. |
| CFO-BUG-002 | P2 | Model settings | Currency default and currency selector behavior are inconsistent. |
| CFO-BUG-003 | P1 | Formatting | Percentage and ratio values display with inconsistent scaling. |
| CFO-BUG-004 | P1 | Builder / planner state | Indicator format and roll-up changes do not reliably refresh dependent views. |
| CFO-BUG-005 | P0 | Block categories | Newly created blocks do not reliably appear in the selected category immediately. |
| CFO-BUG-006 | P2 | Block cards | Block cards can render indicator objects as `[object Object]`. |
| CFO-BUG-007 | P2 | CSV import | Uploaded CSV display names and S3 names are mangled. |
| CFO-BUG-008 | P1 | CSV import | Multi-period time detection can fail and block import progress. |
| CFO-BUG-009 | P3 | CSV import UX | Import wizard needs manual mapping, header-row, and row-skip controls. |
| CFO-BUG-010 | P1 | Plan inputs | Yearly input grid can become unstable while editing values. |
| CFO-BUG-011 | P2 | Plan inputs layout | Yearly input panel is too narrow to show expected year columns. |
| CFO-QA-012 | P1 | Regression coverage | Add automated coverage for the demo-critical workflows above. |

---

## CFO-BUG-001 - Model settings save/reload corrupts start period and duration

Type: Bug  
Priority: P0  
Severity: Critical  
Components: Model settings, model time properties, model overview header, model API

### Problem Statement

When a model is configured with a monthly start period such as `Jan-25` and an end period such as `Oct-28`, reopening the settings can show the wrong year/date range. In affected demo flows this can shift the model timeline, which then cascades into plan tables, charts, and CSV import period detection.

### Context / Background

The source report calls out the EFM Masterclass demo model and describes a saved range reloading with an incorrect start year. Code inspection found both a frontend parsing hazard and an API duration-contract inconsistency.

### Current Behavior

- The model settings modal can parse `MMM-YY` strings through JavaScript `Date` before applying an explicit month-year parser.
- In JavaScript, strings like `Jan-25` can be interpreted as January 25 of the current/default year rather than January 2025.
- The frontend sends `plan_duration_periods` as a period difference, while the backend stores `raw + 1` and some serializers return `stored - 1`.
- Different API representations expose different duration values, which increases the chance of an off-by-one reload.

### Expected Behavior

- A saved range of `Jan-25` through `Oct-28` must reopen exactly as January 2025 through October 2028.
- The generated `time_range` must use the same start/end dates after save, refresh, and modal reopen.
- Duration should have one documented inclusive/exclusive contract across the frontend and backend.

### Steps to Reproduce

1. Open a monthly model in the CFO demo environment.
2. Open model settings.
3. Set the start period to `Jan-25` and the end period to `Oct-28`.
4. Save the model settings.
5. Refresh the model overview and reopen model settings.
6. Compare the displayed start/end periods and generated timeline with the saved values.

### Technical Analysis / Root Cause

Primary root cause: `traction-react/src/pages/ModalOverViewPageHeaderV1/components/ModelSettingsModal.tsx` uses `new Date(period)` in `parseDateFromPeriod` before explicitly parsing `MMM-YY`. In local Node testing with the app timezone context, `new Date('Jan-25')` resolves as a day-based date in year 2001-era parsing behavior rather than January 2025. This is unsafe for the app's own `MMM-YY` period format.

Contributing root cause: `modelAPI/resources/models.py` normalizes `plan_duration_periods` by adding one in `update_time_properties_core`, while `modelAPI/models/models.py` subtracts one in `json()` but exposes raw duration in `full_json()`. This makes it easy for the frontend to display or resubmit the wrong duration depending on which endpoint populated state.

Backend partial mitigation exists: `modelAPI/services/util.py` now includes `parse_month_year_period` / `parse_calendar_period`, and `modelAPI/tests/test_parse_calendar_period.py` covers `Jan-25`, `Feb-25`, `Dec-27`, and `Oct-28`. The frontend modal still needs equivalent safe parsing.

### Affected Modules / Files / Endpoints

- `traction-react/src/pages/ModalOverViewPageHeaderV1/components/ModelSettingsModal.tsx`
- `traction-react/src/pages/ModalOverViewPageHeaderV1/components/ModelSettingsDatePicker.tsx`
- `modelAPI/resources/models.py`
- `modelAPI/models/models.py`
- `modelAPI/services/util.py`
- `modelAPI/tests/test_parse_calendar_period.py`
- Endpoint: model time properties update/read flow, including `ModelTimeProperties`

### User Impact

Critical for demos and production models. A wrong timeline makes periods, charts, data inputs, imports, and calculations appear incorrect even if the underlying business logic is otherwise valid.

### Acceptance Criteria

- Saving `Jan-25` through `Oct-28` reopens as the same periods after refresh.
- Frontend parsing handles `MMM-YY`, `MMM-YYYY`, ISO dates, and yearly values deterministically without falling through to ambiguous `new Date(string)` parsing.
- `plan_duration_periods` has a documented contract and is normalized at a single boundary.
- `time_range`, settings modal fields, and persisted model properties agree after save.
- Regression coverage exists for monthly and yearly settings, including off-by-one duration cases.

### QA Notes

Validate in at least one model with monthly granularity and one model with yearly granularity. After changing the model range, verify downstream import period detection still sees the new `time_range`.

### Open Questions / Assumptions

- Assumption: the intended product behavior is an inclusive visible range from start to end.
- Open question: should API responses expose both stored duration and display duration, or should all callers receive only one normalized value?

---

## CFO-BUG-002 - Currency defaults and currency selector are inconsistent

Type: Bug  
Priority: P2  
Severity: Medium  
Components: Model settings, currency formatting, model creation

### Problem Statement

The model currency can default to an unexpected value instead of USD, and the currency dropdown behavior is not reliable enough for demo or production settings changes.

### Context / Background

The source report describes a wrong default currency and poor selector scrolling. Code inspection shows the default is split across frontend index-based logic and backend model defaults.

### Current Behavior

- The frontend initializes the selected currency with `CURRENCY_OPTIONS[5]` when no persisted match exists.
- The backend model constructor defaults `reporting_currency` to the pound symbol.
- New model creation does not pass an explicit reporting currency, so backend defaults can win.
- The selector has a fixed-height scroll menu, but the UX remains fragile for a long currency list and depends on the current option order.

### Expected Behavior

- New CFO demo models should default to USD unless the template or user explicitly selects another reporting currency.
- Existing models should reopen with the exact persisted reporting currency.
- The dropdown should be searchable or easily scrollable without accidental page scrolling or hidden options.

### Steps to Reproduce

1. Create or open a model that does not explicitly carry a reporting currency.
2. Open model settings.
3. Observe the selected currency.
4. Open the currency dropdown and scroll through the list.
5. Select USD, save, refresh, and reopen model settings.

### Technical Analysis / Root Cause

Primary root cause: the frontend fallback in `ModelSettingsModal.tsx` is index-based (`CURRENCY_OPTIONS[5]`) rather than value-based. If the options array changes order, the fallback changes silently.

Contributing root cause: `modelAPI/models/models.py` defaults `reporting_currency` to the pound symbol. `modelAPI/resources/models.py` creates models without passing a reporting currency, so the backend default is used unless a caller overrides it.

The currency menu code includes a `MenuList` with `maxH` and `overflowY`, which is a partial UX improvement, but the underlying default mismatch remains.

### Affected Modules / Files / Endpoints

- `traction-react/src/pages/ModalOverViewPageHeaderV1/components/ModelSettingsModal.tsx`
- `modelAPI/models/models.py`
- `modelAPI/resources/models.py`
- Currency constants in `traction-react/src/projectConstants`
- Endpoint: model create/update and model time/settings read flows

### User Impact

Users can unknowingly build or present a model in the wrong currency. This undermines trust in financial dashboards, KPI cards, and formatted plan outputs.

### Acceptance Criteria

- Default currency is defined by one product-owned constant or server setting, not an array index.
- New CFO demo models default to USD unless a template overrides it.
- Model settings reload exactly the persisted reporting currency.
- Currency dropdown supports reliable scrolling and selection for the full currency list.
- Currency changes propagate to currency-formatted indicators after save.

### QA Notes

Test model creation, duplicated/copied models, and existing models with non-USD currency. Confirm currency-formatted indicators update after changing the setting.

### Open Questions / Assumptions

- Assumption: USD is the desired CFO demo default based on the source report.
- Open question: should backend defaults be changed globally, or should the CFO demo template explicitly set USD?

---

## CFO-BUG-003 - Percentage and ratio indicators display with inconsistent scaling

Type: Bug  
Priority: P1  
Severity: High  
Components: Plan outputs, dashboard KPI/table, charts, shared formatting

### Problem Statement

Percentage and ratio indicators can display incorrect values in plan tables, KPI cards, and charts because different frontend paths disagree about whether a stored value is a decimal fraction or an already-scaled percentage.

### Context / Background

The source report flags incorrect values for percentage/ratio indicators. Code inspection found multiple independent formatting paths with different scaling assumptions.

### Current Behavior

- Some API service methods multiply percentage outputs by `100` after fetching block outputs.
- Shared formatter comments say ratio/percentage values arrive as decimal fractions, but the code returns the numeric value with a `%` suffix without multiplying by `100`.
- Grid display paths multiply values by `100` in some places and not in others.
- Chart tooltip and axis callbacks apply percent-specific multiplication based on `measure === '%'`, which can double-scale or under-scale depending on the source data.

### Expected Behavior

- One value contract should exist across the app: either percentages are stored/transmitted as decimals and formatted at render time, or they are stored/transmitted as percentage points.
- A decimal value such as `0.355` should display as `35.5%` everywhere.
- A percentage-point value such as `35.5` should not display as `3550%`.

### Steps to Reproduce

1. Open a model with an output indicator formatted as percentage or ratio.
2. Compare the same indicator in the plan table, plan chart tab, dashboard KPI card, and dashboard table.
3. Change the indicator format or refresh the model.
4. Observe whether the displayed values remain consistent across all views.

### Technical Analysis / Root Cause

Primary root cause: percentage scaling is duplicated and inconsistent.

Key examples:

- `traction-react/src/services/index.ts` multiplies percentage block output values by `100` in `get_block_outputs`, `get_block_outputs_V2`, and compare-output flows.
- `traction-react/src/utils/FormatValues.ts` documents that percentage/ratio values are decimal fractions, but `getDataTypeBasedValuewithNewValue` appends `%` to the raw numeric value instead of multiplying by `100`.
- `traction-react/src/pages/PlanPageDetailsOPZ/Inputs/IndicatorPageMultiDim/MultiDimensionTableOpzV2.utils.ts` multiplies non-dimension cell values by `100` when `isPercentage === 'percentage'`.
- `traction-react/src/pages/PlanPageDetailsOPZ/Inputs/IndicatorPageMultiDim/StableTextCellTemplate.tsx` renders percentage cells as `cellText.toFixed(1)%` with no knowledge of whether `cellText` was already scaled.
- `traction-react/src/utils/getChartOptions.ts` applies additional percent-specific logic in chart callbacks.

### Affected Modules / Files / Endpoints

- `traction-react/src/services/index.ts`
- `traction-react/src/utils/FormatValues.ts`
- `traction-react/src/utils/getChartOptions.ts`
- `traction-react/src/utils/dashboardChartOptions.ts`
- `traction-react/src/pages/PlanPageDetailsOPZ/Plan/IndicatorRow.tsx`
- `traction-react/src/pages/PlanPageDetailsOPZ/Plan/ChartTabs.tsx`
- `traction-react/src/pages/ModelOverviewPage/Dashboard-v2/kpi/KpiCard.tsx`
- `traction-react/src/pages/ModelOverviewPage/Dashboard-v2/table/DashboardTableRow.tsx`
- `traction-react/src/pages/PlanPageDetailsOPZ/Inputs/IndicatorPageMultiDim/*`
- Endpoints returning block outputs and dashboard output data

### User Impact

High. A CFO user may see margin, conversion, ratio, or rate metrics that are off by a factor of `100`, or that disagree across surfaces. This creates demo risk and can lead to wrong business interpretation.

### Acceptance Criteria

- Define and document one percentage value contract for API responses and frontend state.
- Remove duplicate ad hoc `* 100` conversions from fetch services or move them into a single formatter boundary.
- Plan table, dashboard KPI, dashboard table, chart tooltip, and chart axis all display the same value for the same percentage indicator.
- Add unit tests for `0`, `0.08`, `0.355`, `1`, negative percentages, and already-formatted strings.
- Add an integration/regression test that compares the same percentage indicator across at least two UI surfaces.

### QA Notes

Use a known indicator with a predictable result, such as gross margin at `35.5%`. Validate the raw API payload and all UI render paths before and after refresh.

### Open Questions / Assumptions

- Assumption: backend calculation outputs should remain numeric and unformatted; formatting should happen at the UI boundary.
- Open question: do existing saved models contain percentage inputs in decimal form, percentage-point form, or both?

---

## CFO-BUG-004 - Indicator format and roll-up changes do not refresh dependent views reliably

Type: Bug  
Priority: P1  
Severity: High  
Components: Builder metadata, plan outputs, Redux state

### Problem Statement

Changing an indicator's format, roll-up type, output format, or chart metadata does not reliably refresh the builder table, plan table, KPI cards, and charts. Users can save a metadata change and still see stale display state.

### Context / Background

The source report describes format and roll-up changes not reflecting reliably. Code inspection found the fulfilled reducer for format updates only updates a subset of Redux state, while related data-view updates are commented out.

### Current Behavior

- `UpdateIndicatorFormat.fulfilled` updates `state.indicators` and `state.selectedIndicators`.
- The reducer does not update `state.indicatorsData`.
- The reducer does not rebuild `state.dimensionalIndicators`.
- Some refetch paths in the indicator table are commented out, so views can retain stale selected indicator/table data after metadata changes.

### Expected Behavior

- After saving a format, roll-up, output-format, or chart-type change, all dependent UI surfaces should update immediately or after one explicit refetch.
- The builder property table, output list, selected indicator panel, plan table, KPI cards, and charts should agree.

### Steps to Reproduce

1. Open a block in builder mode.
2. Select an indicator used in the plan or dashboard.
3. Change its format to percentage or currency, or change roll-up/display-as-subtotal metadata.
4. Save the change.
5. Navigate between builder, plan, KPI, and chart views.
6. Observe whether stale values, stale format labels, or stale chart formatting remain.

### Technical Analysis / Root Cause

Primary root cause: incomplete Redux update after metadata mutation.

`traction-react/src/redux/PlannerModeSlice.ts` has two related reducers:

- `UpdateIndicator.fulfilled` updates output indicators, selected indicators, `indicatorsData`, and rebuilds dimensional output rows.
- `UpdateIndicatorFormat.fulfilled` updates only output indicators and selected indicators. The `indicatorsData` and `dimensionalIndicators` update logic exists but is commented out.

Contributing cause: `FetchBlockOutputs.fulfilled` also leaves selected-indicator synchronization commented out, making stale selected state more likely after refetch.

### Affected Modules / Files / Endpoints

- `traction-react/src/redux/PlannerModeSlice.ts`
- `traction-react/src/pages/BuilderMode/Toolbar.tsx`
- `traction-react/src/pages/BuilderMode/TableHelpers.tsx`
- `traction-react/src/pages/BuilderMode/propertyTableMetadata/buildPropertyTableGridRows.ts`
- `traction-react/src/pages/BuilderMode/IndicatorPage/index.tsx`
- Backend indicator update endpoint in `modelAPI/resources/indicators.py`

### User Impact

High. Users believe they changed a metric definition or display format, but dependent views can show old state. This is especially risky during demos because the same metric can appear differently across panels.

### Acceptance Criteria

- `UpdateIndicatorFormat.fulfilled` updates every state collection that depends on indicator metadata.
- Dimensional indicator rows are rebuilt after format and roll-up changes.
- Selected indicator state is refreshed or invalidated after format changes.
- The UI either updates optimistically with the server response or performs one deterministic refetch.
- Regression tests cover format change, roll-up change, subtotal/display-output change, and chart type change.

### QA Notes

Test both dimensioned and non-dimensioned indicators. Include at least one indicator rendered in dashboard KPI/table and at least one indicator rendered in a plan chart.

### Open Questions / Assumptions

- Assumption: the backend response from the indicator update endpoint is authoritative and can be used to normalize frontend state.
- Open question: should the format update reducer be merged with the broader `UpdateIndicator` reducer logic to avoid divergence?

---

## CFO-BUG-005 - Block created from a selected category does not appear in that category immediately

Type: Bug  
Priority: P0  
Severity: Critical  
Components: Model overview, block categories, block creation

### Problem Statement

When a user creates a new block while viewing a category, the new block does not reliably appear in that selected category immediately after creation. Users can interpret the create action as failed or lose the block in another tab/category.

### Context / Background

The source report marks this as P0 for the CFO demo. Code inspection shows the create payload can include the selected category, and the backend can persist it, but the frontend state update is incomplete after creation.

### Current Behavior

- `AddNewBlockMenu` derives `selectedCategory` from the currently selected category tab and includes `model_category_id` in the create payload when available.
- The backend validates that the category belongs to the model and assigns `block.model_category_id`.
- After create, the Redux reducer replaces only the `blocks` array with `action.payload.blocks`.
- Category metadata, such as counts and possibly category list state, is preserved from stale state.
- No guaranteed `FetchModelBlocks` refresh runs after create.

### Expected Behavior

- Creating a block from a selected category should immediately show that block in the same category view.
- Category block counts and card category labels should update at the same time.
- The "All" and "Others" views should remain consistent with the selected category view.

### Steps to Reproduce

1. Open a model with at least one block category.
2. Select a category tab.
3. Create a new block from the add-block menu.
4. Save the block.
5. Observe whether the new block appears in the selected category without manual refresh or tab switching.
6. Check the category block count and the block card category tag.

### Technical Analysis / Root Cause

Primary root cause: frontend state after block creation is only partially refreshed.

Relevant code:

- `traction-react/src/pages/ModelOverviewPage/blockv1/AddNewBlockMenu/index.tsx` correctly attempts to include `model_category_id` for a selected category.
- `modelAPI/resources/blocks.py` accepts and validates `model_category_id` during block creation.
- `traction-react/src/redux/ModelsSlice.ts` handles `AddModelBlock.fulfilled` by setting `state.blocks = { ...currentBlocks, blocks: action.payload.blocks }`. This updates the block list but not category metadata.
- `traction-react/src/pages/ModelOverviewPage/blockv1/BlockList.tsx` filters blocks by `selectedCategoryId`; if the response shape, selected category, or stale state is mismatched, the new block can be hidden until a full refresh.

### Affected Modules / Files / Endpoints

- `traction-react/src/pages/ModelOverviewPage/blockv1/AddNewBlockMenu/index.tsx`
- `traction-react/src/pages/ModelOverviewPage/blockv1/BlockList.tsx`
- `traction-react/src/pages/ModelOverviewPage/blockv1/index.tsx`
- `traction-react/src/redux/ModelsSlice.ts`
- `modelAPI/resources/blocks.py`
- `modelAPI/models/blocks.py`
- Endpoint: create model block / model block list

### User Impact

Critical in demo and authoring workflows. Users can create duplicate blocks, assume data was lost, or present an incomplete model category.

### Acceptance Criteria

- Creating a block from a selected category persists `model_category_id`.
- The created block appears immediately in that selected category.
- Category counts update immediately.
- The block card category label is correct after create.
- A full model block fetch or complete response normalization occurs after create.
- Regression coverage verifies create-from-category, create-from-All, and create-from-Others behavior.

### QA Notes

Test with categories that have zero blocks and categories that already have blocks. Also test after refreshing the page to confirm persistence, not only optimistic UI state.

### Open Questions / Assumptions

- Assumption: the intended behavior is to default the create modal category to the currently selected category tab.
- Open question: when the selected view is "All", should the modal require an explicit category or allow uncategorized creation?

---

## CFO-BUG-006 - Block cards render indicator objects as `[object Object]`

Type: Bug  
Priority: P2  
Severity: Medium  
Components: Model overview, block cards, block API contract

### Problem Statement

Block cards can display `[object Object]` instead of readable indicator names. This makes cards look broken and reduces trust in the model overview.

### Context / Background

The source report describes block card text rendering as `[object Object]`. Code inspection found inconsistent API shapes for block indicators and a frontend fallback formatter that only handles some object shapes.

### Current Behavior

- Some model/block payloads expose `indicators` as strings.
- Other block payloads expose `indicators` as objects with metadata.
- The block card description fallback attempts to stringify indicator values.
- If an indicator object does not match the expected fields, it can render as `[object Object]` or blank.

### Expected Behavior

- Block cards should display readable indicator names or a safe empty/fallback message.
- Raw objects should never be rendered directly in card text.
- API responses should use a consistent indicator shape, or the frontend should normalize all accepted shapes before rendering.

### Steps to Reproduce

1. Open the model overview for a model containing blocks with indicators.
2. Find a block card whose description is empty or depends on indicator fallback text.
3. Observe the indicator preview text on the card.
4. Compare cards loaded from model overview payloads versus block-list payloads.

### Technical Analysis / Root Cause

Primary root cause: inconsistent API data shape.

- `modelAPI/models/models.py` serializes block indicators as a list of indicator names in `ModelsModel.json()`.
- `modelAPI/models/blocks.py` serializes block indicators as objects in `BlocksModel.json()`.
- `traction-react/src/pages/ModelOverviewPage/blockv1/BlockCard.tsx` uses `getBlockIndicatorDisplayText(indicators)` as fallback description text.
- `traction-react/src/pages/ModelOverviewPage/blockv1/blockIndicatorUtils.ts` now filters known invalid strings and handles object fields like `name` and `indicator_name`, but nested or unexpected objects can still fail.

### Affected Modules / Files / Endpoints

- `traction-react/src/pages/ModelOverviewPage/blockv1/BlockCard.tsx`
- `traction-react/src/pages/ModelOverviewPage/blockv1/blockIndicatorUtils.ts`
- `modelAPI/models/models.py`
- `modelAPI/models/blocks.py`
- Model overview and block-list endpoints

### User Impact

Medium. The model overview appears unpolished and unreliable, especially in a CFO demo where cards are the first summary of model structure.

### Acceptance Criteria

- No block card renders `[object Object]`, `undefined`, or `null` as user-visible text.
- The frontend has a single normalization helper for indicator preview text.
- Backend block summaries expose a documented indicator contract.
- Unit tests cover string indicators, object indicators, nested/unknown objects, empty arrays, and missing descriptions.

### QA Notes

Test cards for newly created blocks, existing blocks, imported/library blocks, and blocks with no description.

### Open Questions / Assumptions

- Assumption: card fallback text should list indicator names when the block description is empty.
- Open question: should the backend standardize on string names for card summaries and reserve full objects for detail endpoints?

---

## CFO-BUG-007 - Uploaded CSV names are mangled and can include `undefined`

Type: Bug  
Priority: P2  
Severity: Medium  
Components: CSV import, S3 upload, file display names

### Problem Statement

CSV upload display names and storage names are altered unexpectedly. Spaces are removed, special characters can become `undefined`, and imported files are uploaded as generic `example.csv`, making files hard to recognize and audit.

### Context / Background

The source report mentions mangled uploaded CSV names and `undefined` in display names. Code inspection found multiple filename sanitizers with inconsistent character maps.

### Current Behavior

- `AddSourceModal` sanitizes the original filename for display but removes spaces by joining name parts with no separator.
- Special-character replacement uses a regex/map mismatch; characters matched by the regex but missing from the map can become `undefined`.
- The parsed CSV blob is uploaded as a new `File` named `example.csv`, so the S3 key does not preserve the original filename.
- `AddVersionModal` repeats the generic `example.csv` upload pattern for source versions.

### Expected Behavior

- The UI should display a readable sanitized version of the original filename.
- S3 object keys should include a safe, traceable form of the original filename.
- Sanitization should never emit the literal string `undefined`.
- Source versions should retain enough original filename metadata for users to distinguish uploads.

### Steps to Reproduce

1. Upload a CSV with spaces and special characters in the filename, for example `Revenue Plan (FY25) - v1.csv`.
2. Complete the source upload flow.
3. Observe the displayed source name and S3 key/url.
4. Add a version of the same source.
5. Compare the version filename and display name.

### Technical Analysis / Root Cause

Primary root cause: filename sanitization is duplicated and not total.

- `traction-react/src/pages/ModelOverviewPage/Import/AddSourceModal.tsx` has a local `replaceSpecialChars` map that does not cover every character matched by its regex. It also joins split name parts without a separator.
- `traction-react/src/pages/ModelOverviewPage/Import/AddVersionModal.tsx` creates uploaded blobs as `example.csv`.
- `traction-react/src/services/s3Service.ts` has a separate sanitizer with a different map and a malformed/broad regex.
- `handleDataFileUpload` builds the S3 key from the provided file name, which is already `example.csv` in the import modal flows.

### Affected Modules / Files / Endpoints

- `traction-react/src/pages/ModelOverviewPage/Import/AddSourceModal.tsx`
- `traction-react/src/pages/ModelOverviewPage/Import/AddVersionModal.tsx`
- `traction-react/src/services/s3Service.ts`
- Import source and import version create endpoints
- S3 upload path for `blox-data-import`

### User Impact

Medium. Users cannot easily identify uploaded files or distinguish versions. This also weakens auditability and support diagnosis for failed imports.

### Acceptance Criteria

- Replace duplicate sanitizers with one shared filename sanitizer.
- Preserve file extension and a readable sanitized base name.
- Use separators instead of collapsing words together.
- Sanitizer has complete coverage for all matched characters and never returns `undefined`.
- S3 key includes timestamp plus sanitized original filename.
- Source/version display names preserve readable original context.

### QA Notes

Test filenames with spaces, parentheses, ampersands, slashes, hyphens, percent signs, repeated spaces, unicode characters, and very long names.

### Open Questions / Assumptions

- Assumption: it is acceptable to sanitize storage keys while separately preserving the original display name.
- Open question: should original filenames be stored as metadata in addition to sanitized S3 keys?

---

## CFO-BUG-008 - CSV multi-period time detection can fail and block import

Type: Bug  
Priority: P1  
Severity: High  
Components: CSV import, time-period mapping, model time range

### Problem Statement

The CSV import wizard can fail to detect valid multi-period time columns/rows and block the user from continuing, even when the file contains recognizable periods such as `Jan-25`, `Feb-25`, and later months.

### Context / Background

The source report describes a multi-period CSV import detection failure. Code inspection shows the parser has been improved, but the wizard still blocks progress if no auto-detected row maps to the current model time range.

### Current Behavior

- The time-period step builds selectable row options only when `selectedVersion.s3_file_url`, `headerNames`, and `modelTime.time_range` are available.
- Candidate rows are filtered to only rows where at least one value maps into the model time range.
- If detection returns no candidates, the Continue button is disabled and the wizard shows a no-valid-time-period-rows message.
- There is no manual override in the time-period step.

### Expected Behavior

- A CSV with period headers/rows such as `Jan-25`, `Feb-25`, ..., `Oct-28` should be detected when those periods exist in the model timeline.
- If auto-detection fails, users should be able to manually select the row/columns and map periods.
- Detection failure should provide actionable detail, not a dead end.

### Steps to Reproduce

1. Configure a model with monthly periods that should match the import file.
2. Upload a CSV containing multiple monthly period labels.
3. Proceed to the time-period mapping step.
4. Observe whether period rows are detected and Continue is enabled.
5. Repeat after changing model settings to verify that model timeline state affects detection.

### Technical Analysis / Root Cause

Primary root cause: the import flow depends entirely on auto-detection against `modelTime.time_range`.

- `traction-react/src/pages/ModelOverviewPage/Import/ActionTwoSteps/Columns.tsx` disables progression when `rowOptions.length === 0`.
- `traction-react/src/pages/ModelOverviewPage/Import/utils/timeRowOptions.ts` filters candidates to rows with at least one period that maps to `modelTimeRange`.
- `traction-react/src/pages/ModelOverviewPage/Import/utils/timePeriodParser.ts` has robust support for `MMM-YY`, ISO-like dates, Excel serials, and yearly periods, but it still requires the parsed value to match the current model timeline.

Contributing cause: if CFO-BUG-001 corrupts model time settings, valid CSV labels can fail to match because the model timeline itself is wrong.

### Affected Modules / Files / Endpoints

- `traction-react/src/pages/ModelOverviewPage/Import/ActionTwoSteps/Columns.tsx`
- `traction-react/src/pages/ModelOverviewPage/Import/utils/timeRowOptions.ts`
- `traction-react/src/pages/ModelOverviewPage/Import/utils/timePeriodParser.ts`
- Model time properties endpoint used by the import wizard
- S3 CSV retrieval/parsing flow

### User Impact

High. Users cannot complete data imports even when the file contains valid period labels. This blocks onboarding and demo data setup.

### Acceptance Criteria

- Detect `Jan-25`, `Feb-25`, `Dec-27`, `Oct-28`, ISO dates, Excel serial dates, and yearly labels when they belong to the model range.
- Continue is enabled when a valid period row/column set is selected.
- Auto-detection failure offers manual mapping instead of blocking the flow.
- Error text identifies whether the failure is due to no period-like labels or labels outside the model time range.
- Regression tests include a multi-period CSV spanning the CFO demo range.

### QA Notes

Test with the model range fixed from CFO-BUG-001. Use both header-row periods and first-column periods. Include a negative test where periods are genuinely outside the model range.

### Open Questions / Assumptions

- Assumption: period labels outside the model range should not import silently.
- Open question: should the wizard allow importing a subset of detected periods when some file periods are outside the model range?

---

## CFO-BUG-009 - Import wizard lacks manual time mapping, header-row, and row-skip controls

Type: Bug / UX Enhancement  
Priority: P3  
Severity: Low to Medium  
Components: CSV import, mapping wizard UX

### Problem Statement

The import wizard depends too heavily on automatic structure detection. Users need explicit manual controls for time mapping, header row selection, and row skipping when CSV files do not match the expected format exactly.

### Context / Background

The source report requests manual mapping plus row-skip/header controls. Code inspection shows some manual dimension mapping exists, but equivalent manual time-period controls are missing from the time-period step.

### Current Behavior

- Dimension mapping has a manual path in parts of the two-step import flow.
- The time-period step only offers detected row candidates.
- If detection fails, the wizard disables Continue.
- Header toggles exist in some import steps, but there is no comprehensive "use row N as headers" or "skip first N rows" control across the full flow.

### Expected Behavior

- Users should be able to manually identify where time periods live in the file.
- Users should be able to choose the header row and skip metadata/comment rows.
- The wizard should preserve and preview mapping decisions before import execution.

### Steps to Reproduce

1. Upload a CSV with one or more title/comment rows before the real header.
2. Upload a CSV where time labels are not in the auto-detected row.
3. Attempt to continue through the import wizard.
4. Observe that the user cannot explicitly choose the correct row/columns when auto-detection fails.

### Technical Analysis / Root Cause

The import UI is optimized around auto-detection and has partial manual controls only in selected steps.

Relevant paths:

- `traction-react/src/pages/ModelOverviewPage/Import/ActionTwoSteps/Columns.tsx`
- `traction-react/src/pages/ModelOverviewPage/Import/ActionTwoSteps/DataType.tsx`
- `traction-react/src/pages/ModelOverviewPage/Import/ActionTwoSteps/DimensionType.tsx`
- `traction-react/src/pages/ModelOverviewPage/Import/ActionTwoSteps/selectDimensionTwosteps/DimensionSelectionSteps.tsx`

This ticket is closely related to CFO-BUG-008 but should remain separate because it is a product/UX capability, not only a parser defect.

### Affected Modules / Files / Endpoints

- Import wizard React components under `traction-react/src/pages/ModelOverviewPage/Import/ActionTwoSteps/`
- CSV parsing helpers under `traction-react/src/pages/ModelOverviewPage/Import/utils/`
- Import preview/run endpoints that receive mapping configuration

### User Impact

Medium for real-world customer files and low-to-medium for controlled demos. Without manual controls, otherwise valid CSVs can require external editing before import.

### Acceptance Criteria

- Add a manual time-period mapping mode.
- Add header-row selection with preview.
- Add row-skip controls with preview.
- Persist mapping choices through the wizard state and import request.
- Allow users to return to previous steps without losing manual selections.
- Provide clear validation for duplicate periods, missing required periods, and periods outside model range.

### QA Notes

Test clean CSVs, files with title rows, files without headers, files with blank rows, files with extra notes, and files where time periods are stored across columns versus down rows.

### Open Questions / Assumptions

- Assumption: manual controls should complement auto-detection rather than replace it.
- Open question: should saved import mappings be reusable for later versions of the same source?

---

## CFO-BUG-010 - Yearly input grid can become unstable while editing values

Type: Bug  
Priority: P1  
Severity: High  
Components: Plan input grid, yearly inputs, ReactGrid editing

### Problem Statement

The yearly input grid can crash, wrap unexpectedly, or become unstable while users edit yearly values. This affects `constant_by_year` planning workflows where CFO users expect fast spreadsheet-like editing.

### Context / Background

The source report describes yearly input grid crash/wrap behavior. Code inspection did not include a runtime stack trace, so the exact crash condition is inferred from the grid editing and layout code.

### Current Behavior

- Yearly inputs use a dynamic ReactGrid configuration with responsive column width calculations.
- Editing commits local cell changes, dispatches a bulk data-input update, then may refetch block outputs.
- Errors from update are swallowed in the catch block.
- Numeric display paths convert edited text to numbers without strong validation, which can create `NaN` display states.
- Layout recalculates based on container width while the input grid is embedded in a narrow side panel.

### Expected Behavior

- Editing yearly values should not crash the grid.
- Values should commit once, remain visible, and persist after refresh.
- Invalid numeric input should be rejected or shown with inline validation.
- The grid should retain stable column widths while editing.

### Steps to Reproduce

1. Open a plan input that uses yearly input mode (`constant_by_year`).
2. Open the side input panel.
3. Edit one or more year values.
4. Press Enter or blur the field.
5. Repeat across adjacent year columns and observe whether the grid wraps, resizes, or crashes.
6. Refresh and verify persistence.

### Technical Analysis / Root Cause

Likely root causes, inferred from static analysis:

- `traction-react/src/pages/PlanPageDetailsOPZ/Inputs/IndicatorPageMultiDim/MultiDimensionTableOpzV2.tsx` applies local cell changes after the async update and refetches outputs for dimensioned blocks. UI state can change from local updates and refetches in quick succession.
- The catch block in `updateCellData` intentionally swallows errors, hiding failed updates and making the grid appear unstable without user feedback.
- `StableTextCellTemplate.tsx` converts display text through `Number(cell.text)` and formatting branches without strong validation for blank or invalid numeric edits.
- `MultiDimensionTableOpzV2.utils.ts` handles `constant_by_year` columns differently and generates payloads by splitting column IDs that combine time dimension ID and year. This path is sensitive to column ID format.

### Affected Modules / Files / Endpoints

- `traction-react/src/pages/PlanPageDetailsOPZ/Inputs/IndicatorPageMultiDim/MultiDimensionTableOpzV2.tsx`
- `traction-react/src/pages/PlanPageDetailsOPZ/Inputs/IndicatorPageMultiDim/MultiDimensionTableOpzV2.utils.ts`
- `traction-react/src/pages/PlanPageDetailsOPZ/Inputs/IndicatorPageMultiDim/StableTextCellTemplate.tsx`
- `traction-react/src/pages/PlanPageDetailsOPZ/Inputs/IndicatorPageMultiDim/NewCell.tsx`
- Data input update endpoint used by `UpdatePlangPageIndicatorTableData`
- Backend data-input validation in `modelAPI/resources/data_inputs.py` and `modelAPI/validators/data_input.py`

### User Impact

High. Yearly plans are a core CFO workflow. Grid instability can cause perceived data loss and prevents confident planning edits during demos.

### Acceptance Criteria

- Editing yearly cells does not crash or wrap unexpectedly.
- Invalid values show validation and do not corrupt local grid state.
- Failed saves surface a user-visible error and restore or preserve the last known good value.
- Local optimistic updates and refetches cannot overwrite each other with stale data.
- Regression tests cover editing yearly values with Enter, blur, rapid edits, invalid input, and refresh persistence.

### QA Notes

Run tests with dimensioned and non-dimensioned yearly inputs. Include narrow side-panel width and full-width table width. Use browser console monitoring to capture any ReactGrid or validation errors.

### Open Questions / Assumptions

- Assumption: the original "crash" refers to a frontend runtime/grid failure rather than a backend 400/500.
- Open question: is there a known stack trace or browser console error from the demo session that can identify the exact failing branch?

---

## CFO-BUG-011 - Yearly input panel is too narrow to show expected year columns

Type: Bug  
Priority: P2  
Severity: Medium  
Components: Plan input layout, side panel, yearly grid

### Problem Statement

The yearly input panel is too narrow to comfortably show the expected number of year columns. The source report states that the panel should show three full year columns.

### Context / Background

This is related to CFO-BUG-010 but should be fixed independently because layout constraints alone can make the input experience unusable even when editing logic is stable.

### Current Behavior

- In plan mode, the input panel can be constrained to a narrow percentage width.
- Yearly column widths are calculated from the current container width.
- The `constant_by_year` width calculation has a minimum column width, but the surrounding panel can still be too narrow to show three full years plus fixed fields.
- Users must horizontally scroll or see clipped/wrapped year columns.

### Expected Behavior

- The yearly input panel should show at least three full year columns, plus required row labels/actions, at the target desktop demo viewport.
- When the viewport cannot support this, the UI should use a stable horizontal scroll area rather than wrapping or squeezing cells.

### Steps to Reproduce

1. Open a plan input with yearly values.
2. Use the side input panel layout.
3. Observe visible year columns at the standard demo viewport.
4. Resize the browser or toggle the side panel expansion.
5. Confirm whether three full year columns remain visible.

### Technical Analysis / Root Cause

Primary root cause: container constraints and column sizing are not aligned with the product requirement.

- `traction-react/src/pages/BuilderMode/IndicatorPage/index.tsx` uses a `30%` width for the input area in plan-page mode when expanded.
- `traction-react/src/pages/PlanPageDetailsOPZ/Inputs/IndicatorPageMultiDim/MultiDimensionTableOpzV2.utils.ts` calculates `constant_by_year` column widths from available container width after subtracting fixed fields.
- `MultiDimensionTableOpzV2.tsx` recalculates columns with a `ResizeObserver`, so narrow panel width directly affects the number of visible year columns.

### Affected Modules / Files / Endpoints

- `traction-react/src/pages/BuilderMode/IndicatorPage/index.tsx`
- `traction-react/src/pages/PlanPageDetailsOPZ/Inputs/IndicatorPageMultiDim/MultiDimensionTableOpzV2.tsx`
- `traction-react/src/pages/PlanPageDetailsOPZ/Inputs/IndicatorPageMultiDim/MultiDimensionTableOpzV2.utils.ts`
- Related SCSS/CSS for plan input table containers

### User Impact

Medium. Users can still edit with scrolling, but the workflow feels cramped and less spreadsheet-like. This is highly visible in demos.

### Acceptance Criteria

- At the target desktop viewport, the yearly input panel shows at least three complete year columns.
- Column widths remain stable while editing.
- Narrow viewports use horizontal scrolling without wrapping cells.
- No text or inputs overlap in yearly mode.
- Visual regression or Playwright screenshot coverage verifies desktop and smaller viewport layouts.

### QA Notes

Validate with 3-year, 4-year, and 5-year models. Test both monthly models shown as fiscal years and yearly-granularity models.

### Open Questions / Assumptions

- Assumption: "three full year columns" means three editable year value columns, excluding dimension/name columns.
- Open question: what is the official target demo viewport width?

---

## CFO-QA-012 - Add automated regression coverage for CFO demo bug flows

Type: QA / Test Coverage  
Priority: P1  
Severity: High  
Components: Frontend tests, API tests, import tests, regression suite

### Problem Statement

The CFO demo regressions are not adequately covered by automated tests. Several issues are cross-surface bugs where a backend/API fix alone or a UI-only fix alone would not prevent recurrence.

### Context / Background

The source report requests automated regression coverage. Code inspection found useful backend coverage for calendar period parsing, but limited frontend/integration coverage for the affected workflows.

### Current Behavior

- Backend tests exist for `parse_calendar_period` and related model time parsing.
- There is no clear frontend regression coverage for model settings date reload, currency default selection, block create-in-category, import time detection, or plan yearly-grid editing.
- Percentage formatting is spread across service, formatter, table, KPI, and chart code paths without a single comprehensive test matrix.

### Expected Behavior

- Each high-risk CFO demo workflow has automated coverage at the correct layer.
- Unit tests cover deterministic helpers.
- Integration or browser tests cover cross-component flows.
- Regression tests run in CI and fail on the known demo failure modes.

### Steps to Reproduce

This is a coverage task rather than a runtime defect. Reproduce by reviewing current test coverage for each ticket above and confirming no automated test fails when the described regression is reintroduced.

### Technical Analysis / Root Cause

The affected functionality spans multiple boundaries:

- Model settings: frontend date parsing plus backend time-property persistence.
- Percent formatting: service response mutation plus shared formatter plus UI renderers.
- Block categories: modal state plus API persistence plus Redux normalization.
- Import: CSV parsing plus model time range plus wizard state.
- Yearly grid: responsive layout plus editing state plus data-input update API.

Coverage is currently strongest around backend parsing and weakest around frontend state/UX flows.

### Affected Modules / Files / Endpoints

Recommended coverage targets:

- `modelAPI/tests/test_parse_calendar_period.py`
- New backend tests for model time-property round trips.
- New frontend unit tests for `FormatValues`, `timePeriodParser`, `timeRowOptions`, filename sanitization, and block indicator normalization.
- New frontend integration/browser tests for model settings, block creation by category, import time mapping, and yearly-grid editing.

### User Impact

High. Without regression coverage, demo-critical workflows can break again during unrelated refactors.

### Acceptance Criteria

- Add model settings round-trip tests for `Jan-25` through `Oct-28`.
- Add currency default tests for new model and existing model settings.
- Add percent/ratio formatter tests with a documented value contract.
- Add block create-in-category test that verifies immediate card visibility and category count.
- Add block card tests that prevent `[object Object]`, `undefined`, and `null` rendering.
- Add CSV filename sanitizer tests.
- Add import time-detection tests for monthly and yearly files, including no-match fallback.
- Add yearly-grid edit tests for successful save, failed save, invalid value, and narrow layout.
- Ensure these tests run in CI.

### QA Notes

Prioritize P0/P1 coverage first: CFO-BUG-001, CFO-BUG-003, CFO-BUG-004, CFO-BUG-005, CFO-BUG-008, and CFO-BUG-010. Add lower-priority visual/UX coverage after functional regressions are guarded.

### Open Questions / Assumptions

- Assumption: frontend tests can use the existing React test setup or Playwright/Cypress if already present in the CI environment.
- Open question: should the CFO demo model IDs from the source report be converted into reusable fixtures, or should tests build isolated models dynamically?

---

## Cross-Ticket Implementation Notes

- Fix CFO-BUG-001 before validating CFO-BUG-008, because import time detection depends on the model timeline.
- Fix CFO-BUG-003 and CFO-BUG-004 together if possible, because percentage formatting and stale indicator metadata can mask each other.
- For CFO-BUG-005 and CFO-BUG-006, prefer normalizing block API response shapes rather than adding more component-specific string coercion.
- For CFO-BUG-007 and CFO-BUG-009, centralize import parsing/sanitization helpers so source upload, version upload, and wizard preview use the same data.
- For CFO-BUG-010 and CFO-BUG-011, validate with both functional tests and screenshots because layout instability is part of the user-visible defect.
