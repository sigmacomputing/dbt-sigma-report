# dbt-sigma-report

Compose [Sigma reports](https://help.sigmacomputing.com/docs/manage-reports-as-code) from dbt macros instead of hand-writing Sigma's JSON. A report is a dbt model: it can `ref()` your dbt models and `source()`s, it can read the tables of your [Sigma data models](https://github.com/sigmacomputing/dbt-sigma-data-model), it is reviewed like the rest of your project, and its compiled spec lands in `manifest.json` on `dbt parse`.

```sql
-- models/sigma_sales_dashboard.sql
{% set spec = sigma_report.simple_dashboard(
    name='Sales at a glance',
    subtitle='Bookings and orders, year to date',
    source=sigma_report.from_ref('stg_orders'),
    kpis=[
      {'value': 'Sum([Amount])', 'name': 'Revenue', 'format': sigma_report.format_currency()},
      {'value': 'Count()', 'name': 'Orders'},
    ],
    charts=[
      {'type': 'bar',   'x': 'region',  'y': 'Sum([Amount])', 'name': 'Revenue by region'},
      {'type': 'donut', 'category': 'segment', 'value': 'Sum([Amount])', 'name': 'Revenue by segment'},
    ],
    table_columns=['order_id', 'order_date', 'region', 'segment', 'amount'],
) %}

{{ sigma_report.materialize(spec) }}

select 1 as report
```

That is a title, a KPI band, two charts and a detail table, laid out on US Letter pages, in the Sigma look, from one model.

The package never calls Sigma. It composes JSON at parse time (no warehouse connection needed) and stores it on the model's `meta`. An optional script, [`scripts/export_reports.py`](#publishing-to-sigma), turns the manifest into files or pushes them to the Sigma API.

## Reports are not workbooks

A Sigma **report** is the paginated-document object: fixed pages (US Letter, A4, ...), elements placed at absolute pixel positions, headers and footers that repeat on every page, built for exporting and scheduling. A **workbook** is the interactive canvas (24-column grid, controls, tabs). The two have different code representations (`POST /v2/reports` with `kind: "report"` versus `POST /v2/workbooks`), and this package targets reports. If you want workbooks, see its sibling [`dbt-sigma-workbook`](https://github.com/sigmacomputing/dbt-sigma-workbook); the macro vocabulary is deliberately similar.

Because a report is paginated, the package lays elements out for you with a simple vertical flow (title, KPI band, charts, table) and starts a new page when one fills up. See [Layout](#layout).

## Requirements

- dbt `>=1.6.0, <3.0.0`. Tested on dbt 1.12 and on dbt 2.0 (the Rust engine) in CI; see [dbt versions](#dbt-versions).
- Any adapter. The macros emit JSON, not SQL.
- To publish: a Sigma organization with reports as code (Beta), a Sigma connection to your warehouse, and an API client for the export script.

## Installation

Not on the dbt Package Hub yet; install from git in `packages.yml`:

```yaml
packages:
  - git: "https://github.com/sigmacomputing/dbt-sigma-report.git"
    revision: v1.0.0          # a release tag; pin it for reproducible builds
```

Then `dbt deps`. To read from Sigma data model tables, also install [`sigmacomputing/dbt-sigma-data-model`](https://github.com/sigmacomputing/dbt-sigma-data-model) (optional).

## Setup

Set project variables in `dbt_project.yml`:

```yaml
vars:
  sigma_connection_id: "<your-sigma-connection-id>"   # Sigma connection to your warehouse
  sigma_folder_id: "<your-sigma-folder-id>"           # folder new reports are created in
```

| Variable | Used for | Required |
| --- | --- | --- |
| `sigma_connection_id` | Warehouse sources. Or pass `connection_id=` per source. | When reading from the warehouse |
| `sigma_folder_id` | Where the report is created. Or pass `folder_id=` per report. | To create (not update) a report |
| `sigma_data_model_ids` | `{dbt_model_name: sigma_data_model_id}` for [`from_data_model`](#sources). | When reading from data models |
| `sigma_report_theme` | The project-wide [theme](#theming). | No |

`sigma_connection_id` and `sigma_folder_id` are the same names `dbt-sigma-data-models` and `dbt-sigma-workbook` use, so a project using several configures them once. Find the ids in Sigma (the connection's page; `GET /v2/files?typeFilters=folder` for folders).

## Quickstart

1. Create `models/sigma_sales_dashboard.sql` from the example above (adjust the `ref` and the columns).
2. Run `dbt parse`.
3. The report is in `target/manifest.json` at `nodes` -> `model.<project>.sigma_sales_dashboard` -> `config.meta.sigma_report`.
4. Write it out: `python dbt_packages/sigma_report/scripts/export_reports.py --out sigma_reports/`, or push it ([below](#publishing-to-sigma)).

Things worth knowing about the model:

- `materialize()` sets the model's `meta` via `config(meta=...)`, which replaces `meta` wholesale, so keep report models single-purpose.
- The body (`select 1 as report`) is a throwaway. By default the model is a view; `materialize(spec, materialized='ephemeral')` creates nothing in the warehouse.
- Pair the model with an [`exposures:`](integration_tests/models/exposures.yml) entry so `dbt ls -s +exposure:sales_dashboard` lists everything the report depends on.
- Column names in formulas are the names Sigma shows: `Sum([Amount])`, `[Order Date]`. A bare lowercase name such as `'order_date'` is title-cased the way Sigma title-cases warehouse columns.

## Layout

A report has **pages**. A page stacks **blocks** top to bottom, and a block is a small tree:

| Block | Is |
| --- | --- |
| an element | a chart, table, text, ... (full width of where it sits) |
| `row([...])` | children side by side, sharing the width (equally, or by `widths=` weights) |
| `stack([...])` | children one above another |
| `at(el, x, y, width, height)` | an element pinned at an absolute position, outside the flow |
| `page_break()` | start a new physical page here |

**Rows and stacks nest to any depth**, so any grid-like arrangement can be written directly: a stack of KPIs beside a row whose second cell is itself a stack of charts is `row([stack([k1, k2, k3]), row([chart, stack([c2, c3])], widths=[3, 2])], widths=[1, 3])`. Width flows down (a row splits its width between children by weight); height is measured up and then shared out (see [Heights](#heights)).

```sql
{% set spec = sigma_report.report(
    name='Sales report',
    page_size='a4',                       -- letter (default) | legal | tabloid | a4 | a3 | screen | {width: .., height: ..}
    margin=40,
    header=[sigma_report.text('**Acme Corp** | Sales', key='header', h=32)],   -- repeats on every page
    footer=[sigma_report.text('Generated by dbt', key='footer', h=24)],
    pages=[
      sigma_report.page('Summary', [
        sigma_report.report_header('Sales summary', 'Hand-assembled from primitives'),
        sigma_report.row([sigma_report.stack([revenue, orders]), by_month], widths=[1, 2]),
        sigma_report.section('Mix', [pivot]),
      ]),
      sigma_report.page('Detail', [detail_table], paginate=true),
    ],
) %}
```

- `grid(elements, per_row=2)` turns a list of elements into rows; `section`, `report_header` and `kpi_row` are small composites over the same blocks.
- Content that does not fit a page **fails the build** with the overflow in pixels, so a clipped page can never ship silently. Add `paginate=true` to a `page()` and it continues on a new physical page instead (`Detail`, `Detail (2)`, ...), repeating the header and footer. `page_break()` forces a new page at a point of your choosing (and needs `paginate=true`).
- Header and footer heights are subtracted from the content area, so panel content can never overlap page content.
- `at(...)` takes no room in the flow; it lands where you say, on the page it is listed on. An element that is *meant* to overlap (a watermark, a logo over a banner) is declared with `overlap=true`, otherwise the geometry check treats overlap as a mistake. A pinned element must lie inside the content area.
- Pixel positions are validated in CI: nothing outside its page, nothing overlapping (`scripts/check_layout.py`, which you can run on your own manifest too).

### Heights

Every block has a **natural height**: an element's `h=` (default from [project settings](#project-settings)); text sized from its characters and the width it lands in (a long title wraps and the page flows down; the same paragraph in a narrow cell gets taller); a row is its tallest child; a stack is its children plus gaps. Pass `h=` in pixels to pin any of them.

A block can also be **flexible**: it takes a share of whatever height is left over, in proportion to its weight, like CSS `flex-grow`.

```sql
sigma_report.stack([sigma_report.flex(trend, weight=2), sigma_report.flex(mix), note])
-- trend and mix share the leftover page height 2:1 on top of their own heights; note keeps its height
```

- `flex(block, weight=1, min=none)` works on an element, a row or a stack; `h='fill'` on an element is shorthand for `flex` with a 120px minimum. A flexible block on a page takes the rest of that page; in a stack it shares what the stack has left; a row with a flexible child is flexible.
- A row stretches every child to its own height, so a chart beside a stack is as tall as the stack.

### Tables

A Sigma table element has no pagination or row-limit setting: it is a window onto the first rows of its sort order, as many as its height shows, with the total row count in a summary bar. So the useful question is "how many rows", and the package answers it four ways:

- `table(..., rows=10)`: show 10 rows. The height follows (about 130px of title, header and summary bar plus 20px a row; both numbers are [settings](#project-settings)). `sort=` chooses which rows. `h='fill'` gives a table the rest of its page.
- `table(..., window=[21, 40])`: show rows 21 to 40 of the sort order, using a hidden `RowNumber()` column and a range filter on it.
- `table_pages(source, columns, rows_per_page=40, total_rows=200, name=..., sort=...)`: a table **over several pages** - one window per page with a page break between (so the page needs `paginate=true`). Each page's title reads "(page i of n)", and each summary bar still shows the true total. The compiler cannot see your data, so you say how many rows to cover (`total_rows=` or `pages=`); rows beyond that are not shown, so leave headroom. About 40 rows fill a Letter page.

- `table(..., paginate=true)`: Sigma's own **"Expand table rows"**: the table grows to every row when the report renders (up to 10,000), with no row count needed from you. The trade-off is that nothing is known at compile time about how far it will grow, so it must be the last block on its page and cannot sit in a row, stack or panel ([details below](#paginated-tables-and-page-numbers)). Checked in Sigma: a 70-row table continued onto a second page with the footer page number ("1 of 2", "2 of 2").

**Which one?** Use `paginate=true` when the table is the last thing on the page and its length is unknown (a listing, a statement). Use `table_pages` when you need content after the table, fixed page boundaries or a known page count. Use `rows=` / `h='fill'` for a short "top N" table.

`simple_dashboard(..., table_columns=[...], table_sort=[...], table_page_count=3, table_rows=30)` windows its detail table over pages; `table_paginate=true` expands it instead.

## Components

Everything is a macro under `sigma_report.`; every public macro is documented with its arguments in [`macros/schema.yml`](macros/schema.yml).

| Layer | Macros |
| --- | --- |
| Report | `report`, `page`, `materialize`, `filter_by_value`, `filter_last_days` |
| Layout | `row`, `stack`, `flex`, `at`, `page_break`, `grid`, `section`, `report_header` |
| Elements | `kpi`, `table`, `table_pages`, `pivot_table`, `text`, `title`, `heading`, `divider`, `image`, `embed`, `button`, `progress`, `page_number` |
| Charts | `bar_chart`, `line_chart`, `area_chart`, `scatter_chart`, `pie_chart`, `donut_chart`, `combo_chart`, `waterfall_chart`, `box_chart`, `funnel_chart`, `gauge_chart`, `treemap_chart`, `sankey_chart`, `point_map`, `region_map`, `geography_map` |
| Composites | `simple_dashboard`, `simple_leaderboard`, `kpi_row`, `kpi_from_spec`, `chart_from_spec`, `facts_line`, `bullet_list` |
| Report presets | `simple_executive_summary`, `simple_period_review`, `simple_statement` |
| Sources | `from_ref`, `from_source`, `from_data_model`, `warehouse_table`, `data_model`, `element` |
| Fields | `field`, `format_currency`, `format_integer`, `format_percent`, `format_compact`, `format_number`, `format_date` |
| Actions | `button`, `action`, `with_actions`, and effects: `set_control_value`, `clear_control`, `clear_controls`, `open_url`, `navigate`, `refresh_element`, `download`, `effect` |
| Schedules | `schedule` (passed to `report(schedules=[...])`) |
| Escape hatches | `extend`, `raw`, `channel_element` (build your own typed kind) |
| Theme and settings | `theme`, `color`, `setting`; the `sigma_report_theme` and `sigma_report_settings` vars |

### Dashboards in one call

`simple_dashboard` is the general-purpose shape: title, KPI band, charts two to a row, optional detail table. Every part is optional; `show_title=false` drops the title block. `simple_leaderboard` is a second one built from the same pieces: a KPI plus one ranked horizontal bar chart per dimension.

```sql
{% set spec = sigma_report.simple_leaderboard(
    name='Regional standings',
    source=sigma_report.from_ref('stg_orders'),
    dimensions=['region', 'segment'],
    measure='Sum([Amount])', measure_name='Revenue',
    measure_format=sigma_report.format_currency(),
) %}
```

Both take `report_args={...}`: any argument of `report()` (`page_size`, `orientation`, `margin`, `header`, `footer`, `page_numbers`, `theme`, `description`, `folder_id`, `report_id`).

KPI mappings take `value`, `name`, `format`, `compare_to` (compare against a target measure), `timeline` + `period` (compare the latest period with the one before), `trend` (sparkline), `direction` (`higher` / `lower` / `none`), `description`. Chart mappings take a `type` (`bar`, `line`, `area`, `scatter`, `pie`, `donut`, `combo`, `waterfall`, `box`, `funnel`, `gauge`, `treemap` or `sankey`) plus that chart macro's arguments (`x`, `y`, `color`, `sort_by`, `stacking`, `labels`, ...).

### The chart family

The typed chart macros take fields (a bare column name or a formula) and build the columns and channels for you, so none of them needs hand-written Sigma JSON:

```sql
sigma_report.combo_chart(trips, 'DateTrunc("month", [Start Date])', ['Count()', 'Avg([Duration]) / 60'],
                         series_types=['bar', 'line'], secondary=[1], name='Trips and average length')   -- second measure on a 2nd axis
sigma_report.funnel_chart(orders, stage='segment', value='Sum([Amount])')
sigma_report.gauge_chart(orders, 'Sum([Amount])', minimum='0', maximum='Sum([Target Amount])')           -- number literals are fine
sigma_report.treemap_chart(orders, 'region', 'Sum([Amount])', color='Sum([Target Amount])')              -- color_by='scale' | 'category'
sigma_report.sankey_chart(orders, ['segment', 'region'], 'Sum([Amount])')
sigma_report.region_map(orders, 'state', 'us-state', color='Sum([Amount])')
sigma_report.waterfall_chart(...)   sigma_report.box_chart(...)   sigma_report.point_map(...)   sigma_report.geography_map(...)
```

A new kind that has no macro is a few lines with `channel_element` (see [`docs/design.md`](docs/design.md)); anything finer-grained is `extend`.

### Report presets

Whole documents from data alone, each a thin layer over the same public macros as `simple_dashboard` (and each taking `report_args`):

| Preset | What it makes |
| --- | --- |
| `simple_executive_summary` | headline KPIs, one full-width chart, a few highlight bullets |
| `simple_period_review` | KPIs for the latest period against the one before (with sparklines), a trend, a breakdown per dimension, notes and actions |
| `simple_statement` | a statement or invoice: facts, a line-item table, a totals band, a footer note |

```sql
{% set spec = sigma_report.simple_period_review(
    name='Monthly review',
    source=sigma_report.from_ref('stg_orders'),
    date_field='order_date', period='month',
    kpis=[{'value': 'Sum([Amount])', 'name': 'Revenue'}, {'value': 'Count()', 'name': 'Orders'}],
    breakdowns=['region', 'segment'],
    notes=['Action: review SMB pricing'],
) %}
```

Reports have no per-viewer filters, so `simple_statement` describes whatever its `source` holds: point it at a model already limited to one customer, and loop in dbt to make one per customer.

### Paginated tables and page numbers

A table's row count is not known when the report is compiled, so by default it gets a fixed height (see [Tables](#tables) for the options, including row windows). For a listing or a statement that can run to any length, ask for Sigma's "Expand table rows":

```sql
sigma_report.table(source, ['order_id', 'order_date', 'amount'], name='Orders', h=300, paginate=true)
```

The table (or `pivot_table`) then grows to hold all its rows when the report renders and continues onto further pages (up to 10,000 rows, vertically only); `h` is the height it starts with. It is emitted as `flow="paginated"` on the element's layout entry. Since the package cannot know how far it will grow, the rules are strict, and compile-time errors say so: a paginated element must be the **last block on its page**, cannot sit in a `row`, `stack` or panel, and anything you want "after" it belongs in the footer. `simple_dashboard(table_paginate=true)` does this for its detail table, and `simple_statement` is built around it.

`report(page_numbers=true)` puts a "1 of 3" page number at the right of the footer on every page; with a footer of your own, its elements share the footer row with the number. For full control add `sigma_report.page_number()` (or `page_number('Page {page}')`) to your own footer row.

### Building your own

`simple_dashboard` uses only public macros; when you outgrow it, write the same thing by hand. [`integration_tests/models/sigma_sales_report.sql`](integration_tests/models/sigma_sales_report.sql) is a runnable two-page example with header and footer panels, a pivot, a calculated column, a chart built on a table that is already on the page, and both escape hatches.

### Scoping a report to one value

Reports have no controls, so a report cannot be filtered by whoever opens it. `filter_by_value` fixes the scope in the report instead, which lets one general model feed a report about a single customer, region or API key rather than hard-coding that scope in the model:

```sql
{% set spec = sigma_report.simple_dashboard(name='West region', source=sigma_report.from_ref('stg_orders'), ...) %}
{% set spec = sigma_report.filter_by_value(spec, 'region', ['West']) %}   -- mode='exclude' drops the values instead
{{ sigma_report.materialize(spec) }}
```

It adds a hidden column and a `list` filter on it to every element built on a warehouse table. Elements built on another element inherit the filter, text and dividers have no data, and the build fails if no element could be filtered, so a report is never left silently unscoped. Take the values from a project var (`[var('my_key')]`) to keep them out of the SQL.
### Rolling date windows

A report has no date control, so "the last 7 days" has to be part of the element. `filter_last_days(element, column, days)` limits one chart, KPI or table to the last `days` days of a date column, measured from the day the report renders (Sigma evaluates `Today()` at render time, so a scheduled report never goes stale). It adds a hidden `If([Order Date] >= DateAdd("day", -7, Today()), 1, 0)` column and a `number-range` filter keeping 1. Elements you do not limit keep their full history, so one report can show an all-time trend next to a last-week breakdown.

`simple_dashboard` exposes it as a key on each KPI, chart or pivot, plus the source's date field:

```sql
{% set spec = sigma_report.simple_dashboard(
    name='Spend',
    source=sigma_report.from_ref('daily_spend'),
    date_column='activity_date',
    kpis=[{'value': 'Sum([Usage Usd])', 'name': 'Spend past week', 'last_days': 7},
          {'value': 'Sum([Usage Usd])', 'name': 'Spend past day', 'last_days': 1}],
    charts=[{'type': 'line', 'x': 'activity_date', 'y': 'Sum([Usage Usd])', 'name': 'Daily spend'},
            {'type': 'bar', 'x': 'model_id', 'y': 'Sum([Usage Usd])', 'name': 'Spend by model', 'last_days': 7}],
    table_pivot={'rows': ['model_id', 'provider_name'], 'values': ['Sum([Usage Usd])', 'Sum([Requests])'], 'last_days': 7, 'h': 'fill'},
) %}
```

`table_pivot` replaces the row-by-row detail table with a pivot, which totals by group. The window is the `days` calendar days up to now, so a source that lags a day shows through yesterday, and `last_days=1` is the most recent day only when the source is current. A `last_days` without `date_column`, a window on an element that is not built on a warehouse table, or the same column limited twice fails the build instead of leaving an element unscoped. It composes with `filter_by_value`.

### Actions

Sigma's request schema for reports takes an `actions` array on buttons, tables, controls, charts and KPIs: the same sequences workbooks have. An action is a trigger plus effects that run in order.

```sql
{% set refresh = sigma_report.button('Refresh', key='refresh', actions=[
    sigma_report.action('on-click', [
        sigma_report.refresh_element(orders),
        sigma_report.navigate(page=notes),
    ]),
]) %}

{# or onto an element you already built #}
{% do sigma_report.with_actions(orders, [
    sigma_report.action('on-select', [sigma_report.set_control_value('region-control', 'West')]),
]) %}
```

Effects with no builder (`call-api`, `insert-rows`, `update-rows`, `send-embed-event`, `if-else`, ...) go through `effect('call-api', {...})` with Sigma's own fields. Action ids are derived from the element id, trigger and position (or `key=`), so they are stable across compiles. [`integration_tests/models/sigma_actions_report.sql`](integration_tests/models/sigma_actions_report.sql) is a runnable example.

The schema accepts all of this (it is checked in CI). Whether a rendered, paginated report *runs* its actions is Sigma's behavior, not something the package can prove: check a real report after `--push --dry-run`.

### Scheduled exports

There is no scheduled trigger for actions, and a report's contents have no schedule field. Schedules are a separate Sigma resource (`/v2/reports/{id}/schedules`), and reports export only as a whole PDF. So `schedule(...)` is compiled as data, carried on the spec beside `contents`, and applied by the export script after the report exists:

```sql
{% set spec = sigma_report.report(
    name='Sales report',
    pages=[...],
    schedules=[
      sigma_report.schedule('Weekly sales', cron='0 8 * * 1', timezone='America/New_York',
          to=['finance@example.com', {'teamId': '<your-team-id>'}],
          message='This week''s numbers.', only_if_not_empty=orders),
    ],
) %}
```

`export_reports.py --push` creates each schedule, or updates the one with the same name (the title Sigma shows; renaming creates a new schedule). It never deletes schedules, and `--dry-run` doesn't touch them since Sigma has no dry run for them. `--out` writes them to `<model>.schedules.json`. Recipients are email addresses (strings) or Sigma target mappings used as written, such as `{'teamId': ...}` or `{'slackConversationId': ...}`. Sigma documents `slackConversationId` as a Slack channel ID or member ID: the schedule dialog also accepts a `#channel-name`, but the API docs don't say the API does, so use the ID. Anything the macro does not model (control values, attachment merging, a column condition) goes in `options={...}`, merged onto Sigma's `configV2`.

### Escape hatches

Sigma's schema has hundreds of options beyond what the macros model, and reaching for one should not mean writing unchecked JSON.

- `extend(element, {...})` deep-merges raw Sigma properties onto an element you built with a normal macro.
- `raw(kind, key, {...}, h=...)` makes an element of any kind in Sigma's report schema that the package has no macro for (`report-repeated-container`, `input-table`, `plugin`, ...). It still takes part in layout and id checks.

Both are **validated at `dbt parse`** against a registry generated from Sigma's own schema ([`macros/elements/kinds.sql`](macros/elements/kinds.sql), rebuilt by `scripts/generate_kinds.py`; CI fails if it is stale): an unknown kind, a missing required property (`raw`) or a misspelt property name fails with the list of what Sigma allows. Property *values* are checked by `scripts/validate_against_sigma_schema.py` against Sigma's published schema, and `--push --dry-run` against your organization.

## Sources

| Macro | Reads from |
| --- | --- |
| `from_ref('stg_orders')` | a dbt model, seed or snapshot (calls `ref()`, so the report is downstream of it in the DAG) |
| `from_source('raw', 'orders')` | a dbt source (calls `source()`) |
| `from_data_model('my_model', 'orders')` | a table in a Sigma data model built with `sigma_data_models` |
| `warehouse_table([db, schema, table])` | any warehouse table by physical path |
| `element(table_element)` | another element already on the page |

A Sigma report refers to warehouse tables by physical path, but at parse time dbt cannot resolve other nodes' relations, so the path is built from `target.database` / `target.schema` and the model name. If your models build into other schemas, pass `schema=` per call, or override the one macro that decides this once:

```yaml
# dbt_project.yml
dispatch:
  - macro_namespace: sigma_report
    search_order: ['my_project', 'sigma_report']
```

```sql
-- macros/sigma_report_overrides.sql  (the name must be exactly `default__relation_path`)
{% macro default__relation_path(kind, name, database=none, schema=none, identifier=none) %}
  {% do return([database or target.database, schema or 'analytics', identifier or name]) %}
{% endmacro %}
```

`kind` is `'ref'` or `'source'`. A runnable example is in [`integration_tests/macros/overrides.sql`](integration_tests/macros/overrides.sql).

**Sigma data models.** `from_data_model` takes the dbt model that carries the data model, and the `key` the table was given in `sigma_data_models.table()`. Sigma assigns the data model's id when you create it, so record it once in the `sigma_data_model_ids` var; until then the report carries an `unresolved:<model>` placeholder that the export script refuses to push. The table's element id is derived from the same path `sigma_data_models` uses, so the two packages agree without coordinating.

## Theming

Defaults are Sigma's: the Neutral gray scale with the Blue accent, Inter type, rounded white cards with a hairline border, and Sigma's `superbloom` categorical palette for multi-series charts. Re-skin every report from `dbt_project.yml`:

```yaml
vars:
  sigma_report_theme:
    hue: Emerald                       # swap the accent scale
    base: Slate                        # swap the gray scale
    colors: {primary: "#0b6e4f"}       # or pin any single token to your brand color
    categorical: ["#0b6e4f", "#f2a900", "#3d5a80"]   # series palette (or a Sigma palette name)
    fonts: {text: Inter, data: Inter}
    border_radius: round               # square | round | pill
    cards: true                        # false = no card border around data elements
    page_background: surface           # a token or hex; white by default (reports are printed)
```

| Key | Choices |
| --- | --- |
| `base` | `Neutral` (default), `Stone`, `Zinc`, `Slate` |
| `hue` | `Blue` (default), `Neutral`, `Indigo`, `Violet`, `Emerald`, `Teal`, `Green`, `Rose`, `Orange`, `Amber` |
| `colors` | any of the tokens below |
| `categorical`, `sequential`, `diverging` | a Sigma palette name; `categorical` also takes a list of hex colors |

Tokens: `canvas`, `ink`, `body`, `muted`, `hairline`, `fill`, `series_grey` (from the base); `primary`, `tint`, `edge`, `series`, `series_mid` (from the hue); `surface`, `success`, `danger`, `warning` and their `_fill` variants. The scales are Sigma's own design-system scales.

Precedence, lowest to highest: package defaults, the `sigma_report_theme` var, then `theme={...}` on a single `report()`. Each level is deep-merged, so overriding one thing keeps the rest. A per-report theme recolors elements that were built before `report()` ran (colors are late-bound), and an unknown hue, base or token fails at `dbt parse` rather than silently falling back.

## Project settings

Every size and calibration constant the package would otherwise hard-code is a setting, so a team sets its house style once and every report follows. Override any key in `dbt_project.yml` (keys you leave out keep the defaults; an unknown key fails at `dbt parse`, so a typo cannot silently do nothing):

```yaml
vars:
  sigma_report_settings:
    page_size: a4
    margin: 40
    gap: 12
    chart_h: 300          # default chart height
    kpi_h: 80
    title_size: 32        # px
    table_row_h: 24       # recalibrate if your theme or font draws taller rows
    char_width: 0.6       # em per character, for estimating text height
```

| Group | Keys (defaults) |
| --- | --- |
| Page | `page_size` (letter), `margin` (48), `gap` (16), `header_height` (48), `footer_height` (32) |
| Element heights | `kpi_h` (96), `chart_h` (260), `pivot_h` (300), `map_h` (320), `gauge_h` (180), `divider_h` (12), `image_h` (64) |
| Tables | `table_rows` (10), `table_row_h` (20), `table_chrome` (130), `table_chrome_untitled` (92) |
| Type | `title_size` (28), `heading_size` (20), `body_size` (14), `h1_size` / `h2_size` / `h3_size` (36 / 28 / 22) |
| Text estimate | `char_width` (0.55 em per character), `line_height` (1.35), `text_padding` (6) |

A per-call argument (`h=`, `margin=`, `page_size=`, ...) always wins over the project setting, which wins over the package default. The table and text numbers are *measurements* of Sigma's rendering, not documented by Sigma, so they are the settings to recalibrate if your theme or font differs: push a report, export it to PDF (`POST /v2/reports/{id}/export`) and compare.

## Publishing to Sigma

```bash
python dbt_packages/sigma_report/scripts/export_reports.py --out sigma_reports/   # one <model>.json per report
python dbt_packages/sigma_report/scripts/export_reports.py --check                # offline sanity checks
python dbt_packages/sigma_report/scripts/export_reports.py --push --dry-run       # validate against your Sigma org
python dbt_packages/sigma_report/scripts/export_reports.py --push                 # create / update
```

The script needs only the Python standard library. Pushing needs [API credentials](https://help.sigmacomputing.com/docs/generate-api-client-credentials) in `SIGMA_BASE_URL` (your cloud's API URL), `SIGMA_CLIENT_ID` and `SIGMA_CLIENT_SECRET`.

- First push creates the report (`POST /v2/reports`). Note the `reportId` Sigma returns and pass it as `report_id=` (or `report_args={'report_id': ...}`) from then on: the next push replaces that report's contents in place (`PUT /v2/reports/{id}/contents`), so the report keeps its URL, schedules and sharing. Any `schedules=` on the report are applied after the push (see [Scheduled exports](#scheduled-exports)).
- `--dry-run` sends `dryRun: true`, which validates formulas, sources and layout without saving. Run it against your own organization after any change to what elements look like.
- The script refuses to push an `unresolved:` data model id or the all-zero placeholder ids used in this repository's tests.

Reports as code is a Beta Sigma feature; its API can change.

## dbt versions

The package declares `dbt >=1.6.0, <3.0.0`, and CI runs the whole integration suite on **both dbt 1.12 and dbt 2.0** (`dbt-core==2.0.0rc8`), producing byte-identical output (the same golden files pass on both). What dbt 2 conformance took:

- **Macro arguments are documented and checked.** `macros/schema.yml` lists every public macro's arguments with types from dbt's grammar (`string`, `bool`, `int`, `any`, `list[any]`, `dict[string,any]`). dbt 2 validates it against the macro signatures, and the integration project sets `flags: validate_macro_args: true` and runs `dbt parse --warn-error`, so a drift between the two fails CI on either version. (dbt >= 1.10 warns on a mismatch; dbt 2 is stricter.)
- **Portable Jinja only.** No `modules.re` or other Python-only escapes: strings are parsed with `split`, mutation uses `namespace`, and dict keys that collide with dict methods (`values`, `items`) are read with `['values']`.
- **Config goes in `meta`.** The spec is attached with `config(meta=...)`, not a custom config key.
- **Dispatch is by adapter prefix.** The one override point, `relation_path`, uses `adapter.dispatch`, which works the same on both engines.
- **No warehouse needed to compose.** Everything resolves at parse time, which is also what makes `dbt parse` on dbt 2 enough to produce reports.

## Testing

```bash
pip install -r ci/requirements-dbt1.txt        # or ci/requirements-dbt2.txt for dbt 2.x
bash scripts/run_integration_tests_duckdb.sh   # the whole suite; DuckDB, no credentials
python -m unittest discover -s scripts -p "test_*.py"
```

The integration suite is a real dbt project ([`integration_tests/`](integration_tests)) that compiles eight reports from a seed, then:

1. asserts the shape of every compiled report (field names, cross-references, theme flow, ids from vars);
2. checks pixel geometry from the layout XML (inside the page, nothing overlapping);
3. compares each report with a committed golden JSON file (output is deterministic: every id is a hash of a stable path);
4. validates each against Sigma's request schema, extracted from the public OpenAPI document;
5. runs 62 inputs that must fail `dbt parse`, each with the exact error it must raise;
6. re-runs the assertions with different Sigma ids to prove every id flows from the vars, and with different project settings to prove every size flows from those;
7. checks that the element-kind registry (`kinds.sql`) matches Sigma's schema.

CI (`.github/workflows/integration-tests.yml`) runs all of it on dbt 1.x and dbt 2.x.

## Coverage and limits

An initial version aimed at dashboards.

- **Validated against Sigma's schema, not rendered in CI.** The request schema check, the geometry check and the golden files run in CI. CI has no Sigma credentials, so how a report *looks* in Sigma, and whether Sigma accepts the formulas against your data, is checked by `--push --dry-run` against your own organization. To *see* a report, push it and export it: `POST /v2/reports/{id}/export` returns a PDF. Panel sizing is an assumption: header and footer heights are subtracted from the content area, which is safe (no overlap) but may leave more white space than Sigma needs.
- **Modeled elements**: KPI; bar, line, area, scatter, pie, donut, combo, waterfall, box, funnel, gauge, treemap and sankey charts; point, region and geography maps; table (with row windows and multi-page `table_pages`), pivot table; text, title, heading, divider, image, embed, button, progress. Not modeled (use [`raw`](#escape-hatches), which is validated against Sigma's schema): input tables, plugins, repeated containers, conditional formatting, reference marks, trellis. Verified by rendering a pushed report in Sigma: combo (with a secondary axis), treemap, funnel, waterfall, gauge, box, nested layouts, flexible heights, `at()` and a 90-row table over three pages. Sankey, the three maps, progress, button and embed are checked against Sigma's request schema in CI (and the compiled output of the others also passed Sigma's dry run), but not yet rendered.
- **No control macros.** Sigma's schema does have report controls (and actions can set them by id), but the package does not build them yet; use [`raw`](#escape-hatches).
- **Schedule bodies are checked offline only** (cron, recipients, unique names, the element a condition tests). Sigma's schema check in CI covers a report's contents, not its schedules.
- **Layout is a tree of rows and stacks, plus pinned elements.** There is no free-form drag layout; `at()` pins single elements, and anything else is a row or a stack. Overlap is only allowed for pinned elements declared `overlap=true`.
- **Text height is an estimate.** It is computed from character counts and font size (generously, so text is not clipped); unusual fonts, very long unbroken words or `{{formula}}` segments can differ. Pin `h=` or tune `char_width` / `line_height` when it matters. The table row height (20px) and chrome (130px) were measured from a Sigma PDF export, not documented by Sigma, so they could shift - they are [settings](#project-settings).
- **Multi-page tables need you to say how many pages.** `table_pages` slices the sort order into row windows (`RowNumber()` + a filter); the compiler cannot see your data, so rows beyond `total_rows` are not shown (each window's summary bar still shows the true total, so a shortfall is visible). A windowed table needs a deterministic `sort`.
- **Source paths** use `target.database` / `target.schema` unless overridden (see [Sources](#sources)).
- **Formulas are strings.** The package cannot check them against your data; the dry run does.
- One `sigma_report` per model; other `meta` keys are lost to `materialize()`.

## More

- [`docs/design.md`](docs/design.md): why the package is shaped this way, a critique of how composable it really is, what that review changed, and the recipe for adding an element.
- [`CONTRIBUTING.md`](CONTRIBUTING.md) and [`AGENTS.md`](AGENTS.md): working on the package.

## License

MIT, see [`LICENSE`](LICENSE).
