# TRD: Sample Data Feature (Frontend)

**Repo:** `GameTrainer/GameTrainer-Frontend`  
**Feature:** “Try Sample Hands” (sample dataset that new users can load as default)  
**PRD Source:** `PRD-Sample-Data.md` (2026-01-05)  
**Backend TRD Source:** `TRD-Sample-Data-Backend.md`  
**Generated:** 2026-01-06  

---

## 1) Goal

Enable a new user (0 uploads) to click **“Try Sample Hands”** and immediately see:
- A batch of sample “uploaded files” appear in the **same upload surfaces** as real uploads (Navbar upload badge/panel + DB table).
- Real-time progress (WebSocket-driven), identical to normal upload experience.
- Once processing completes, all “Sessions / Reports / Replayer” screens behave normally because the data exists server-side.

Non-goals (frontend):
- Shipping any sample hand history files inside the web bundle.
- Re-implementing upload parsing client-side.
- Any data privacy / masking (handled by dataset curation + backend).

---

## 2) Product constraints (from PRD)

Key PRD constraints relevant to frontend:
- Sample import uses the **native upload flow** and should look/behave like a real upload.
- Sample data **persists until explicitly removed** (user can later delete sample dataset).
- Must **coexist** with user’s real uploads (sample shouldn’t block real uploads).
- Dataset is **~500 hands**, pre-analyzed.
- Sample hand history files are stored on backend (S3), not shipped by FE.

---

## 3) Current frontend architecture (what we are integrating with)

### 3.1 Upload system (already exists)

Frontend already has a robust upload system:
- `uploadManager` singleton orchestrates uploads and syncs to Redux (optimistic jobs) and listens to WebSocket updates.
- Redux sources of truth:
  - `uploadReducer.uploads`: optimistic uploads in-flight
  - `replayerReducer.playerHandProcessing`: authoritative list of transactions from API + WebSocket merging
- UI surfaces:
  - Navbar badge/panel (`components/Navbar/*`) displays current processing / progress.
  - DB table pages (`components/UploadHands/*` + `components/HandProcessing/*`) show all uploaded files.

See:
- Upload manager definition: `store/uploadManager.ts` (core types + logic)
- WebSocket normalization: `utils/fileStatusNormalizer.ts` (maps WS `status` strings to `Processing` values)
- Empty state: `components/UploadHands/EmptyLanding.tsx` (rendered when the user has 0 uploads)

### 3.2 Fetching processing list

- `store/asyncActions.ts` contains `getPlayerHandsProcessing` thunk using `PLAYER_HANDS_PROCESSING` endpoint.
- The app uses silent refresh patterns + WebSocket updates (see navbar code and docs).

---

## 4) Desired UX & state machine (frontend)

### 4.1 Entry points

**Primary entry point:** empty state screen (0 hands uploaded)
- Add button: **“Try Sample Hands”** (secondary primary action next to Upload)
- Clicking triggers backend import.

**Secondary entry point (optional, if PRD wants persistent access):**
- If sample is not installed but user is no longer “empty”, we may still want an entry point (e.g., in upload drawer or a settings menu).
- For v1, implement only empty-state entry point unless product requests more.

### 4.2 State & user feedback

When user clicks “Try Sample Hands”:
1. Disable the button, show “Starting…” spinner.
2. Call `POST /sample-data/import`.
3. On success:
   - Immediately refetch `getPlayerHandsProcessing(page=1, perPage=PAGINATION_LIMIT, sortBy=UploadDate desc)` to show transactions (queued/processing).
   - Open upload drawer OR show toast “Sample hands added — processing started”.
4. Progress updates continue through WebSocket, no special-case UI needed.
5. On failure:
   - Show toast error.
   - Re-enable button.

When sample data is already installed:
- Hide the button OR replace with “Sample hands already loaded”.
- Display is controlled by `GET /sample-data/status`.

---

## 5) API contract (frontend-facing)

Per backend TRD, frontend must integrate 3 endpoints:

### 5.1 `GET /sample-data/status`
Used to decide whether to show CTA and to show “Remove sample data” actions.

Expected response shape (from backend TRD intent):
```json
{
  "installed": true,
  "installing": false,
  "sample_transaction_ids": ["..."],
  "sample_database_id": "..." 
}
```

Frontend requirements:
- Cache for the session (RTK query or local slice cache).
- Refresh after:
  - successful import
  - successful remove
  - page reload

### 5.2 `POST /sample-data/import`
Starts the backend import workflow.

