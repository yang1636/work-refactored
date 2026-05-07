# RE NXT → Domo Data Pipeline

Ingests constituent, gift, and fundraising data from the Blackbaud SKY API into Domo datasets using scheduled Jupyter notebooks.

---

## Architecture overview

```
Blackbaud SKY API
       │
       ▼
renxt_core.ipynb  ←── shared library (token management, fetch engine, write helpers)
       │
       ▼
schedule_raw_re_*.ipynb  ──►  Domo datasets (raw_re_*)
       │
       ▼
renxt_pipeline_state  ──►  watermark tracking (last successful run per endpoint)
```

All notebooks share two base files loaded via `%run` at the top of every notebook:

- **`user_configuration.ipynb`** — loads credentials from the Domo account store (`re_creds`), defines API constants and file paths
- **`renxt_core.ipynb`** — all shared logic: token manager, HTTP helpers, fetch engine, schema helpers, smart upsert, watermark read/write

---

## Notebooks

### Utility notebooks (run manually, never scheduled)

| Notebook | Purpose |
|---|---|
| `first_time_auth.ipynb` | One-time interactive OAuth authorisation. Run once to obtain and persist the Blackbaud refresh token. |
| `domo_env_check.ipynb` | Sanity check — confirms credentials load and the API is reachable. Run after any credential change. |
| `hist_raw_re.ipynb` | Manual full historic pull for any endpoint. Set `ENDPOINT_TO_PULL` and `DATASET_ID`, run all cells. Used for first-time backfill or rebuilding a dataset. |
| `subscribe_webhooks.ipynb` | Registers Blackbaud webhook subscriptions. Run manually when adding new event type subscriptions. |

### Scheduled notebooks

Each notebook is deployed in its own Domo Jupyter workspace and runs on a daily schedule.

| Notebook | Blackbaud endpoint | Domo dataset | Update method |
|---|---|---|---|
| `schedule_raw_re_constituents.ipynb` | `/constituent/v1/constituents` | `9b2cc153-...` (raw_re_constituents) | native upsert on `id` |
| `schedule_raw_re_actions.ipynb` | `/constituent/v1/actions` | `d6047fa0-...` | smart upsert on `id` |
| `schedule_raw_re_addresses.ipynb` | `/constituent/v1/addresses` | `b68978d8-...` | smart upsert on `id` |
| `schedule_raw_re_constituentcodes.ipynb` | `/constituent/v1/constituents/constituentcodes` | `e76d5758-...` | smart upsert on `id` |
| `schedule_raw_re_customfields.ipynb` | `/constituent/v1/constituents/customfields` | `6cf69420-...` | smart upsert on `id` |
| `schedule_raw_re_emailaddresses.ipynb` | `/constituent/v1/emailaddresses` | `02d4c77f-...` | smart upsert on `id` |
| `schedule_raw_re_phones.ipynb` | `/constituent/v1/phones` | `99786b37-...` | smart upsert on `id` |
| `schedule_raw_re_gifts.ipynb` | `/gift/v1/gifts` | `raw_re_gifts` | native upsert on `id` |
| `schedule_raw_re_gift_customfields.ipynb` | `/gift/v1/gifts/customfields` | `raw_re_gift_customfields` | native upsert on `id` |
| `schedule_raw_re_appeal.ipynb` | `/fundraising/v1/appeals` | `6bbcf6f9-...` | full REPLACE |
| `schedule_raw_re_campaign.ipynb` | `/fundraising/v1/campaigns` | `b0f7ea3c-...` | full REPLACE |
| `schedule_raw_re_fund.ipynb` | `/fundraising/v1/funds` | `db3b695a-...` | full REPLACE |
| `schedule_raw_re_opp_custom_fields.ipynb` | `/opportunity/v1/opportunities/{id}/customfields` | `raw_re_opportunity_custom_fields` | full REPLACE |
| `schedule_raw_re_comm_preferences.ipynb` | `/constituent/v1/constituents/{id}/communicationpreferences` | `raw_re_communicationpreferences` | full REPLACE |

### Event-driven notebooks (run on trigger or manually)

