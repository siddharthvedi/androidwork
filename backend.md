# TRD: Sample Dataset (Default/Importable Sample Hands) — Backend

> Repo: `GameTrainer/GameTrainer-Backend`  
> Target branch: `feature/reports` (per team convention)  
> Document: Technical Requirements / Design (TRD)  
> Source: PRD `PRD-Sample-Data.md`

---

## 1. Goal

Implement **sample hands dataset** support in the backend so that:
- A user can **import sample data** (per game variant) via a backend API.
- Sample data appears in the UI **as normal uploaded hands** (same upload/processing UX).
- User can **remove sample data** later.
- Sample data **persists** across sessions/devices until removed.
- Re-import rules follow PRD constraints (notably: reimport allowed if user has <100 of their own uploaded hands).

This TRD focuses on backend behavior, data model, API endpoints, storage strategy, and all files/flows impacted.

---

## 2. Non-goals

- Frontend UI changes (upload panel / empty state) except noting required API contracts.
- Generating / curating the sample hand histories themselves (we only define how they are stored and consumed).
- Multi-tenant “global sample feed” across all users (sample is *per-user*, created on demand).

---

## 3. Existing backend architecture (relevant)

### 3.1 Upload ingestion pipeline (current)
The existing upload flow:
1. API receives one or more files (`.txt`, `.zip`) and normalizes/validates them.
2. Each file is zipped (if needed), uploaded to **S3_USER_HANDS_BUCKET**, and a `PlayerHandsUploadsTransaction` row is created/updated.
3. A `PlayerHandsUploadFileVersion` is created referencing the S3 URL.
4. Websocket “queued” status is sent.
5. Downstream processing happens asynchronously via S3 + SQS ingestion.

Key components:
- `hand_analysis/services/upload_flow.py` (core upload logic, S3 upload, transaction creation)
- `hand_analysis/models/transactions.py` (`PlayerHandsUploadsTransaction`, `PlayerHandsUploadFileVersion`)
- `hand_analysis/tasks/sqs_ingestion.py` (SQS worker calls `contracts.save_raw_hand`)
- `hand_analysis/ingestion/status.py` (WebSocket updates)

### 3.2 Deletion
Transactions and related file versions are deleted via:
- `hand_analysis/api/transactions.py` `_delete_transactions_for_user(...)` and endpoints under `hand_analysis/urls.py`.
- `PlayerHandsUploadFileVersion` has a `pre_delete` hook that deletes the S3 folder if the URL points to `S3_USER_HANDS_BUCKET`.

This is good for sample data removal: deleting sample transactions will delete their S3 payloads automatically.

---

## 4. Backend design for Sample Dataset

### 4.1 High-level approach
**Treat sample dataset as a server-triggered “upload” performed on behalf of the user**, reusing the existing upload pipeline to ensure:
- identical transaction records,
- identical S3 layout,
- identical SQS ingestion & analysis behavior,
- identical websocket progress reporting.

Implementation:
- Store curated sample files (zip or txt) in a **shared sample source** (recommended: separate S3 bucket or prefix).
- On import, backend **downloads sample files**, wraps them as Django `UploadedFile` objects, and calls `process_uploads()` from the existing upload flow.
- Transactions created by sample import are marked with a dedicated **Site enum value**: `SiteNameEnum.Sample`.

### 4.2 Why this approach (best practice rationale)
- **Avoids introducing a parallel ingestion pipeline** (less risk, less code).
- Guarantees sample behaves like real user data (same limits, same normalization, same analytics).
- Allows easy cleanup by deleting the transactions (leveraging existing FK cascades + S3 cleanup hook).
- Supports future experimentation (A/B tests, multiple variants) by selecting different sample bundles.

---

## 5. Data model changes

### 5.1 Site enum: add `Sample`
Add a new enum member:
- `replayer/models/site_choices.py`: `SiteNameEnum.Sample = 62` (or another unused value < 63; `All` is 63)
- Display name: either add to `DISPLAY_NAME_BY_SITE` or rely on default enum `name` (“Sample”).

**Purpose**: transactions imported from sample will show `Site = "Sample"` in API & websocket payloads.

