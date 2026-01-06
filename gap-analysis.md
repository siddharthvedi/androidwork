# Sample Data Feature - PRD/TRD Gap Analysis

**Review Date:** 2026-01-06  
**Reviewer:** AI Assistant  
**PRD Source:** [PRD: Sample Data Feature](https://github.com/siddharthvedi/androidwork/blob/main/prd.md)  
**TRD Sources:** 
- [TRD: Sample Data Feature (Frontend)](https://github.com/siddharthvedi/androidwork/blob/main/frontend.md)
- [TRD: Sample Dataset (Backend)](https://github.com/siddharthvedi/androidwork/blob/main/backend.md)

---

## Executive Summary

The current implementation successfully delivers all 6 user flows from the PRD using a **prototype approach** (client-side file fetching from `/public/sample-hands/`). However, it does **not** align with the TRD's production architecture, which requires backend API endpoints for status checking, import, and removal.

**Overall Compliance:**
- ✅ **PRD Compliance:** 100% (All 6 flows implemented)
- ⚠️ **TRD Compliance:** ~60% (UI/UX complete, API integration missing)
- ⚠️ **Production Readiness:** Prototype only (files shipped in bundle)

---

## 1. API Integration Gaps

### 1.1 Missing API Endpoint Constants

**TRD Requirement (Section 6.1):**
- Add endpoint constants in `store/apiEndpoints.ts`:
  - `SAMPLE_DATA_STATUS = '/sample-data/status'`
  - `SAMPLE_DATA_IMPORT = '/sample-data/import'`
  - `SAMPLE_DATA_REMOVE = '/sample-data/remove'`

**Current State:**
- ❌ No sample data endpoints in `store/apiEndpoints.ts`
- ✅ Other endpoints follow same pattern

**Impact:** Cannot call backend APIs for sample data operations

**Priority:** HIGH

**Files Affected:**
- `store/apiEndpoints.ts` (missing constants)

---

### 1.2 Missing Async Thunks

**TRD Requirement (Section 6.1):**
- Add async thunks in `store/asyncActions.ts`:
  - `getSampleDataStatus` (GET)
  - `importSampleData` (POST)
  - `removeSampleData` (POST)

**Current State:**
- ❌ No async thunks for sample data
- ✅ Uses `createApiAsyncThunk` pattern for other endpoints
- ✅ Uses prototype: `fetchAllSampleFiles()` + `uploadManager.addFiles()`

**Impact:** Cannot integrate with backend API endpoints

**Priority:** HIGH

**Files Affected:**
- `store/asyncActions.ts` (missing thunks)

---

### 1.3 Prototype vs Production Approach

**TRD Requirement (Section 1 & 2):**
- Files should be stored on backend S3, not shipped by FE
- Frontend should call `POST /sample-data/import` (empty body)
- Backend fetches from S3 and imports

**Current State:**
- ✅ Prototype: Files in `/public/sample-hands/`
- ✅ Prototype: Client-side fetch via `fetchAllSampleFiles()`
- ✅ Prototype: Upload via `uploadManager.addFiles()`
- ❌ No API call to `/sample-data/import`

**Impact:** 
- Works for prototype/testing
- Files shipped in frontend bundle (increases bundle size)
- Backend cannot manage sample data centrally
- Cannot support different datasets per user/game

**Priority:** MEDIUM (acceptable for prototype, must change for production)

**Files Using Prototype Approach:**
- `components/UploadHands/EmptyLanding.tsx` (lines 55-64)
- `components/UploadHands/UploadHandsLanding.tsx` (lines 454-461)
- `utils/sampleDataUtils.ts` (entire file)

---

## 2. State Management Gaps

### 2.1 Missing State Fields

**TRD Requirement (Section 6.2):**
```typescript
sampleDataSlice should have:
- installed: boolean
- installing: boolean
- isLoadingStatus: boolean
- lastError?: string
```

**Current State:**
```typescript
// Current implementation
export interface SampleDataState {
  sampleTransactionIds: string[];  // ✅ Present
  isImporting: boolean;              // ⚠️ Similar to installing, but different purpose
  isRemoving: boolean;               // ✅ Present (not in TRD, but needed)
  lastError: string | null;          // ✅ Present
  // ❌ Missing: installed: boolean
  // ❌ Missing: installing: boolean (separate from isImporting)
  // ❌ Missing: isLoadingStatus: boolean
}
```

**Gap Analysis:**
- `installed`: Currently derived from `sampleTransactionIds.length > 0` or checking `playerHandProcessing` for `[Sample]` prefix
- `installing`: Currently using `isImporting` (client-side flag), but TRD wants backend-provided `installing` flag
- `isLoadingStatus`: Missing - needed for `GET /sample-data/status` loading state

**Impact:**
- Status checking relies on client-side detection instead of authoritative backend status
- Cannot distinguish between "user is importing" vs "backend is importing"
- No loading state for status API calls

**Priority:** MEDIUM

**Files Affected:**
- `store/slices/sampleDataSlice.ts` (lines 10-26)

---

## 3. Status Checking Logic Gaps

### 3.1 No API-Based Status Check

**TRD Requirement (Section 4.2 & 5.1):**
- Call `GET /sample-data/status` on page load
- Use `installed` flag from API to show/hide CTA
- Use `installing` flag from API to show loading state

**Current State:**
- ✅ Checks `sampleTransactionIds` (client-side tracking)
- ✅ Checks `playerHandProcessing` for files with `[Sample]` prefix
- ✅ Uses `isImporting` flag (client-side)
- ❌ No API call to `/sample-data/status`
- ❌ No `installed`/`installing` flags from backend

**Impact:**
- Status detection is client-side only
- May miss sample data if transaction IDs aren't tracked
- Cannot detect backend-initiated imports
- Status may be stale after page refresh

**Priority:** MEDIUM

**Files Using Client-Side Detection:**
- `components/UploadHands/EmptyLanding.tsx` (lines 115-120)
- `components/SampleDataBanner/index.tsx` (lines 22-40)
- `components/UploadHands/UploadHandsLanding.tsx` (lines 424-425)

---

## 4. Import Flow Gaps

### 4.1 Missing API Call

**TRD Requirement (Section 4.2 & 5.2):**
- Call `POST /sample-data/import` (empty body)
- Backend fetches from S3 and imports
- Returns transaction IDs immediately
- WebSocket updates stream progress

**Current State:**
- ✅ Uses `fetchAllSampleFiles()` from `/public/sample-hands/`
- ✅ Uploads via `uploadManager.addFiles()`
- ✅ Shows progress in navbar (WebSocket works)
- ✅ Returns job IDs immediately
- ❌ No API call to `/sample-data/import`
- ❌ Files shipped in frontend bundle

**Impact:**
- Works for prototype but not production-ready
- Backend cannot manage sample data centrally
- Bundle size includes sample files

**Priority:** MEDIUM (prototype acceptable, production requires change)

**Files Affected:**
- `components/UploadHands/EmptyLanding.tsx` (lines 55-64)
- `components/UploadHands/UploadHandsLanding.tsx` (lines 454-461)

---

### 4.2 Error Handling Gaps

**TRD Requirement (Section 9):**
- Handle 409 "already installed" as success
- Handle 503/500 with retry option
- Default to showing CTA if status call fails (optimistic)

**Current State:**
- ✅ Basic error handling with toasts
- ✅ Generic error messages
- ❌ No 409 conflict handling (treat as success)
- ❌ No 503/500 specific handling
- ❌ No retry mechanism
- ❌ No optimistic fallback for status failures

**Impact:**
- User sees error even if sample data is already installed (409)
- No retry option for transient failures (503)
- Status failures may hide CTA unnecessarily

**Priority:** LOW (basic error handling works, but could be improved)

**Files Affected:**
- `components/UploadHands/EmptyLanding.tsx` (lines 89-108)
- `components/UploadHands/UploadHandsLanding.tsx` (lines 491-510)

---

## 5. Removal Flow Gaps

### 5.1 Missing Bulk Remove API

**TRD Requirement (Section 5.3):**
- Call `POST /sample-data/remove` (empty body)
- Backend removes all sample transactions atomically
- Refetch `getPlayerHandsProcessing` and status after removal

**Current State:**
- ✅ Uses `deleteHandsProcessed()` for each transaction ID
- ✅ Refetches `getPlayerHandsProcessing` after removal
- ✅ Shows confirmation modal and toast
- ✅ Handles partial failures
- ❌ No API call to `/sample-data/remove`
- ❌ Uses individual delete calls instead of bulk remove
- ❌ No refetch of status endpoint

**Impact:**
- Multiple API calls instead of single bulk operation
- Not atomic (some files may fail while others succeed)
- Status may be stale after removal

**Priority:** MEDIUM

**Files Affected:**
- `components/UploadHands/RemoveSampleDataButton.tsx` (lines 125-186)

---

## 6. Flow-by-Flow Compliance

### Flow 1: New User Empty State ✅ COMPLIANT

**PRD Requirement:**
- Two-panel layout on `/database` page
- Left: "START IMPROVING TODAY"
- Right: "EXPLORE FIRST [500]" with "Try Sample Hands" button
- Only shows when user has 0 hands

**Current Implementation:**
- ✅ Two-panel layout implemented
- ✅ Only shows on `/database` route
- ✅ Button shows/hides based on sample data presence
- ✅ Matches PRD design

**Status:** FULLY COMPLIANT

---

### Flow 2: Importing Sample Data ✅ COMPLIANT (Prototype)

**PRD Requirement:**
- Native upload experience
- Navbar upload indicator activates
- Files appear in navbar dropdown with progress
- Processing completes → files appear in table

**Current Implementation:**
- ✅ Uses native upload pipeline (`uploadManager.addFiles()`)
- ✅ Navbar indicator activates
- ✅ Progress bars in navbar dropdown
- ✅ WebSocket updates work
- ⚠️ Uses prototype approach (files from `/public/`)

**Status:** COMPLIANT (prototype approach, not production-ready)

---

### Flow 3: Sample Data Active ✅ COMPLIANT

**PRD Requirement:**
- Banner visible on Database, Sessions, Reports
- Shows hand count badge
- "Remove Samples" button
- Gradient background

**Current Implementation:**
- ✅ Banner component implemented
- ✅ Shows on all required pages
- ✅ Hand count badge
- ✅ Remove button with confirmation
- ✅ Gradient background matching design
- ✅ Hides while processing

**Status:** FULLY COMPLIANT

---

### Flow 4: Sample Data + Real Uploads ✅ COMPLIANT

**PRD Requirement:**
- Both coexist in all views
- Real uploads appear at top
- Reports aggregate both
- User can remove samples anytime

**Current Implementation:**
- ✅ Both appear in same table
- ✅ Real uploads sorted by UploadDate desc (most recent first)
- ✅ Sample files identified by `[Sample]` prefix
- ✅ Reports aggregate both (backend handles)
- ✅ Removal only affects sample files

**Status:** FULLY COMPLIANT

---

### Flow 5: Removing Sample Data ✅ COMPLIANT

**PRD Requirement:**
- Confirmation dialog
- Toast notification on success
- Only real uploads remain
- Banner disappears

**Current Implementation:**
- ✅ Confirmation modal with correct text
- ✅ Success toast with correct message
- ✅ Only real uploads remain
- ✅ Banner disappears
- ✅ Blocks removal while processing

**Status:** FULLY COMPLIANT

---

### Flow 6: Re-importing Sample Data ✅ COMPLIANT (Prototype)

**PRD Requirement:**
- Shows in Upload Panel when:
  - `totalHands > 0 && totalHands < 100`
  - `sampleDataPreviouslyRemoved === true`
  - Sample data not currently active
- "Add Sample Hands" button

**Current Implementation:**
- ✅ Section in upload panel
- ✅ Correct conditions checked
- ✅ Hand count calculated from table
- ✅ localStorage tracks "previously removed"
- ⚠️ Uses prototype approach (files from `/public/`)

**Status:** COMPLIANT (prototype approach, not production-ready)

---

## 7. Error Handling Gaps

### 7.1 Missing 409 Conflict Handling

**TRD Requirement (Section 9):**
- If import returns 409 "already installed":
  - Treat as success
  - Refresh status + processing list

**Current State:**
- ❌ No 409 handling
- Generic error shown even if already installed

**Priority:** LOW

---

### 7.2 Missing 503/500 Specific Handling

**TRD Requirement (Section 9):**
- If backend returns 503/500:
  - Show error
  - Allow retry

**Current State:**
- ✅ Generic error shown
- ❌ No retry mechanism
- ❌ No distinction between error types

**Priority:** LOW

---

### 7.3 Missing Optimistic Fallback

**TRD Requirement (Section 9):**
- If status endpoint fails:
  - Default to showing CTA (optimistic onboarding)
  - OR hide CTA (safe)

**Current State:**
- ⚠️ Uses client-side detection (doesn't fail, but not authoritative)
- No fallback logic needed (no API call)

**Priority:** LOW (becomes relevant when API integration is added)

---

## 8. Persistence Behavior ✅ COMPLIANT

**PRD Requirement:**
- Sample data persists across:
  - Page navigation ✅
  - Browser refresh ✅
  - Browser close/reopen ✅
  - Logout/login ✅

**Current State:**
- ✅ Data stored in backend (persists naturally)
- ✅ Uses localStorage for "previously removed" flag
- ⚠️ Relies on filename prefix for identification (not backend status)

**Status:** COMPLIANT (works, but identification method could be improved)

---

## 9. File Storage ⚠️ PROTOTYPE ONLY

**PRD/TRD Requirement:**
- Production: Files in backend S3, not shipped by FE
- Prototype: Files in `/public/sample-hands/` (acceptable for now)

**Current State:**
- ✅ Prototype approach implemented
- ✅ Files in `/public/sample-hands/`
- ⚠️ Not production-ready (files in bundle)

**Status:** PROTOTYPE COMPLIANT (production requires backend API)

---

## 10. Documentation Review

### 10.1 Implementation Documentation ✅

**Files:**
- `docs/SAMPLE_DATA_IMPLEMENTATION.md` - Accurate
- `docs/SAMPLE_DATA_FLOWS_IMPLEMENTATION.md` - Accurate
- `docs/BACKEND_API_CONTRACT_SAMPLE_DATA.md` - Accurate (for future API integration)
- `docs/SAMPLE_DATA_I18N_KEYS.md` - Complete

**Status:** DOCUMENTATION COMPLETE

---

## Summary of Gaps

### Critical Gaps (Must Fix for Production)

1. **Missing API Endpoints** (HIGH)
   - No endpoint constants
   - No async thunks
   - Cannot integrate with backend

2. **Missing State Fields** (MEDIUM)
   - Missing `installed`, `installing`, `isLoadingStatus`
   - Status detection is client-side only

3. **Prototype Approach** (MEDIUM)
   - Files shipped in bundle
   - Client-side file fetching
   - Must migrate to API-based for production

### Non-Critical Gaps (Nice to Have)

4. **Error Handling** (LOW)
   - Missing 409 conflict handling
   - Missing 503/500 specific handling
   - No retry mechanism

5. **Bulk Remove** (MEDIUM)
   - Uses individual delete calls
   - Not atomic operation
   - Should use `/sample-data/remove` API

---

## Recommendations

### Priority 1: API Integration (Production Migration)

**Action Items:**
1. Add endpoint constants to `store/apiEndpoints.ts`
2. Add async thunks to `store/asyncActions.ts`
3. Update `sampleDataSlice` to include `installed`, `installing`, `isLoadingStatus`
4. Replace prototype calls with API calls:
   - `EmptyLanding.tsx`: Call `importSampleData()` instead of `fetchAllSampleFiles()`
   - `UploadHandsLanding.tsx`: Same replacement
   - `RemoveSampleDataButton.tsx`: Call `removeSampleData()` instead of individual deletes
   - `EmptyLanding.tsx` & `SampleDataBanner.tsx`: Call `getSampleDataStatus()` on mount

**Estimated Effort:** 4-6 hours

---

### Priority 2: State Management Updates

**Action Items:**
1. Add `installed: boolean` to `SampleDataState`
2. Add `installing: boolean` (separate from `isImporting`)
3. Add `isLoadingStatus: boolean`
4. Update reducers to handle API responses
5. Update components to use new state fields

**Estimated Effort:** 2-3 hours

---

### Priority 3: Error Handling Improvements

**Action Items:**
1. Handle 409 conflict in import flow (treat as success)
2. Add retry mechanism for 503/500 errors
3. Add optimistic fallback for status failures
4. Improve error messages with specific guidance

**Estimated Effort:** 2-3 hours

---

### Priority 4: Bulk Remove API

**Action Items:**
1. Replace individual `deleteHandsProcessed()` calls with `removeSampleData()` API
2. Update error handling for atomic operation
3. Refetch status after removal

**Estimated Effort:** 1-2 hours

---

## Migration Path

### Phase 1: Keep Prototype, Add API Infrastructure
- Add API endpoints and thunks
- Add missing state fields
- Keep prototype approach working
- **Result:** Code ready for API integration, prototype still works

### Phase 2: Switch to API-Based (When Backend Ready)
- Replace `fetchAllSampleFiles()` calls with `importSampleData()` API
- Replace individual deletes with `removeSampleData()` API
- Add `getSampleDataStatus()` calls
- Remove `utils/sampleDataUtils.ts` file fetching logic
- **Result:** Production-ready implementation

### Phase 3: Error Handling & Polish
- Add 409/503/500 handling
- Add retry mechanisms
- Add optimistic fallbacks
- **Result:** Robust, production-grade implementation

---

## Compliance Matrix

| Requirement | PRD | TRD | Current Status | Gap |
|------------|-----|-----|----------------|-----|
| Flow 1: Empty State | ✅ | ✅ | ✅ Implemented | None |
| Flow 2: Import | ✅ | ✅ | ✅ Prototype | API integration |
| Flow 3: Banner | ✅ | ✅ | ✅ Implemented | None |
| Flow 4: Coexistence | ✅ | ✅ | ✅ Implemented | None |
| Flow 5: Removal | ✅ | ✅ | ✅ Implemented | Bulk API |
| Flow 6: Re-import | ✅ | ✅ | ✅ Prototype | API integration |
| API Endpoints | N/A | ✅ | ❌ Missing | Critical |
| State Management | N/A | ✅ | ⚠️ Partial | Missing fields |
| Error Handling | N/A | ✅ | ⚠️ Basic | Missing scenarios |
| Status Checking | N/A | ✅ | ⚠️ Client-side | API-based needed |

---

## Conclusion

The current implementation successfully delivers **all PRD user flows** using a prototype approach. The UI/UX is complete and matches the PRD specifications. However, to align with the TRD's production architecture, the following must be addressed:

1. **API Integration** - Add endpoint constants and async thunks
2. **State Management** - Add missing state fields (`installed`, `installing`, `isLoadingStatus`)
3. **Migration** - Replace prototype file fetching with API calls
4. **Error Handling** - Add specific error scenario handling

The prototype approach is acceptable for testing and development, but production deployment requires migration to the API-based approach as specified in the TRD.

**Overall Assessment:**
- ✅ **PRD Compliance:** 100%
- ⚠️ **TRD Compliance:** 60% (UI complete, API integration missing)
- ⚠️ **Production Readiness:** Prototype only

**Recommendation:** Proceed with Phase 1 (API infrastructure) while keeping prototype functional, then migrate to Phase 2 when backend APIs are ready.

