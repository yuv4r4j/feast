# Feature Specification: Refresh the SMOKEY demo page with actual data, using Snowflake Cortex

**Feature Branch**: `013-cortex-data-refresh`
**Created**: 2026-11-25
**Status**: Draft
**Companion file**: `smokey-demo.html` (the one-file demo page this spec updates)
**Input**: "Create new spec for artifact to be updated by Cortex with actual data. Two files: one HTML page for the demo and a spec for updating real data."

## 1. Summary

The demo is one HTML file. All of its portfolio data sits in one JSON block inside that file. This spec says how a Snowflake Cortex agent replaces that block with actual data from Snowflake, and nothing else.

Three rules carry the design:

1. **The agent never types a number.** It runs SQL against agreed views and runs a fixed script that builds the JSON and splices it into the page. Every figure on the page comes from a query result, not from model output.
2. **Only the data block changes.** Page code, styles and text stay byte-for-byte the same. The script proves this with a hash of everything outside the block.
3. **The page says what is actual.** After a refresh, each screen is labelled "Actual data", "Partly actual" or "Illustrative scenario". A scripted example is never passed off as the bank's data.

## 2. What "Cortex" means here

Snowflake's coding agent, documented as Snowflake CoCo (Cortex Code). Two ways to run it fit this job:

| Option | Where it runs | Fits when |
|---|---|---|
| **CoCo CLI** (`cortex`) | On a workstation or build runner, using an existing Snowflake connection | The page file lives in a repository or shared folder |
| **Cortex Agents Coding Agent** | In a Snowflake-hosted sandbox, called by REST, SQL or MCP | The page file lives in a Snowflake workspace or stage |

Facts taken from Snowflake's documentation on 25 Nov 2026 (read as extracts by an automated reader; confirm against the live pages before build):

- The CLI can run SQL (`SnowflakeSqlExecute`), read, write and edit local files, and run shell commands. It runs in batch mode with `cortex -p` or `cortex exec`; with no one to ask, any tool call that is not pre-approved is denied. `--allowed-tools` and `--disallowed-tools` set the list.
- It acts with the active Snowflake role and honours role-based access control.
- Standing instructions go in an `AGENTS.md` file; reusable procedures go in skills built around a `SKILL.md` file.
- The hosted Coding Agent's SQL tool is read-only (SELECT and SHOW) and can write to stages; its default permission is to ask before each action.
- No file-size limit is stated for reading or writing files. The page is about 430 KB, which is the reason rule 1 has the script do the splice, not the agent.

Not established: the read-only SQL flag for the CLI (seen only in a summary), scheduling ("Automations" page not read), and the models available in the bank's account.

## 3. User Scenarios & Testing *(mandatory)*

### User Story 1 - Monthly refresh (Priority: P1)
A model risk analyst runs one command after month-end monitoring closes. Cortex reads the agreed views, builds the data block, validates it, splices it into the page and writes a refresh report. The analyst opens the page and sees the as-of date and "Actual data" in the header.

**Independent Test**: Run the refresh against the test schema; open the page; the header shows the new as-of date and the seven data-bound screens show the test schema's models.

**Acceptance Scenarios**:
1. **Given** valid views, **When** the refresh runs, **Then** a new page file is written whose data block validates and whose content outside the block has the same hash as before.
2. **Given** a refreshed page, **When** any screen opens, **Then** it carries one of the three data-status labels.
3. **Given** a refresh, **Then** a report lists row counts per section, each validation rule with pass or fail, the query IDs, the role used and the hashes.

### User Story 2 - A bad refresh changes nothing (Priority: P1)
**Acceptance Scenarios**:
1. **Given** a view returning a model with 11 months of PSI, **When** validated, **Then** the run stops with rule V-03 and the page file is untouched.
2. **Given** a section with zero rows, **Then** the run stops (V-01); an empty portfolio is never published.
3. **Given** any validation failure, **Then** the previous page stays in place and the report says which rule failed, on which key.