### 5.2 Sample data state tracking (new model)
We need to track:
- whether sample is currently “active/imported” for the user
- whether user removed it before (so we can persist “do not auto-show” / “do not auto-import” semantics)
- cohort / variant for A/B testing support (PRD scalability section)

Create:
`hand_analysis/models/sample_data.py`

Proposed model:

```py
class UserSampleDataState(models.Model):
    UserId = models.CharField(max_length=255, db_index=True)
    GameVariant = models.CharField(max_length=32, db_index=True)  # matches settings.GAME_VARIANT
    Cohort = models.CharField(max_length=64, blank=True, default='')
    Active = models.BooleanField(default=False)
    RemovedAt = models.DateTimeField(null=True, blank=True)
    ImportedAt = models.DateTimeField(null=True, blank=True)

    class Meta:
        unique_together = [('UserId', 'GameVariant')]
        db_table = 'hand_analysis_usersampledatastate'
```

Create migration under `hand_analysis/migrations/`.

### 5.3 No changes required to transactions tables
We do **not** need to modify `PlayerHandsUploadsTransaction` schema. Sample-ness is encoded via:
- `Site = Sample`
- deterministic sample file naming `[Sample] ...` (for UX clarity)
- optional `ProcessingMetadata` additions (e.g., `{"sample": true, "sample_key": "plo_6max_cash_100hands"}`) if desired

---

## 6. Storage & configuration

### 6.1 Where sample files live
Recommended:
- Add environment-backed settings:
  - `S3_SAMPLE_HANDS_BUCKET`
  - `S3_SAMPLE_HANDS_PREFIX` (e.g. `sample-hands/`)
- Sample files are stored as `.zip` (preferred) to reduce size and match accepted upload types.

Folder layout example:
```
s3://S3_SAMPLE_HANDS_BUCKET/sample-hands/
  plo/
    sample_plo_6max_cash_100hands.zip
  nlhe/
    sample_nlhe_6max_cash_100hands.zip
```

### 6.2 Mapping: sample dataset selection by game variant
We need a deterministic mapping:
- `PLO` variant -> `sample_plo_6max_cash_100hands.zip`
- `NLHE` variant -> `sample_nlhe_6max_cash_100hands.zip`
- Extendable later for more datasets / segmentation.

Store this mapping in:
- `hand_analysis/services/sample_data/catalog.py`

Example:

```py
SAMPLE_DATASETS = {
  "plo": [
    {"key": "plo_6max_cash_100hands", "filename": "sample_plo_6max_cash_100hands.zip",
     "display_name": "[Sample] PLO 6-Max Cash Session.txt"},
  ],
  "nlhe": [
    {"key": "nlhe_6max_cash_100hands", "filename": "sample_nlhe_6max_cash_100hands.zip",
     "display_name": "[Sample] NLHE 6-Max Cash Session.txt"},
  ],
}
```

---

## 7. API design

All endpoints are under the existing variant prefix (`/{settings.GAME_VARIANT}/...`) because current hand_analysis URLs are already variant-scoped.

### 7.1 Import sample data
**POST** `/{variant}/sample-data/import`

**Behavior**
- Validate authenticated user (`request.user.id`, `request.user.username` required).
- If user already has sample imported (state `Active=True`), return idempotent success.
- Enforce re-import rule:
  - If state was previously removed (`RemovedAt != null`) AND user has ≥100 non-sample hands, return `409` with message “Sample can only be re-imported if you have fewer than 100 uploaded hands”.
  - “Non-sample hands” = transactions for user where `Site != Sample` (or null) and/or `FileName` not starting with `[Sample]` (prefer Site-based).
- Download the sample file(s) from sample bucket and build `UploadedFile` objects.
- Call existing `process_uploads(context)` to upload to user S3 bucket and create transactions/file_versions.
- After successful import, update `UserSampleDataState.Active=True`, `ImportedAt=now`, `RemovedAt=null`.

**Response (200)**
```json
{
  "success": true,
  "imported": true,
  "transactions": [
    {"id": "<uuid>", "filename": "[Sample] PLO 6-Max Cash Session.txt"}
  ]
}
```

Notes:
- The existing upload endpoint returns `processed_files` map. We can optionally return that too for parity.

### 7.2 Remove sample data
**DELETE** `/{variant}/sample-data/remove`