Expected behavior:
- Returns quickly (kicks off async work).
- After success, processing transactions should appear in `getPlayerHandsProcessing` and WebSocket should stream progress updates like normal uploads.

### 5.3 `POST /sample-data/remove`
Removes sample dataset and its derived artifacts.

Frontend requirements:
- Confirmation dialog (destructive action).
- On success, refetch:
  - `getPlayerHandsProcessing` (list should shrink)
  - any session summary endpoints used on landing pages
  - sample status

---

## 6) Implementation plan (frontend)

### 6.1 Add endpoints to frontend API layer

There are two existing patterns in this repo:
- **RTK thunks** in `store/asyncActions.ts` using `apiServiceInstance`
- **RTK Query slices** (used for some modules like invoice)

For this feature, use the thunk approach to match existing upload-related patterns.

#### Changes
1) Add new endpoint constants in `store/apiEndpoints.ts`:
- `SAMPLE_DATA_STATUS = '/sample-data/status'`
- `SAMPLE_DATA_IMPORT = '/sample-data/import'`
- `SAMPLE_DATA_REMOVE = '/sample-data/remove'`

2) Add API functions/thunks in `store/asyncActions.ts`:
- `getSampleDataStatus` (GET)
- `importSampleData` (POST)
- `removeSampleData` (POST)

Notes:
- Keep return types minimal and typed in `store/types.ts` if that file contains API DTOs; otherwise inline types in asyncActions with `any` (consistent with current file style).

### 6.2 Frontend state: where to store “sample status”

Add a small slice to hold sample data state:
- `sampleDataSlice` (new) with:
  - `installed: boolean`
  - `installing: boolean`
  - `isLoadingStatus: boolean`
  - `lastError?: string`
This avoids sprinkling `useState` flags across components and supports other entry points later.

Where:
- `store/slices/sampleDataSlice.ts` (new)
- register in `store/index.ts` root reducer
- optional persistence (not required; backend is authoritative; always refetch status after login).

### 6.3 Empty state CTA integration

Modify `components/UploadHands/EmptyLanding.tsx`:
- On mount: dispatch `getSampleDataStatus` (or trigger in parent page if already fetching).
- Render a new button:
  - Label: `Try Sample Hands`
  - Icon (optional): a “sparkle” / “lightning” icon (consistent style; repo already uses `HiOutlineLightningBolt` elsewhere).
- Click handler:
  - dispatch `importSampleData()`
  - show toast on success
  - dispatch `getPlayerHandsProcessing(...)` to populate DB table
  - optionally call `showDrawer()` to open upload drawer so user sees the processing list immediately

Edge cases:
- If `installing=true`, show disabled “Preparing sample hands…”
- If `installed=true`, hide CTA.

### 6.4 “Remove sample data” UX surface

The PRD says sample persists until removed; we need a removal affordance.

Best v1 location:
- Uploads database page (where user sees files): `components/UploadHands/UploadHandTables.tsx` (or nearby toolbar component)

Implementation:
- If `installed=true`, show a small action button in the table header: “Remove sample hands”
- On click:
  - Confirmation modal (Antd `Modal.confirm`)
  - Block if there are active uploads/processing in progress (optional but recommended): check `replayerReducer.playerHandProcessing` for `Processing < 4` and show message.
  - dispatch `removeSampleData()`
  - then refetch:
    - `getPlayerHandsProcessing` (page 1)
    - `getSampleDataStatus`

If multi-database feature is active in the same timeframe:
- Prefer to remove sample **database** rather than individual transactions. Frontend still only calls `/sample-data/remove`.

### 6.5 Analytics events

Track:
- `sample_data_try_clicked` (from empty landing)
- `sample_data_import_success`
- `sample_data_import_failed`
- `sample_data_remove_clicked`
- `sample_data_remove_success`
- `sample_data_remove_failed`

Use existing `track()` helper (already used in upload flows).

### 6.6 i18n

Add translations keys (en default) for:
- `landing.trySampleHands` = “Try Sample Hands”
- `landing.sampleHandsPreparing` = “Preparing sample hands…”
- `landing.sampleHandsAdded` = “Sample hands added — processing started”
- `landing.sampleHandsAddFailed` = “Couldn’t add sample hands”
- `uploadHands.removeSampleHands` = “Remove sample hands”
- `uploadHands.removeSampleHandsConfirmTitle` = “Remove sample hands?”
- `uploadHands.removeSampleHandsConfirmBody` = “This will delete the sample dataset and all associated sessions/reports.”
- `uploadHands.removeSampleHandsBlocked` = “You can’t remove sample hands while files are processing.”