### User Story 3 - Nothing personal or customer-level leaves Snowflake (Priority: P1)
**Acceptance Scenarios**:
1. **Given** the refresh role, **When** it queries any object outside the `SMOKEY_DEMO` schema, **Then** Snowflake refuses.
2. **Given** a cell built from fewer than the minimum number of accounts, **Then** the view returns null and the page shows "not available", never a small count.
3. **Given** owner names in the source, **Then** the view returns a role label (for example `Acq-owner-03`) unless the data owner has approved names.

### User Story 4 - Scripted screens are labelled (Priority: P2)
**Acceptance Scenarios**:
1. **Given** actual data, **When** a scripted screen opens (for example L0 Decisions), **Then** it shows "Illustrative scenario: its figures and model keys are not from your data."
2. **Given** model keys that differ from the demo's, **Then** all 28 screens load with no script error (verified: see section 9).

### User Story 5 - Roll back (Priority: P2)
**Acceptance Scenario**: **Given** a refresh someone disputes, **When** the previous file is restored from `history/`, **Then** the page shows the earlier as-of date; no rebuild is needed.

### Edge Cases
- A model is live but has no monitoring rows this month: health is `Unknown`, the series ends in null, and the page shows Unknown (never Healthy).
- A new model key appears mid-year: it gets 12-month arrays padded with null at the start.
- The portfolio has more than about 60 live models: the drift grid still works but the file grows; see NFR-2.
- Two refreshes on the same as-of date: the second overwrites the page and both are kept in `history/` with their times.
- The agent proposes to "fix" the page code: forbidden (rule 2); it must stop and report.

## 4. The page contract

`smokey-demo.html` contains exactly one element:

```html
<script id="smokey-data" type="application/json">{ ...one JSON object... }</script>
```

followed by a one-line loader that parses it into `window.SMK`. The refresh replaces the text between those tags. Inside the JSON, every `</` is written `<\/` so the block cannot close early.

Top-level keys (schema `smokey-data/1.0`):

| Key | Shape | Read by screens |
|---|---|---|
| `meta` | object | header line, data-status labels |
| `models` | array, one per live model | L2 Drift, Model sheet, Features, Owner, Policy preview |
| `feat` | array, one per monitored feature | Features |
| `views` | array, one per feature view | Features |
| `port`, `portED` | portfolio delinquency series | L2 Drift |
| `reg` | array, registry rows | Registry |
| `tier` | `{models, tools, sum}` | Tiering, Policy, Model sheet |
| `inv` | object keyed by model key, 70 columns | Model sheet (inventory record) |
| `biz` | `{cycle_releases, campaign_segments}` | Cycle time, Campaign |
| `trail` | object keyed by model key, optional | Audit trail |

### 4.1 `meta`
| Field | Type | Rule |
|---|---|---|
| `schema` | string | must be `smokey-data/1.0` |
| `source` | `synthetic` or `actual` | the refresh sets `actual` |
| `as_of` | date `YYYY-MM-DD` | the monitoring month-end |
| `environment` | string | for example `PROD` or `UAT`; shown in the header |
| `refreshed_by`, `refreshed_at` | string, timestamp | agent name and run time |
| `row_counts` | object | models, inventory, tier_models |
| `focus` | object, optional | `breach_model`, `watch_model`, `tier_example`, `registry_mismatch`: model keys the scripted screens use as their example. If absent, the page picks the highest-tier model in breach. |

### 4.2 `models[]` (one per live model)
| Field | Type | Meaning |
|---|---|---|
| `id` | string | Model Key, 5 characters (for example `XCE01`) |
| `type`, `bu` | string | model type; business area |
| `tier` | 1 to 4 | tier in force |
| `h` | `Healthy`, `Watch`, `Breach`, `Unknown` | production health as the control plane holds it |
| `dep` | string | main feature view |
| `own` | string | owner label |
| `v` | string | live version |
| `last` | `YYYY-MM` | last month evaluated |
| `psi` | 12 numbers | population stability index, oldest month first |
| `ks` | 12 numbers | relative drop in KS from validation, per cent |
| `ksv`, `ksc` | number | KS at validation; current KS |
| `csi` | array of `[feature, rank, feature_view, 12 numbers]` | characteristic stability per feature the model reads |
| `csiW` | 12 numbers | worst feature CSI per month |
| `edA`, `edE`, `edR` | 12 numbers or null | early delinquency actual, expected, ratio (30+ days past due within 3 months of scoring) |
| `edM` | 12 of `Full`, `Partial`, `Not yet` | cohort maturity |
| `exp`, `a30`, `a60`, `a90` | 12 numbers | expected 30+ rate; actual 30+, 60+, 90+ rates, per cent |
| `ddR` | 12 numbers | actual over expected 30+ |

