# Responsive audit — blocks, base-blocks, registry components

Scope: `packages/ui/src/blocks/**`, `packages/ui/src/base-blocks/**`,
`registry/bases/{base,radix}/components/**`. Core primitives
(`packages/ui/src/components`, `packages/ui/src/base-components`) are out of
scope and were not touched or flagged — all fixes below live in block
compositions and registry pattern files only.

`blocks/` and `base-blocks/` are structural mirrors (Radix vs. Base UI
render API, identical layout code), so findings apply to both engines unless
noted otherwise.

## Findings

### P1 — Kanban board columns override the block's responsive default
- `registry/bases/{base,radix}/components/kanban/c-kanban-{1,2,3,4,5,6}.tsx`
  (12 files: 6 patterns × 2 engines)
- The `KanbanBoard` primitive (`packages/ui/src/blocks/kanban.tsx:625`) already
  ships the right default: `grid auto-rows-fr gap-4 sm:grid-cols-3` (1 column
  until `sm`). Every one of these 12 pattern files passes its own
  `className="grid auto-rows-fr grid-cols-3"`, dropping the `sm:` prefix and
  forcing 3 columns at every viewport — columns crush to ~100px on a 375px
  phone.
- Fix: change each override to `sm:grid-cols-3`, or delete the override and
  inherit the block default.

### P1 — Table patterns have no horizontal-scroll wrapper
- `registry/bases/{base,radix}/components/table/c-table-{1,2,3,4,5,6,7,8,9,10,11,13,15}.tsx`
  (12 of 17 files per engine)
- None of these wrap `<Table>` in an `overflow-x-auto` container (grep
  confirms zero "overflow" hits per file), and the `Table` primitive itself
  has no built-in scroll handling. Any table wide enough to exceed viewport
  (most of these: id/status/method/amount/actions columns) overflows the
  whole page horizontally on mobile with no way to scroll it into view.
  `c-table-14/16/17.tsx` already do this correctly — use them as the
  reference pattern.
- Fix: wrap `<Table>...</Table>` in `<div className="overflow-x-auto">` in
  each of the 12×2 files.
- **Needs a closer look, not yet confirmed:** `data-grid/c-data-grid-*.tsx`
  (29 files, also 0 "overflow" hits) may share this gap, or may already be
  covered by `DataGridTable`'s internal virtualized/pinned-column scroll
  container (`packages/ui/src/blocks/data-grid/data-grid-table.tsx`) — verify
  before treating as a defect.

### P1 — Kanban column drag handle is invisible/undiscoverable on touch
- `packages/ui/src/blocks/kanban.tsx:738-750`, `KanbanColumnHandle`
- The handle is `opacity-0` by default and only reveals on
  `group-hover/kanban-column:opacity-100`. Touch devices have no hover state,
  so column reordering has no visible affordance and is effectively unusable
  on mobile/tablet.
- Fix: add a `pointer-coarse:opacity-100` (or focus-visible) fallback so the
  handle is always visible/reachable on touch and keyboard.

### P2 — File-upload image grids hardcode 4–5 columns with no breakpoint
- `registry/bases/{base,radix}/components/file-upload/c-file-upload-7.tsx:232,267`
  (`grid-cols-4`, two grids), `c-file-upload-8.tsx:312` (`grid-cols-5`)
- 120px-tall image cards squeeze to ~70–85px at 4–5 unresponsive columns on a
  375px viewport. Sibling patterns in the same category already show the
  correct fix: `c-file-upload-4.tsx:185`
  (`grid-cols-2 sm:grid-cols-3 md:grid-cols-4`) and `c-file-upload-9.tsx:375`
  (`grid-cols-1 sm:grid-cols-4 lg:grid-cols-6`).
- Fix: apply the same responsive ramp to `c-file-upload-7`/`8`.

### P2 — Small touch targets on rating stars and stepper indicators
- `packages/ui/src/blocks/rating.tsx:117-134` — each star is a bare `<div>`
  sized only by the icon (well under 44px), no touch-target expansion.