**Behavior**
- Delete only sample transactions for the user:
  - `PlayerHandsUploadsTransaction.objects.filter(UserId=user_id, Site=SiteNameEnum.Sample.value).delete()`
- This cascades to hands (via FK) and file versions; file version delete hook removes S3 folders.
- Update `UserSampleDataState.Active=False`, `RemovedAt=now`.

**Response (200)**
```json
{ "success": true, "removed": true, "deleted_transactions": 2 }
```

### 7.3 Sample status (optional but strongly recommended)
**GET** `/{variant}/sample-data/status`

This endpoint allows FE to decide whether to show “Try Sample Hands”, “Remove Sample Data”, or “Re-import Sample Data”.

**Response**
```json
{
  "active": true,
  "removedPreviously": false,
  "canReimport": true,
  "userNonSampleTransactionCount": 12
}
```

---

## 8. Service implementation details

### 8.1 New service: `hand_analysis/services/sample_data/service.py`
Responsibilities:
- compute dataset selection based on game variant & cohort
- enforce rules (idempotency, re-import restriction)
- download sample zip(s) from sample bucket to temp files
- wrap them as `UploadedFile` and call `process_uploads`

Pseudo-flow:

```py
def import_sample_data(request):
    user_id = request.user.id
    username = request.user.username
    variant = settings.GAME_VARIANT

    state = UserSampleDataState.get_or_create(...)
    if state.Active: return already imported

    if state.RemovedAt and count_user_non_sample_transactions(user_id) >= 100:
        return 409

    files = download_sample_files_as_uploaded_files(variant)
    context = UploadRequestContext(user_id=user_id, username=username, files=files)
    outcome = process_uploads(context)

    if not outcome.success: return 500
    state.Active = True; state.ImportedAt = now; state.RemovedAt = None; state.save()

    return outcome + transaction metadata
```

### 8.2 Downloading from S3
Add helper:
- `hand_analysis/services/sample_data/storage.py`

Uses existing `AwsS3().download_file(...)` (already used elsewhere), downloads to a temp file. Then create `SimpleUploadedFile` or `InMemoryUploadedFile`:
- If the sample zip is not too large, use `SimpleUploadedFile` from bytes.
- Safer: stream to disk then use `UploadedFile`-compatible object; but the existing upload flow expects `.chunks()`, which `SimpleUploadedFile` provides.

### 8.3 Making sure sample files pass upload validations
Upload validations enforce:
- extension is allowed (`settings.HANDS_ALLOWED_EXTENSIONS`)
- file size min/max limits

Therefore curated sample zips must be:
- `.zip` (or `.txt`)
- size > `HANDS_FILE_UPLOAD_MIN_SIZE`
- inner `.txt/.xml` sizes within `HANDS_TXT_FILE_UPLOAD_MAX_SIZE` (zip validation checks inner file sizes)

---

## 9. Flows affected (end-to-end)

### Flow A — Import sample (empty state button)
1. FE calls `POST /{variant}/sample-data/import`
2. Backend downloads sample file(s), calls existing upload pipeline
3. Transaction(s) created (Site=Sample)
4. Websocket queued status is sent (same as normal upload)
5. SQS ingestion processes and saves raw hands / analysis
6. FE fetches transactions or receives websocket updates; sample appears as uploaded file row(s)

### Flow B — Remove sample data (upload panel action)
1. FE calls `DELETE /{variant}/sample-data/remove`
2. Backend deletes only sample transactions for the user
3. DB cascades delete hands; S3 cleanup occurs via file version pre_delete hook
4. Backend invalidates cache keys (reuse existing `delete_hands_cache_keys_by_user_id`)
5. FE refreshes list; sample disappears

### Flow C — Re-import sample data
1. FE checks `GET /{variant}/sample-data/status` (or handles errors from import)
2. If removedPreviously and user has <100 non-sample uploads, FE enables reimport button
3. FE calls import again

---

## 10. Files & modules impacted (complete list)

### Existing files to modify
1. `replayer/models/site_choices.py`
   - Add `SiteNameEnum.Sample`
   - (Optional) add display name mapping

2. `hand_analysis/urls.py`
   - Add URL routes for sample data endpoints

3. `hand_analysis/__init__.py` / `hand_analysis/apps.py` (only if needed to auto-import new models; usually not necessary)