Every array has exactly 12 entries, oldest first, ending at `last`. The nine delinquency series (`edA` to `ddR`) are present together or not at all: models that do not score accounts for delinquency (loss forecasting, capital, collections and recovery models in the synthetic set) omit them, and the page then shows those two metrics as not applicable. The month labels on the page are derived from `meta.as_of` (task T-06; today they are fixed at Dec to Nov).

### 4.3 `feat[]`, `views[]`
`feat`: `f` name, `view`, `band` (worst band across users), `min`, `max` (CSI range this month), `nb`, `nw` (models in breach, on watch), `live` (array of `[model key, rank]`), `fl` (array of `[model key, in-flight version]`).
`views`: `feature_view`, `source`, `owner`, `ttl_hours`, `hours_since_materialised`, `freshness`, `features`, `live_models_using`, `inflight_versions_using`.

### 4.4 `reg[]`
`model_id`, `registered_model`, `mlflow_version`, `control_plane_version`, `alias`, `flavor`, `run_id`, `artifact_digest` (first 16 hex characters only), `control_plane_state`, `authorised_for_production`, `cluster_serving`, `agrees_with_control_plane`, `note`.

### 4.5 `tier`
`models[]`: `id`, `code`, `type`, `bu`, `st` status, `h`, `t0` tier in force, `t` proposed tier, `d` `c` `i` (impact, uncertainty, implementation risk, each 1 to 3), `s` score, `exp` exposure in $bn, `band`, `use`, `rel`, `aa` adverse-actionable flag, `meth`, `cx`, `ib`, `fl` floor applied, `cls` (`Model` or `Tool`), `why`.
`tools[]`: `id`, `n` name, `k` kind, `bu`, `f` model it feeds, `exp`, `cr`, `cs`, `at`.
`sum`: `version`, `weights`, `thresholds`, counts by tier, lists of keys moving up and down, `health_by_tier`.

### 4.6 `inv`
One object per model key with the bank's inventory columns exactly as named in spec 001 (70 columns; names beginning `+` are SMOKEY additions). Missing values are empty strings, never the word "null".

### 4.7 `biz`
`cycle_releases`: array of `[release label, 9 numbers]`: months in development, validator queue, validation, approval, IT release queue, implementation, test accrual, outcome wait, ramp.
`campaign_segments`: array of 10 `[response %, one-year charge-off %, three-year return on assets %, share of list %]`, best score band first.

### 4.8 `trail` (optional)
One array per model key, oldest entry first. Each entry is `[sequence, date, month_index, event, role, kind, detail, rule, hash]`:
`sequence` integer from the event log; `date` as shown (for example `10 Nov 2026`); `month_index` 0 to 11 within the 12-month window; `event` the event name (for example `ApprovalDecided`, `CommandRefused`); `role` the acting role, never a person's name; `kind` one of `dec` (decision by a person), `ref` (refusal), `chg` (override or change), `auto` (automatic); `detail` one line, no customer data; `rule` the rule ID for a refusal or flag, else empty; `hash` the first 8 characters of the entry's chain hash.
When `trail` is absent the page builds an illustrative trail from the monitoring series and labels the screen "Illustrative scenario". When present, the Audit trail screen and its summary across models are "Actual data".

## 5. Where the data comes from

Cortex reads only views in one schema, `SMOKEY_DEMO`. The bank's data team owns the views; they are the place where source tables are joined, aggregated and masked. Cortex never queries source tables.