Where translations live depends on repo conventions (typically `public/locales/*/*.json` or similar).

---

## 7) Detailed flows affected (frontend)

### Flow A: Empty user loads landing page
1. App calls `getPlayerHandsProcessing` (returns empty).
2. Empty landing renders.
3. App calls `getSampleDataStatus` (optional; but required before showing CTA).
4. CTA “Try Sample Hands” visible if `installed=false`.

### Flow B: User clicks “Try Sample Hands”
1. UI disables button, shows spinner.
2. POST `/sample-data/import`
3. On success:
   - show toast
   - dispatch `getPlayerHandsProcessing` (page 1)
   - optional: open upload drawer
4. WebSocket updates update processing rows (no special logic).

### Flow C: User removes sample data
1. User clicks “Remove sample hands”
2. Confirmation modal
3. POST `/sample-data/remove`
4. On success:
   - toast
   - refetch `getPlayerHandsProcessing`
   - refetch `getSampleDataStatus`
5. If list becomes empty, Empty landing renders again.

---

## 8) File-by-file impact (frontend)

### New files
- `store/slices/sampleDataSlice.ts`  
  Holds sample status state and reducers for loading/error flags.

### Modified files (expected)

**API layer**
- `store/apiEndpoints.ts`  
  Add `SAMPLE_DATA_STATUS`, `SAMPLE_DATA_IMPORT`, `SAMPLE_DATA_REMOVE`.
- `store/asyncActions.ts`  
  Add `getSampleDataStatus`, `importSampleData`, `removeSampleData` thunks.

**Redux wiring**
- `store/index.ts` (or root reducer file)  
  Register `sampleDataSlice.reducer`.

**UI**
- `components/UploadHands/EmptyLanding.tsx`  
  Render CTA + call status + import action.
- `components/UploadHands/UploadHandTables.tsx` (or the nearest toolbar component)  
  Add “Remove sample hands” action and wire remove flow.

**Types (if strict typing is used)**
- `store/types.ts`  
  Add `SampleDataStatusResponse` type.

**Tests**
- Add unit tests for CTA rendering + click behavior:
  - `__tests__/.../EmptyLanding.test.tsx` (new)
  - Update any existing upload-related tests if global store shape changes.
- If you already have `Navbar.test.tsx`, ensure new slice doesn’t break test store mocks.

---

## 9) Error handling & resiliency

### Import
- If import returns 409 “already installed”:
  - Treat as success, refresh status + processing list.
- If backend returns 503/500:
  - Show error and allow retry.

### Status calls
- If status endpoint fails, default to showing CTA (optimistic onboarding) OR hide CTA (safe).
  - Recommendation: default to showing CTA but guard import call with proper error messages.

### Remove
- If remove fails:
  - Show error, keep status state unchanged (still installed).

---

## 10) Security & permissions

Frontend assumes backend enforces:
- Authenticated user required.
- Only the owning user can import/remove their sample dataset.

Frontend must:
- Send auth headers via existing `apiServiceInstance` (already configured).

---

## 11) QA checklist

- Empty user sees CTA.
- Clicking CTA triggers import and populates DB table within a refresh.
- Upload badge/panel reflects sample transactions as they process.
- User can still upload real hands while sample is processing.
- Refresh page mid-processing: status + processing list rehydrates correctly; WebSocket continues.
- Remove sample hands clears them and returns to empty landing.

---

## 12) Notes on multi-database roadmap

This repo already contains a `PRD_MULTI_DATABASE.md` with a “Sample Data Feature” section; if multi-db is implemented first:
- Sample data likely lives in a dedicated “Sample Data” database.
- The same FE integration remains, but removal UX should operate at DB level (still via `/sample-data/remove`).

Keep this TRD compatible by treating `/sample-data/*` as the authoritative interface.

---

## 13) Implementation order (recommended)

1. Add API endpoint constants.
2. Add asyncActions thunks for status/import/remove.
3. Add `sampleDataSlice`.
4. Wire EmptyLanding CTA + optimistic UX.
5. Add remove action in UploadHandTables.
6. Add analytics + i18n keys.
7. Add tests.

---

## 14) Appendix: Concrete payload examples

### Import (request)
`POST /sample-data/import`
```http
Authorization: Bearer <token>
Content-Type: application/json

{}
```

### Remove (request)
`POST /sample-data/remove`
```http
Authorization: Bearer <token>
Content-Type: application/json

{}
```

### Status (request)
`GET /sample-data/status`
```http
Authorization: Bearer <token>
```

---
