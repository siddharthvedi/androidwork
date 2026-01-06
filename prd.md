# PRD: Sample Data Feature

## Document Info

| Field              | Value        |
| ------------------ | ------------ |
| **Status**         | Draft        |
| **Author**         | Product Team |
| **Created**        | 2026-01-05   |
| **Last Updated**   | 2026-01-05   |
| **Target Release** | Q1 2026      |

---

## Executive Summary

Enable new users to explore FlopHero with pre-analyzed sample hands before uploading their own. Sample data provides a risk-free way to experience Sessions, Reports, Database, and Hand Replayer features while learning the platform's upload flow.

---

## Problem Statement

New users face a cold-start problem:

1. They must upload hands before seeing any value
2. Upload process is unfamiliar and may feel risky
3. No way to evaluate the product before committing time/data

**Impact**: Lower activation rates, higher bounce rates on empty states.

---

## Solution Overview

Provide a "Try Sample Hands" option that:

1. Imports 500 pre-analyzed hands using the **native upload flow** (Navbar progress, etc.)
2. Populates Sessions, Reports, and Database with real data
3. Coexists with user's real uploads
4. **PERSISTS until explicitly removed** - survives page navigation, refresh, browser close
5. Can be removed at any time without affecting real uploads
6. Can be re-imported via Upload Panel if user has < 100 hands

---

## Persistence Behavior (Critical)

Sample data is **persistent by default**. It does NOT disappear unless the user explicitly removes it.

| Scenario                                     | Sample Data Status         |
| -------------------------------------------- | -------------------------- |
| User navigates Sessions → Reports → Database | ✅ Stays active            |
| User refreshes browser (F5)                  | ✅ Stays active            |
| User closes browser, reopens next day        | ✅ Stays active            |
| User logs out and logs back in               | ✅ Stays active            |
| User clicks "Remove Samples" button          | ❌ Removed                 |
| User uploads real hands                      | ✅ Stays active (coexists) |


---

## User Flows

### Flow 1: New User Empty State

**Trigger**: User navigates to `/[game]/database` with 0 hands

**UI**: Left-to-right two-column layout

```
┌─────────────────────────────────┐     ┌─────────────────────────────────┐
│     START IMPROVING TODAY       │     │       EXPLORE FIRST [500]       │
│                                 │     │                                 │
│  Upload your hand histories     │ or  │  Not ready to upload? Try       │
│  and get instant GTO feedback   │     │  FlopHero with real PLO hands   │
│  on every decision.             │     │  analyzed by our solver.        │
│                                 │     │                                 │
│  [Upload] [Live Tracker]        │     │  • Review 500 real hands        │
│                                 │     │  • See your session stats       │
│                                 │     │  • Replay analyzed spots        │
│                                 │     │                                 │
│                                 │     │  [Try Sample Hands]             │
└─────────────────────────────────┘     └─────────────────────────────────┘
```
---

### Flow 2: Importing Sample Data (Native Upload Experience)

**Trigger**: User clicks "Try Sample Hands"

**Experience**: Identical to real file upload

1. Button shows loading state
2. Navbar upload indicator activates (same as real uploads)
3. Files appear in Navbar dropdown with progress bars
4. Processing completes → files appear in Database table

```
┌──────────────────────────────────────────────────────────────────────────┐
│  FlopHero   [Sessions]  [Reports]  [Database]               [📤 3 files]│
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                 NAVBAR DROPDOWN (same as real uploads)                   │
│                 ┌────────────────────────────────────────┐               │
│                 │ [Sample] PLO 6-Max Cash    ████████░░  │               │
│                 │ [Sample] PLO 3-Bet Pots    ██████░░░░  │               │
│                 │ [Sample] PLO Multiway      ████░░░░░░  │               │
│                 │                                        │               │
│                 │ Importing sample hands...              │               │
│                 └────────────────────────────────────────┘               │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Why Native Experience?**

- Teaches user the real upload flow
- Builds familiarity with Navbar progress indicator
- Same mental model for sample and real uploads

---

### Flow 3: Sample Data Active (Database View)

**Trigger**: Sample data import complete

**UI**: Banner + sample files in table

```
┌──────────────────────────────────────────────────────────────────────────┐
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  FH  Sample Data Active   [500 hands]            [Remove Samples]  │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  File Name                      │ Site   │ Hands │ Status         │  │
│  ├────────────────────────────────────────────────────────────────────┤  │
│  │  [Sample] PLO 6-Max Cash        │ Sample │  250  │ ✓ Complete     │  │
│  │  [Sample] PLO 3-Bet Pots        │ Sample │  150  │ ✓ Complete     │  │
│  │  [Sample] PLO Multiway          │ Sample │  100  │ ✓ Complete     │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

**Banner Requirements**:

- Visible on Database, Sessions, and Reports pages
- Shows hand count badge
- "Remove Samples" button (confirmation required)
- Gradient background matching FlopHero style

---

### Flow 4: Sample Data + Real Uploads (Coexistence)

**Trigger**: User uploads real hands while sample data is active

**Behavior**: Both coexist in all views

```
┌────────────────────────────────────────────────────────────────────────┐
│  File Name                      │ Site        │ Hands │ Status        │
├────────────────────────────────────────────────────────────────────────┤
│  my_session_jan_05.txt          │ PokerStars  │  342  │ ✓ Complete    │  ← Real
│  [Sample] PLO 6-Max Cash        │ Sample      │  250  │ ✓ Complete    │  ← Sample
│  [Sample] PLO 3-Bet Pots        │ Sample      │  150  │ ✓ Complete    │  ← Sample
│  [Sample] PLO Multiway          │ Sample      │  100  │ ✓ Complete    │  ← Sample
└────────────────────────────────────────────────────────────────────────┘
```