| View | One row per | Feeds | Likely source (to confirm) |
|---|---|---|---|
| `V_SMK_META` | run | `meta` | monitoring calendar |
| `V_SMK_MODELS` | live model | `models` scalars | inventory, control plane |
| `V_SMK_MONITOR_MONTHLY` | model × month × metric | `psi`, `ks`, `ed*`, `a30/60/90`, `exp`, `ddR` | monitoring result tables |
| `V_SMK_CSI_MONTHLY` | model × feature × month | `csi`, `csiW`, `feat` | monitoring result tables |
| `V_SMK_FEATURE_VIEWS` | feature view | `views` | Feast registry or Snowflake feature store |
| `V_SMK_PORTFOLIO_DQ` | month | `port`, `portED` | portfolio delinquency reporting |
| `V_SMK_REGISTRY` | model × registered version | `reg` | MLflow, Snowflake model registry |
| `V_SMK_TIER_MODELS`, `V_SMK_TOOLS`, `V_SMK_TIER_SUMMARY` | model, tool, run | `tier` | tiering inputs |
| `V_SMK_INVENTORY` | model | `inv` | model inventory |
| `V_SMK_CYCLE_RELEASES` | release | `biz.cycle_releases` | stage clocks (spec 012) |
| `V_SMK_CAMPAIGN_SEGMENTS` | score band | `biz.campaign_segments` | campaign forecast models |
| `V_SMK_AUDIT_EVENTS` | model × event, last 12 months | `trail` | control-plane event log; until that exists, approval, validation and change records |

Each view has an `AS_OF` column; every query filters on the one as-of date from `V_SMK_META`. Until a view exists, its section stays synthetic and `meta.sections_actual` (list) leaves it out; the page then labels the screens that read it "Illustrative scenario".

## 6. Requirements *(mandatory)*

### Functional
- **FR-001**: The refresh is one command with two inputs: the page path and the as-of date (default: the latest in `V_SMK_META`).
- **FR-002**: Cortex runs the fixed query set in `refresh/queries.sql`, one statement per view, and saves each result as a file under `refresh/out/<as_of>/`. It does not write or alter these queries during a run.
- **FR-003**: `refresh/build_block.py` turns the result files into the JSON object. `refresh/splice.py` validates it and replaces the block (appendix A). Cortex invokes the scripts; it does not edit the page with its own file tools.
- **FR-004**: Validation rules V-01 to V-12 (section 7) run before any write. Any failure stops the run with a non-zero exit and leaves the page untouched.
- **FR-005**: The script records SHA-256 of the page outside the data block before and after; they must match.
- **FR-006**: The page resolves the example models for scripted screens from `meta.focus`, or picks them from the data, so unknown model keys never break a screen. *(Built in this version of the page.)*
- **FR-007**: The page shows, on every screen, one of: "Actual data", "Partly actual", "Illustrative scenario", whenever `meta.source` is `actual`; the header shows environment, model count, as-of date and who refreshed. *(Built.)*
- **FR-008**: Each run writes `refresh/out/<as_of>/report.md`: role, warehouse, query IDs, row counts, validation results, hashes, and the previous as-of date.
- **FR-009**: The previous page is copied to `history/smokey-demo.<as_of>.<time>.html` before the new one is written.
- **FR-010**: Publishing the refreshed page (to the Claude artifact, the Kubernetes deployment in `deploy/`, or a Snowflake stage) is a separate, human step. Cortex does not publish.
- **FR-011**: The agent's standing instructions are the `AGENTS.md` text in appendix B; the procedure is packaged as a skill named `smokey-refresh`.
- **FR-012**: The refresh role (`SMOKEY_REFRESH_RO`) has USAGE on one warehouse and the `SMOKEY_DEMO` schema and SELECT on its views, and nothing else.

### Privacy and data handling
- **FR-013**: No customer-level, account-level or applicant-level row leaves Snowflake. Every view is aggregated to model, feature, month or score band.
- **FR-014**: Cells built from fewer than a minimum count (proposed 20 accounts; the data owner sets it) are null.
- **FR-015**: Names of people are replaced by role labels unless approved. Digests are truncated. No connection strings, tokens or query text with literals appear in the page or the report.
- **FR-016**: The page with actual data is confidential bank data. It must not be published to a link outside the bank's control without the data owner's approval; the synthetic version stays the shareable one.

### Non-functional
- **NFR-1**: A refresh completes in under 10 minutes for 60 models.
- **NFR-2**: The page stays under 2 MB; above about 150 models the data block needs trimming (drop `inv` columns not shown).
- **NFR-3**: The run is repeatable: same views and as-of date give a byte-identical data block (keys sorted, fixed number formatting).