4. **(Security/Correctness fix recommended)** `hand_analysis/api/transactions.py`
   - `list_uploads_transaction` currently returns **all users**. Update it to filter by `UserId=request.user.id` to avoid data leakage and to keep sample UX correct (sample should be per-user).

### New files to add
1. `hand_analysis/models/sample_data.py`
2. `hand_analysis/migrations/00xx_usersampledatastate.py`
3. `hand_analysis/services/sample_data/catalog.py`
4. `hand_analysis/services/sample_data/storage.py`
5. `hand_analysis/services/sample_data/service.py`
6. `hand_analysis/api/sample_data.py` (DRF views)
7. `hand_analysis/tests/api/test_sample_data_api.py`
8. `hand_analysis/tests/fixtures/sample_data/` (optional if storing local fixtures for tests)

### Tests to update
- If `list_uploads_transaction` is made user-scoped, adjust:
  - `hand_analysis/tests/api/test_upload_transactions.py` expectations for count (should only include the authenticated user’s rows).

---

## 11. Error handling & idempotency

- Import should be idempotent:
  - If sample already imported: return 200 with `{ imported: false, active: true }`
- If sample bucket missing / download fails: return 500 with safe error message.
- If upload flow rejects sample file due to size/format: return 500 and log the validation error (this is a configuration/content issue).
- Remove should be idempotent:
  - If no sample transactions exist: return 200 with deleted count 0, but set state to removed.

---

## 12. Observability

Add structured logging (logger `gametrainer`) for:
- Import start/end with user_id, variant, cohort, dataset keys
- Number of files imported and created transaction IDs
- Remove start/end, deleted counts
- Re-import blocked reasons (non-sample count >= 100)

Optional: Add Slack notification for repeated failures (pattern consistent with existing `post_to_slack` usage in some services).

---

## 13. Security considerations

- Sample endpoints require authentication.
- Deletion must scope strictly to `UserId=request.user.id`.
- Transaction listing endpoint must not leak other users’ transactions (strongly recommended fix).

---

## 14. Migration & rollout plan

1. Add `SiteNameEnum.Sample` (no DB migration needed).
2. Add `UserSampleDataState` migration.
3. Add endpoints and service.
4. Deploy sample source bucket + files (or ensure they exist).
5. Enable FE integration.

Feature flags:
- Add `SAMPLE_DATA_ENABLED` setting (default false) if you want to gate launch safely.

---

## 15. Test plan

### Unit tests
- Import:
  - creates `UserSampleDataState` and sets `Active=True`
  - creates sample transactions with `Site=Sample`
  - calls upload pipeline (mock S3 upload/download)
  - idempotent behavior when already active

- Remove:
  - deletes only sample transactions
  - leaves non-sample transactions intact
  - sets `Active=False`, `RemovedAt` set

- Re-import restriction:
  - state removed + non-sample count >=100 -> import returns 409

### Integration tests (optional)
- End-to-end with localstack / S3 emulator + SQS is possible but heavy; OK to keep unit tests with mocks.

---

## 16. Open questions / assumptions (captured for implementation)

- Exact game variants: assume `settings.GAME_VARIANT` values map to `plo`, `nlhe` keys.
- Where sample files will live in production: assumed S3 sample bucket.
- Whether FE strictly requires “sample-...” IDs: backend will return real UUID transaction IDs; FE should treat them as normal transaction rows. If FE needs a stable “sample id”, add `sample_key` in response and/or in `ProcessingMetadata`.

---

# Appendix A — Suggested endpoint wiring

Add to `hand_analysis/urls.py`:

```py
from hand_analysis.api.sample_data import import_sample_data, remove_sample_data, sample_data_status

urlpatterns += [
  path('sample-data/import', import_sample_data, name='import_sample_data'),
  path('sample-data/remove', remove_sample_data, name='remove_sample_data'),
  path('sample-data/status', sample_data_status, name='sample_data_status'),
]
```

---

# Appendix B — Backend best practices checklist (for this feature)

- Reuse existing upload/ingestion pipeline
- Keep sample per-user and deletable
- Idempotent APIs
- Strict user scoping in queries
- Config-driven dataset selection
- Track user state in DB (not in-memory)
- Observability + safe error responses
