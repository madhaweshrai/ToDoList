# PLAN — ToDoList Modernisation (G01 Structured Todo Model)

## Objective
Modernise the ToDoList app’s persistence layer by replacing the legacy **array-of-strings** `localStorage` format with a **structured todo object model** with **stable IDs and metadata**, enabling reliable operations (delete/complete) even with duplicate titles and providing a foundation for future features (edit, sync, priorities, etc.).

Primary business outcome: **Correctness and extensibility** of todo operations through a stable data model and backward-compatible migration.

---

## Scope

### In scope (this release)
- **G01**: Store todos in `localStorage` as structured objects with:
  - `id` (stable unique string)
  - `title` (string)
  - `completed` (boolean)
  - `createdAt` (ISO timestamp string)
  - `updatedAt` (ISO timestamp string)
- Backward-compatible **migration** on app load:
  - If legacy storage contains `string[]`, migrate to `Todo[]` and persist back.
- Update UI behaviors to rely on `todo.id` rather than `innerText` or array index:
  - Add todo creates object and persists
  - Toggle complete updates correct object and persists
  - Delete removes by `id` and persists
- Add/extend tests and/or a lightweight test plan to prove:
  - Correct migration
  - No duplicate-title deletion bug
  - No regression of add/complete/delete
- Documentation updates covering:
  - Storage keys
  - Schema
  - Migration behavior and troubleshooting reset steps

### Out of scope (explicitly not in this release)
- Backend/API, database, authentication, multi-user sync
- Edit functionality, priorities, due dates, search/filter
- UI/UX redesign, accessibility overhaul, theme refactor
- CI/CD pipeline setup or containerisation

---

## Target architecture (post-change)

### Frontend modules (logical)
- **storage layer** (new/updated): responsible for read/write/migrate
- **domain model**: `Todo` type + helper functions
- **UI binding**: render list items with `data-id="<todo.id>"` and drive actions via IDs

### Data model
```ts
type Todo = {
  id: string;          // stable unique identifier
  title: string;       // trimmed user-entered text
  completed: boolean;  // completion status
  createdAt: string;   // ISO datetime
  updatedAt: string;   // ISO datetime
};
```

### Persistence contract
- `localStorage["todos"]` (or existing key if already present; document final choice) stores:
  - JSON stringified `Todo[]`
- Migration:
  - Detect `string[]` and convert each string to a `Todo` object.
  - Preserve display order as currently shown to users.
  - Re-write storage as `Todo[]` once migrated.

---

## Milestones (M1–M4)

> Estimates are in **story points** for milestones, and **timebox guidance** for subtasks where points are not defined in Jira. Owners are placeholders unless Jira assignees are set.

| Milestone | Goal | Jira key(s) | Estimate | Owner |
|---|---|---:|---|---|
| **M1** | Confirm storage key, schema, and migration approach; agree acceptance criteria & edge cases | **EPMCDMETST-68567** | **5 SP** | Delivery Lead + Dev |
| **M2** | Implement structured model + ID-based DOM binding + migration | **EPMCDMETST-68568** | ~1–2 dev days | Dev |
| **M3** | Verification: tests (or test plan) for migration, persistence, duplicates; regression checks | **EPMCDMETST-68569** | ~0.5–1 dev day | QA/Dev |
| **M4** | Documentation: schema, localStorage keys, migration behavior, troubleshooting/reset steps | **EPMCDMETST-68570** | ~0.25–0.5 dev day | Dev |

---

## Implementation plan (by Jira issue)

### EPMCDMETST-68567 — Story: G01 structured todo storage
**Key deliverables**
- Finalise the `Todo` schema and storage key naming.
- Define migration rules for legacy `string[]`:
  - What constitutes “legacy” (e.g., parsed JSON is array of strings).
  - Behavior on corrupted JSON.
- Define acceptance test checklist.

**Design notes**
- Prefer an ID generation function with extremely low collision risk:
  - Option A: `crypto.randomUUID()` when available; fallback to timestamp+random.
- Ensure title is `trim()`med; decide whether to reject empty strings (current behavior uses alert—out of scope to redesign, but do not store empty items).

---

### EPMCDMETST-68568 — Sub-task: Implement object model + migration
**Steps**
1. Introduce `Todo` model and helper functions:
   - `makeTodo(title): Todo`
   - `serializeTodos(todos): string`
   - `parseTodos(raw): Todo[] | legacy string[] | []`
2. Implement storage functions:
   - `loadTodos(): Todo[]` (includes migration)
   - `saveTodos(todos: Todo[]): void`
3. Update render logic:
   - When creating `<li>`, attach `li.dataset.id = todo.id`
   - Ensure completed state uses `todo.completed` to set CSS class.
4. Update event handlers:
   - Toggle complete:
     - Find todo by `id`
     - Flip `completed`
     - Set `updatedAt`
     - Persist + update UI
   - Delete:
     - Remove by `id` (filter by id)
     - Persist + remove DOM node
5. Migration:
   - If legacy `string[]` detected:
     - Convert each string entry to `Todo`
     - Persist back to new format immediately
     - Render converted list