## 7. Validation rules

| Rule | Check |
|---|---|
| V-01 | Every top-level key is present; `models`, `tier.models` and `inv` are non-empty |
| V-02 | `meta.schema` is `smokey-data/1.0`; `as_of` is a valid date not in the future |
| V-03 | Every monthly array has exactly 12 entries of number or null; `psi`, `ks`, `csiW` are required; the nine delinquency series come all together or not at all |
| V-04 | Model keys are unique and match `^[A-Z]{3}[0-9]{2}$` |
| V-05 | `h` is one of the four health values; `tier` is 1 to 4 |
| V-06 | Every `models[].id` exists in `tier.models` and in `inv` |
| V-07 | Every `feat[].live` and `reg[].model_id` key exists in `models` or `tier.models` |
| V-08 | For every `feat[].live` pair, that model's `csi` holds that feature |
| V-09 | Ranges: PSI and CSI 0 to 5; rates 0 to 100; ratios 0 to 10 |
| V-10 | A model with `h` other than `Unknown` has a number in the last month of `psi` |
| V-11 | The JSON contains no `</` sequence after escaping, no `NaN`, no `undefined`, and no string longer than 2,000 characters |
| V-13 | If `trail` is present: every key exists in `models`; `kind` is one of the four values; sequences rise within a model; no entry holds an email address or a name from the staff directory (pattern check) |
| V-12 | Change guard: model count differs from the previous page by no more than 25%, unless `--accept-change` is given by a person |

## 8. Which screens read the data block

From a scan of the page code and a test with renamed model keys (section 9). Confirm in phase 1.

| Status after refresh | Screens |
|---|---|
| **Actual data** | L2 Drift, Model sheet, Features, Tiering, Policy (preview of what moves), Cycle time, Campaign; Audit trail when `trail` is supplied |
| **Partly actual** (charts and tables from data; story text scripted) | Owner, Registry, Validator, Operations, Head of model risk |
| **Illustrative scenario** | Overview, Lifecycle, L0 Decisions, L2 Why, L1 Posture, Research, Model build, Platform, Head of validation, Approval, Compliance, Audit, Recalibration, Live test, Proof |

Binding the illustrative screens to actual data is later work and needs new views (decision queue, findings, validation cases, deployments, tests). It is out of scope here.

## 9. What was tested on the page (25 Nov 2026)

- The page was rebuilt with the single JSON block and passes its existing tests at 1300 and 390 px.
- A copy was made with every model key renamed, five cycle releases instead of six, and `meta.source` set to `actual`. All 28 screens loaded with no script error; the header and the three labels showed correctly.
- The splice script in appendix A was run on the page: one valid block was accepted (outside-block hash unchanged, previous page copied to `history/`, second run byte-identical) and six bad blocks were refused with the page untouched (11-month series V-03, model count drop V-12, key missing from inventory V-06, empty section V-01, invalid health value V-05, future date V-02). The spliced page then loaded all 28 screens with no script error.
- **Known defect in that test**: the Owner screen printed "undefined" in 12 places for model keys absent from the page's built-in map of design boards. Task T-07 removes that map's dependence on fixed keys.
- Not tested: a real refresh. No Snowflake account was used, the views do not exist yet, and Cortex was not run.

## 10. Tasks

**Phase 1: Agree**
- [ ] T-01 Data owner and model risk approve the view list, the minimum cell count and whether owner names may appear
- [ ] T-02 Security approves the role, the place the page file lives and where the refreshed page may be shown
- [ ] T-03 Choose CoCo CLI or the hosted Coding Agent; confirm batch mode, read-only SQL and scheduling against the live documentation

**Phase 2: Build**
- [ ] T-04 Create `SMOKEY_DEMO` views against the test schema; `refresh/queries.sql`
- [ ] T-05 `refresh/build_block.py`, `refresh/splice.py` (appendix A), unit tests for V-01 to V-12
- [ ] T-06 Page: derive month labels from `meta.as_of`; read `meta.sections_actual`
- [ ] T-07 Page: board-coverage map tolerates unknown keys (Owner screen)
- [ ] T-08 `AGENTS.md` and the `smokey-refresh` skill (appendix B)