| Notebook | Purpose |
|---|---|
| `blackbaud_solicit_code_webhook_events_sync.ipynb` | Reads Blackbaud webhook events from `raw_blackbaud_webhook_events`, fetches updated communication preferences for affected constituents, and upserts into `raw_re_communicationpreferences`. |
| `sfmc_sync_re_email_status.ipynb` | Reads SFMC opt-out and bounce event datasets, computes which constituent emails need updating, and POSTs communication preference and action records back to Blackbaud SKY API. |

---

## How incremental loading works

Each scheduled notebook uses a **watermark** stored in the `renxt_pipeline_state` Domo dataset to determine what date range to fetch.

### Run flow

```
1. Read renxt_pipeline_state → get last_successful_run_utc for this endpoint
2. Compute since date:
     - If watermark exists  → use last_successful_run_utc
     - If no watermark yet  → fall back to 7 days ago
3. Fetch from Blackbaud API with ?last_modified=<since_date>
4. Apply schema transformations
5. Write to Domo dataset
6. Update renxt_pipeline_state:
     - Success → advance watermark to today
     - Failure → preserve existing watermark (next run re-fetches from last good point)
```

### Missed runs / failures

If a scheduled run fails or is skipped, the watermark does not advance. The next successful run automatically fetches everything since the last good run — no manual intervention or backfill needed.

### First run of a new endpoint

On the first run, there is no watermark. The notebook falls back to 7 days and fetches data from that point. If a full historic load is needed first, use `hist_raw_re.ipynb` to backfill the dataset, then let the scheduled notebook take over for incremental updates.

---

## Update methods explained

### `smart_upsert`
Used for: actions, addresses, constituentcodes, customfields, emailaddresses, phones.

Reads the existing Domo dataset, merges the new incremental pull in Python (new values win, existing values fill nulls in new rows), then writes the full result back as REPLACE. Used when incremental API responses may omit optional fields on modified records — the merge preserves any existing non-null values that the new pull doesn't return. Also automatically deduplicates on `id` at read time and before write.

### `upsert` (native Domo)
Used for: constituents, gifts, gift_customfields.

Passes `update_method="upsert"` directly to `domo.write_dataframe`. Domo handles the merge server-side — matched rows are updated, new rows are inserted, unmatched existing rows are left alone. Faster than `smart_upsert` on large datasets because no full read is required.

Constituents also applies a strict schema conformance step (`conform_df_to_schema`) before writing, ensuring exact column types and order match the Domo dataset schema.

### `REPLACE`
Used for: appeal, campaign, fund, opp_custom_fields, comm_preferences.

Full overwrite on every run. Used for reference/lookup tables (appeal, campaign, fund) where the dataset is small and always fetched in full, and for ID-based endpoints (opp_custom_fields, comm_preferences) where every run re-fetches all records for all IDs.

---

## Token management

Blackbaud uses OAuth 2.0 with rotating refresh tokens. Because Domo Jupyter containers are ephemeral (filesystem is wiped after each run), tokens cannot be stored in files between runs.

The pipeline uses two persistent stores:

- **`renxt_token_store`** Domo dataset — stores the latest refresh token. Written after every successful token refresh so the next scheduled run can read it.
- **`re_creds` Domo account store** — stores static credentials (client ID, client secret, subscription key). Also used as a manual fallback for the initial refresh token before `renxt_token_store` is populated.

### Token refresh flow

```
1. Read BB_REFRESH_TOKEN from re_creds account store (or env var)
2. If not found, read from renxt_token_store dataset
3. Exchange refresh token for new access token via Blackbaud OAuth endpoint
4. Write new refresh token back to renxt_token_store
5. Cache access token in local file for duration of this run
6. On 401/403 during API calls → auto-refresh and retry once
```

---

## Domo workspace setup

Each scheduled notebook must be deployed in its own Domo Jupyter workspace (Domo only supports one scheduled dataflow per workspace).

For every workspace, configure the following in the workspace settings:

**Credentials (Accounts):**
- `re_creds` — Blackbaud API credentials (BB_CLIENT_ID, BB_CLIENT_SECRET, BB_SUBSCRIPTION_KEY, BB_REDIRECT_URI, BB_REFRESH_TOKEN)