**Key Behaviors**:

- Sample banner remains visible
- Real uploads appear at top (most recent)
- Reports aggregate both (clearly labeled)
- User can remove samples anytime

---

### Flow 5: Removing Sample Data

**Trigger**: User clicks "Remove Samples"

**Confirmation Dialog**:

```
┌─────────────────────────────────────────────────────────┐
│  Remove Sample Data?                                    │
│                                                         │
│  This will remove the 500 sample hands from your        │
│  database. Your uploaded hands will not be affected.    │
│                                                         │
│  You can re-import sample data from the Upload panel    │
│  if you have fewer than 100 hands.                      │
│                                                         │
│                    [Cancel]  [Remove Samples]           │
└─────────────────────────────────────────────────────────┘
```

**After Removal**:

- Only real uploads remain
- Banner disappears
- Sample files removed from Sessions/Reports/Database

**Toast Notification (Required)**:

Upon successful removal, display a toast notification using the existing toast/message manager:

```typescript
// Use Ant Design message or existing toast system
message.success({
  content: t(
    'sampleData.removedSuccess',
    'Sample data has been removed. Your uploaded hands remain unchanged.'
  ),
  duration: 4,
  key: 'sample-data-removed',
});
```

| Property       | Value                                                                 |
| -------------- | --------------------------------------------------------------------- |
| **Type**       | Success toast (green checkmark)                                       |
| **Message**    | "Sample data has been removed. Your uploaded hands remain unchanged." |
| **i18n Key**   | `sampleData.removedSuccess`                                           |
| **Duration**   | 4 seconds                                                             |
| **Unique Key** | `sample-data-removed` (prevents duplicate toasts)                     |

**Implementation Note**: Use the existing `message` API from Ant Design that is already configured in the project. The toast should appear at the top-center of the viewport, consistent with other notifications in the app.

---

### Flow 6: Re-importing Sample Data (Upload Panel)

**Condition**: User has < 100 total hands AND previously removed sample data

**Location**: Upload Panel (drawer)

```
┌─────────────────────────────────────────────────────────┐
│  UPLOAD PANEL                                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   Drop files here or click to browse                    │
│                                                         │
│   ─────────────────────────────────────────────────     │
│                                                         │
│   ┌─────────────────────────────────────────────────┐   │
│   │  FH  Want more hands to explore?                │   │
│   │                                                 │   │
│   │  You have 42 hands. Add 500 sample hands        │   │
│   │  to explore more features.                      │   │
│   │                                                 │   │
│   │                      [Add Sample Hands]         │   │
│   └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Conditions**:

- `totalHands < 100`
- `sampleDataPreviouslyRemoved === true`
- NOT shown if sample data is currently active

---

## Sample Data Specification

### Data Structure

Each sample file follows the existing `PlayerHandProcessingResponse` type:

```typescript
interface SampleFile {
  ID: string; // Format: 'sample-{game}-{variant}-{index}'
  FileName: string; // Format: '[Sample] {Description}'
  Site: 'Sample'; // Identifies as sample data
  Processing: 4; // Always complete
  HandCount: number;
  TotalHandCount: number;
  AnalyzedHandCount: number;
  // ... all other PlayerHandProcessingResponse fields
}
```

### Sample Data by Game

| Game            | Variant    | Hands | Files | Description                               |
| --------------- | ---------- | ----- | ----- | ----------------------------------------- |
| PLO (`omaha`)   | 6-Max Cash | 500   | 3     | Diverse spots: 3-bet pots, multiway, etc. |
| NLHE (`holdem`) | 6-Max Cash | 500   | 3     | Coming later                              |
| Spins (`spins`) | 3-Max      | 500   | 3     | Coming later                              |

### File Naming Convention

```
[Sample] {Game} {Spot Type}
```

Examples:

- `[Sample] PLO 6-Max Cash Session`
- `[Sample] PLO 3-Bet Pots Focus`
- `[Sample] PLO Multiway Pots`

---

## Data Management

### Who Manages Sample Data?

**Product Team** uploads and maintains sample data via:

1. Cursor IDE (current workflow)
2. Future: Admin panel (Phase 2)

### Storage & Distribution

| Aspect   | Current (Prototype)                         | Production                  |
| -------- | ------------------------------------------- | --------------------------- |
| Storage  | Raw `.txt` files in `/public/sample-hands/` | Backend database            |
| Upload   | Real upload via `uploadManager.addFiles()`  | API call to clone dataset   |
| Per-user | Uploaded to user's account (real data)      | Cloned to user's account    |
| Updates  | Replace files in `/public/sample-hands/`    | Backend update, no redeploy |

**Prototype Approach:**

- Sample hand history files stored in `/public/sample-hands/`
- When user clicks "Try Sample Hands", files are fetched and uploaded via the REAL upload pipeline
- Backend processes them like any other upload
- User sees REAL data in Sessions, Reports, and Hand Replayer

### Sample Data Lifecycle

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Product Team   │────▶│  Sample Data    │────▶│  User Account   │
│  uploads hands  │     │  Repository     │     │  (cloned copy)  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                               │
                               ▼
                        Shared by all users
                        (read-only source)
```

---

## Scalability: A/B Testing & Segmentation

### Future Architecture

Support different sample datasets by:

| Dimension   | Example Values                      | Use Case                 |
| ----------- | ----------------------------------- | ------------------------ |
| Game        | `omaha`, `holdem`, `spins`          | Game-specific samples    |
| Client Type | `free`, `pro`, `enterprise`         | Feature showcase by tier |
| Platform    | `web`, `mobile`, `desktop`          | Platform-optimized hands |
| Experiment  | `control`, `variant_a`, `variant_b` | A/B testing activation   |
| Region      | `us`, `eu`, `latam`                 | Localized content        |