**Phase 3: Prove**
- [ ] T-09 Refresh against the test schema; run the page tests on the result
- [ ] T-10 Negative tests: 11-month series, empty section, key missing from inventory, attempt to query outside the schema
- [ ] T-11 First production refresh with a person watching; compare 5 models by hand against the monitoring report

## 11. Success Criteria *(mandatory)*
- **SC-001**: A refresh changes only the data block: the outside-block hash is identical in 100% of runs.
- **SC-002**: Every number on an "Actual data" screen can be traced to a query ID in the run report.
- **SC-003**: No failed validation ever produces a changed page.
- **SC-004**: Five models checked by hand match the monitoring report to the displayed precision.
- **SC-005**: The refresh role can read nothing outside `SMOKEY_DEMO` (tested by an attempted query).

## 12. Open questions
1. Which Snowflake tables hold monitoring results, and do they keep 12 months of CSI per feature?
2. Is expected delinquency (`exp`, `edE`) stored, or must it be derived from the model's calibration?
3. Does the control plane exist yet as a source for `h`, approvals and registry agreement, or do those come from the inventory for now?
4. Where will the actual-data page be shown: a bank-hosted deployment only, or also a private Claude artifact?
5. May the agent run unattended on a schedule, or must a person start each refresh?

## Appendix A: the splice script (reference implementation)

```python
#!/usr/bin/env python3
"""splice.py PAGE NEW_BLOCK.json  -- replace the smokey-data block; change nothing else."""
import sys, re, json, hashlib, shutil, datetime, pathlib

OPEN, CLOSE = '<script id="smokey-data" type="application/json">', '</script>'
KEY = re.compile(r'^[A-Z]{3}[0-9]{2}$')
HEALTH = {'Healthy', 'Watch', 'Breach', 'Unknown'}
REQUIRED = ['psi', 'ks', 'csiW']
OPTIONAL = ['edA', 'edE', 'edR', 'edM', 'exp', 'a30', 'a60', 'a90', 'ddR']   # delinquency: all nine or none

def fail(rule, msg):
    sys.exit(f'{rule} FAILED: {msg}; page not changed')

def validate(d):
    for k in ['meta', 'models', 'feat', 'views', 'port', 'portED', 'reg', 'tier', 'inv', 'biz']:
        if k not in d: fail('V-01', f'missing {k}')
    if not d['models'] or not d['tier'].get('models') or not d['inv']: fail('V-01', 'empty section')
    m = d['meta']
    if m.get('schema') != 'smokey-data/1.0': fail('V-02', 'schema')
    try:
        if datetime.date.fromisoformat(m['as_of']) > datetime.date.today(): fail('V-02', 'as_of in the future')
    except (KeyError, ValueError): fail('V-02', 'as_of')
    ids = [x['id'] for x in d['models']]
    if len(set(ids)) != len(ids): fail('V-04', 'duplicate model key')
    tier_ids = {x['id'] for x in d['tier']['models']}
    for x in d['models']:
        i = x['id']
        if not KEY.match(i): fail('V-04', i)
        if x.get('h') not in HEALTH or x.get('tier') not in (1, 2, 3, 4): fail('V-05', i)
        if i not in tier_ids or i not in d['inv']: fail('V-06', i)
        have = [s for s in OPTIONAL if s in x]
        if have and len(have) != len(OPTIONAL): fail('V-03', f'{i} has some delinquency series but not all')
        for s in REQUIRED + have:
            a = x.get(s)
            if not isinstance(a, list) or len(a) != 12: fail('V-03', f'{i}.{s}')
        for c in x.get('csi', []):
            if len(c[3]) != 12: fail('V-03', f'{i}.csi.{c[0]}')
            if any(v is not None and not 0 <= v <= 5 for v in c[3]): fail('V-09', f'{i}.csi.{c[0]}')
        if any(v is not None and not 0 <= v <= 5 for v in x['psi']): fail('V-09', f'{i}.psi')
        if x['h'] != 'Unknown' and x['psi'][-1] is None: fail('V-10', i)
    known = set(ids) | tier_ids
    by = {x['id']: x for x in d['models']}
    for f in d['feat']:
        for k, _ in f.get('live', []):
            if k not in known: fail('V-07', f'feat {f["f"]} -> {k}')
            if k in by and f['f'] not in [c[0] for c in by[k]['csi']]: fail('V-08', f'{k} lacks {f["f"]}')
    for r in d['reg']:
        if r['model_id'] not in known: fail('V-07', f'reg -> {r["model_id"]}')

def main(page, block, accept_change=False):
    html = pathlib.Path(page).read_text(encoding='utf-8')
    if html.count(OPEN) != 1: sys.exit('page has no single smokey-data block; page not changed')
    a = html.index(OPEN) + len(OPEN); b = html.index(CLOSE, a)
    new = json.loads(pathlib.Path(block).read_text(encoding='utf-8'))
    validate(new)
    old = json.loads(html[a:b].replace('<\\/', '</'))
    n0, n1 = len(old['models']), len(new['models'])
    if abs(n1 - n0) > 0.25 * n0 and not accept_change: fail('V-12', f'model count {n0} -> {n1}')
    text = json.dumps(new, separators=(',', ':'), ensure_ascii=False, sort_keys=True).replace('</', '<\\/')
    if 'NaN' in text or 'undefined' in text: fail('V-11', 'NaN or undefined')
    outside = lambda h, i, j: hashlib.sha256((h[:i] + h[j:]).encode()).hexdigest()
    out = html[:a] + text + html[b:]
    if outside(html, a, b) != outside(out, a, a + len(text)): sys.exit('outside-block hash changed; page not changed')
    hist = pathlib.Path(page).parent / 'history'; hist.mkdir(exist_ok=True)
    stamp = datetime.datetime.now().strftime('%Y%m%dT%H%M%S')
    shutil.copy2(page, hist / f'smokey-demo.{old["meta"].get("as_of","na")}.{stamp}.html')
    pathlib.Path(page).write_text(out, encoding='utf-8')
    print(json.dumps({'models': n1, 'as_of': new['meta']['as_of'], 'outside_sha256': outside(out, a, a + len(text)),
                      'block_sha256': hashlib.sha256(text.encode()).hexdigest()}))

if __name__ == '__main__':
    main(sys.argv[1], sys.argv[2], '--accept-change' in sys.argv)
```