**Input datasets:**
- `renxt_token_store`
- `renxt_pipeline_state`
- The relevant source dataset (e.g. `raw_re_opportunities` for opp_custom_fields)

**Output datasets:**
- `renxt_token_store`
- `renxt_pipeline_state`
- The target dataset for this notebook

---

## Shared infrastructure datasets

| Dataset | Purpose | Access |
|---|---|---|
| `renxt_token_store` | Persists the Blackbaud OAuth refresh token across container restarts | Input + Output on all workspaces |
| `renxt_pipeline_state` | Watermark tracking — one row per endpoint with last successful run date and status | Input + Output on all workspaces |

### `renxt_pipeline_state` schema

| Column | Type | Description |
|---|---|---|
| `endpoint` | text | Endpoint name e.g. `actions` |
| `last_successful_run_utc` | text | Date of last successful run — `YYYY-MM-DD` |
| `last_run_status` | text | `success` or `failed` |
| `rows_written` | decimal | Row count from last run |
| `updated_at` | text | Full ISO timestamp of last state update |

---

## Blackbaud API endpoints

All endpoints include `include_inactive=true` so inactive records are captured. The `constituents` endpoint also includes `include_deceased=true`.

| Endpoint name | API path | Incremental field |
|---|---|---|
| constituents | `/constituent/v1/constituents` | last_modified |
| actions | `/constituent/v1/actions` | last_modified |
| addresses | `/constituent/v1/addresses` | last_modified |
| constituentcodes | `/constituent/v1/constituents/constituentcodes` | last_modified |
| customfields | `/constituent/v1/constituents/customfields` | last_modified |
| educations | `/constituent/v1/educations` | last_modified |
| emailaddresses | `/constituent/v1/emailaddresses` | last_modified |
| memberships | `/constituent/v1/memberships` | last_modified |
| notes | `/constituent/v1/notes` | last_modified |
| onlinepresences | `/constituent/v1/onlinepresences` | last_modified |
| phones | `/constituent/v1/phones` | last_modified |
| relationships | `/constituent/v1/relationships` | last_modified |
| opportunity | `/opportunity/v1/opportunities` | last_modified |
| gifts | `/gift/v1/gifts` | last_modified |
| gift_customfields | `/gift/v1/gifts/customfields` | last_modified |
| appeal | `/fundraising/v1/appeals` | none — always full pull |
| campaign | `/fundraising/v1/campaigns` | none — always full pull |
| fund | `/fundraising/v1/funds` | none — always full pull |

API rate limit is 5 calls per second. The fetch engine handles throttling, retries on 429/5xx with exponential backoff, and automatic token refresh on 401/403.

---

## Adding a new endpoint

1. Add an entry to the `ENDPOINTS` list in `renxt_core.ipynb`
2. Create a new `schedule_raw_re_<name>.ipynb` by copying an existing one and updating `ENDPOINT_NAME`, `DATASET_ID`, and `UPDATE_METHOD`
3. Create the target Domo dataset
4. Create a new Domo Jupyter workspace, add the dataset as Input/Output, add `renxt_token_store` and `renxt_pipeline_state` as Input/Output
5. Run once manually to verify, then schedule

---

## Troubleshooting

**"refresh token not valid"**
The refresh token in `renxt_token_store` has expired or been invalidated. Run `first_time_auth.ipynb` interactively to re-authorise and write a fresh token.

**"output datasource with id or alias X missing or not configured"**
The dataset is not registered as an Output Dataset on this workspace. Go to workspace settings → Output DataSets and add it.

**Duplicate rows in a dataset**
Caused by a previous bug in the `smart_upsert` logic. The current version automatically deduplicates on read — the next successful run of the affected notebook will clean up legacy duplicates and log `⚠️ Removed N duplicate rows from existing data`.

**Watermark not advancing**
Check the notebook output for `❌ Domo write failed`. The watermark is only written after a successful Domo write — if the write fails, the watermark is preserved so the next run re-fetches from the last good point.

**"No watermark for [endpoint] — using 7d fallback"**
Expected on first run of an endpoint. After the first successful run the watermark is written and future runs use it. If you need data older than 7 days, run `hist_raw_re.ipynb` first.