**Non-functional requirements**
- Do not break existing theme selection and UI layout.
- Maintain ordering of existing todos as loaded from legacy storage.

---

### EPMCDMETST-68569 — Sub-task: Test G01 (persistence, migration, duplicates)
**Approach**
- If no test framework exists, produce a lightweight test plan + manual checklist in repo docs (or as comments) and optionally add a minimal test harness if feasible.
- Recommended minimum verification:
  1. **Create** todo persists as object with required fields and non-empty `id`.
  2. **Migration** rewrites legacy `["A","B"]` to objects and still displays A/B.
  3. **Duplicate titles**: delete one of two same-title entries deletes only one.
  4. **Toggle complete** changes the correct item and persists across reload.

**Artifacts**
- If unit tests are added:
  - Target pure functions: ID generation, migration, serialization/deserialization.
- Manual regression checklist stored in documentation (ties into EPMCDMETST-68570).

---

### EPMCDMETST-68570 — Sub-task: Documentation
**Content to document**
- The final localStorage key(s) used (e.g., `todos`)
- The `Todo` schema and meanings for each field
- Migration behavior:
  - Trigger conditions
  - What happens on malformed storage
- Troubleshooting:
  - How to clear todos (remove localStorage key in devtools)
  - Expected symptoms if storage is corrupted

---

## Dependencies
- Browser environment support:
  - `localStorage` availability (assumed)
  - `crypto.randomUUID()` availability (implement fallback)
- No external services required.
- Team decision on final storage key naming (M1).

---

## Risks and mitigations

| Risk | Impact | Likelihood | Mitigation |
|---|---|---:|---|
| Migration corrupts or loses user data | High | Medium | Implement safe parsing with try/catch; on failure, keep a backup key (e.g., `todos_legacy_backup`) before overwrite; verify migration with test cases |
| ID collisions leading to wrong deletes/toggles | High | Low | Use `crypto.randomUUID()` when available; fallback includes randomness + timestamp; add guard to regenerate if duplicate in loaded set |
| Regression in add/complete/delete flows | Medium | Medium | Maintain parity with current UX; run regression checklist; test duplicates scenario explicitly |
| Malformed `localStorage` content breaks app load | Medium | Medium | Defensive parsing: if JSON invalid, treat as empty list and log to console; avoid throwing in load path |

---

## Test strategy

### Automated (preferred where feasible)
- Unit tests for pure functions:
  - `migrateLegacyTodos(string[]) => Todo[]`
  - `loadTodos` parsing behavior (mocked localStorage)
  - ID generator uniqueness (basic sanity)
- Minimal DOM behavior tests if environment allows; otherwise manual verification.

### Manual regression checklist (must pass)
- Add todo:
  - Adds to UI
  - Persists as object with id + timestamps
- Reload page:
  - Todos load correctly with completed state
- Toggle complete:
  - Correct item toggles; persists after reload
- Delete:
  - Deletes correct item by id
  - Duplicate titles: only one removed

---

## Build and deployment

### Build
- Static site: open `index.html` directly or serve via a local static server.
- No bundler assumed for this scope; keep changes compatible with current structure.

### Deployment
- No deployment changes in scope.
- Ensure changes work when hosted as static files (e.g., GitHub Pages style hosting).

---

## HITL checkpoints (Human-in-the-loop)
1. **After M1 (design checkpoint)**: confirm schema fields, storage key, and migration rules (especially malformed JSON handling and backup strategy).
2. **After M2 (implementation checkpoint)**: reviewer verifies duplicate-title delete fix and confirms DOM uses `data-id`.
3. **After M3 (quality checkpoint)**: reviewer signs off on test evidence (automated results or completed manual checklist).
4. **Before merge (release checkpoint)**: reviewer confirms docs updated and no regressions observed.

---

## Definition of Done (DoD)
- All Jira issues in scope are complete:
  - EPMCDMETST-68568, EPMCDMETST-68569, EPMCDMETST-68570 resolved and linked to EPMCDMETST-68567
- Todos persist as structured objects with stable IDs and required metadata.
- Migration from legacy `string[]` happens automatically on load without user action.
- Duplicate-title deletion removes only the selected todo.
- Completed state is correctly persisted and reflected on reload.
- Documentation is present and accurate for schema and migration.
- Code follows branching and commit conventions below.

---

## Branching strategy
- Branch name format:
  - `feature/<JIRA-KEY>-<slug>`
- Examples:
  - `feature/EPMCDMETST-68568-structured-todo-model`
  - `feature/EPMCDMETST-68569-g01-tests`
  - `feature/EPMCDMETST-68570-g01-docs`

---

## Commit message convention
- Format: `<JIRA-KEY>: <message>`
- Examples:
  - `EPMCDMETST-68568: store todos as objects with stable ids`
  - `EPMCDMETST-68568: migrate legacy string array localStorage format`
  - `EPMCDMETST-68569: add test plan for migration and duplicates`
  - `EPMCDMETST-68570: document localStorage schema and migration behavior`