## Appendix B: standing instructions for Cortex (`AGENTS.md`)

```markdown
# SMOKEY demo refresh

You refresh one data block in one HTML page. Follow these steps and no others.

1. Use role SMOKEY_REFRESH_RO. If the session has any other role, stop and say so.
2. Run each statement in refresh/queries.sql unchanged. Save each result to refresh/out/<as_of>/<view>.json. Record the query IDs.
3. Run: python refresh/build_block.py refresh/out/<as_of>  (writes block.json)
4. Run: python refresh/splice.py smokey-demo.html refresh/out/<as_of>/block.json
5. Write refresh/out/<as_of>/report.md from the script outputs: role, warehouse, query IDs, row counts, validation results, hashes.

Never:
- type, round, estimate or correct a figure yourself;
- edit smokey-demo.html with your own file tools;
- change queries.sql, the scripts or this file during a run;
- query anything outside schema SMOKEY_DEMO;
- publish, upload or send the page anywhere;
- continue after a script exits with an error. Report the error text and stop.
```

Batch invocation (CLI), to be confirmed against the installed version:

```bash
cortex -p "Refresh the SMOKEY demo for the latest as-of date. Follow AGENTS.md." \
  --allowed-tools "read" "SnowflakeSqlExecute" "bash" --disallowed-tools "write" "edit" "WebSearch" "WebFetch"
```

## Appendix C: reliability of what this spec relies on
| Item | Basis | Confidence |
|---|---|---|
| Page contract, field names, the three labels, focus fallback | Built and tested in `smokey-demo.html` on 25 Nov 2026 | High |
| Screen binding table | Code scan plus the renamed-key test | Medium; confirm in phase 1 |
| Cortex capabilities and flags | Snowflake documentation, read as extracts on 25 Nov 2026 | Medium; flag syntax in appendix B is unverified |
| Source tables behind the views | Not known | To be supplied by the bank |
| Minimum cell count of 20 | Proposal | Data owner decides |