- `packages/ui/src/blocks/stepper.tsx:331` — `StepperTrigger` indicator is a
  `size-6` (24px) `<button>`, no touch-target compensation.
- The codebase already has the right fix pattern in-repo:
  `number-field.tsx:35` uses `pointer-coarse:after:min-h-11
  pointer-coarse:after:min-w-11` (an invisible expanded hit-area via a
  pseudo-element) to hit the 44×44 WCAG target without changing visual size.
  This is a one-pattern, two-file rollout, not two new designs.

## Not real issues (checked, ruled out)

- Fixed-width hits in `select`/`popover`/`hover-card`/`sonner`/`context-menu`/
  `navigation-menu`/`filters`/`data-grid-column-visibility`/
  `data-grid-column-filter` are all floating/portaled overlay content sized
  to their own content; Radix/Base UI collision detection clamps them to the
  viewport. Standard shadcn practice, not a defect.
- `gantt/c-gantt-1.tsx:456` and `calendar/c-calendar-{13,26}.tsx` grid
  overrides are inside fixed-size popovers/widgets, not page layout.
- `date-selector.tsx:986` `grid-cols-2` year list is nested inside an
  already-responsive `w-full sm:w-[470px]` container — scales correctly.
- `data-grid`, `gantt`, `event-calendar` blocks already wire explicit
  `overflow-x-auto` scroll containers for wide tabular/timeline content.
- `data-grid-pagination.tsx` and `filters.tsx` already stack correctly on
  mobile via `sm:flex-row`.

## Systemic patterns

1. **Block defaults are already responsive; pattern files sometimes override
   them into a broken state.** Both the kanban-grid and table-scroll issues
   above are cases where the reusable block got it right and a specific
   registry pattern silently regressed it. Before shipping a new pattern
   file, diff its className against the block's own default.
2. **One correct touch-target pattern exists (`number-field`'s
   `pointer-coarse:after:*` technique) but wasn't propagated** to kanban's
   drag handle, rating's stars, or stepper's trigger. Worth turning into a
   documented convention (e.g. a short note in `CONTRIBUTING.md`) so new
   small interactive controls use it by default.

## Improvement plan (priority order)

1. **P1 — Kanban grid fix** (12 files): `grid-cols-3` → `sm:grid-cols-3` in
   `registry/bases/{base,radix}/components/kanban/c-kanban-{1..6}.tsx`.
2. **P1 — Table scroll wrapper** (24 files): wrap `<Table>` in
   `<div className="overflow-x-auto">` in
   `registry/bases/{base,radix}/components/table/c-table-{1,2,3,4,5,6,7,8,9,10,11,13,15}.tsx`.
   Verify `data-grid/c-data-grid-*.tsx` separately before deciding if it
   needs the same treatment.
3. **P1 — Kanban drag handle touch visibility** (1 shared source): add
   `pointer-coarse:opacity-100` fallback in
   `packages/ui/src/blocks/kanban.tsx` `KanbanColumnHandle` (fixes both
   engines at once since blocks/base-blocks mirror the same composition).
4. **P2 — File-upload grid responsiveness** (2 files): apply the existing
   `c-file-upload-4`/`9` responsive ramp to `c-file-upload-7`/`8`.
5. **P2 — Touch targets on rating/stepper** (2 shared sources): apply
   `number-field`'s `pointer-coarse:after:min-h-11 min-w-11` pattern to
   `rating.tsx`'s star hit areas and `stepper.tsx`'s `StepperTrigger`.
6. **Verification after each step**: rebuild the registry pipeline
   (`build:components && build:packages && build:registry &&
   verify:registry`), rebuild `apps/uikit`, then screenshot the affected
   showcase routes at 375px/768px/1280px to confirm the fix and that no
   sibling pattern regressed.

Every fix above is scoped to block compositions and registry pattern files —
none require touching `packages/ui/src/components` or
`packages/ui/src/base-components`.
