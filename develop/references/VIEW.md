# VIEW.md — Views & Routing

A view is a routed page component. It should stay thin: composing components, wiring its store, and handling page-level concerns (SEO meta, route guards, initial data fetch) — not business logic or deep markup.

## Naming

Views live in the `views/` folder and are named `<Subject>View.vue`:

```text
HomeView.vue
FacilitiesView.vue
PropertyCityView.vue
```

The component's `name` (and the file name) always matches what the route config calls it.

## One Store Per View — only when the state is actually shared

A store exists to hold state that needs to be shared between views, components, composables, or
other stores (see `STORE.md`'s opening definition). A view that fetches nontrivial state gets its
own dedicated store **when that state (or the fetching/mutating logic around it) is genuinely
needed by more than just the view's own local `setup()`** — i.e. it's read/written by several of
the view's own child components (so a store is how they share it without prop-drilling everything
several layers deep), or another view/store will plausibly need it too:

```text
PropertyCityView.vue      → stores/property-city.store.js
FlightSearchView.vue      → stores/flight-search.store.js
```

(Stores stay flat under `stores/`, no per-domain subfolder — see `STORE.md`.)

**If a view's fetched state is consumed only inside that one view's own `setup()`, with nothing
passed any deeper than direct child components it renders, don't create a store for it at all** —
keep the refs/fetch function local to the view and pass data down via props. A store whose only
consumer, ever, is the one view that defines it isn't "shared" — it's just indirection. Only
promote it to a store the moment a second, genuinely separate consumer (another view, or several
of the view's own child components needing direct access rather than a prop) actually shows up.

**Carve-out:** a view that does nothing but compose child sections — no fetched data or shared state of its own — doesn't need a dedicated store either. It's fine for it to rely purely on `provide`/`inject` or on the stores/composables its child components already own. Don't manufacture an empty store just to satisfy a 1:1 rule that was never the actual rule.

Other stores (a shared `userStore`, a global `modalStore`) are used when actually needed. When a view uses another store, keep that store's own variable naming convention (`userStore`, not renamed to fit the local file) — see `GENERAL.md`'s naming rules.

A view calling a repository directly, without going through a store, is not an anti-pattern in
itself — it's expected when the view has no store at all (per the carve-out above), or when the
view does have a store but one specific repository is only ever needed in that one view and
nowhere else. See the project's own override file for how initial data fetching is wired up
concretely.

## Script Style & Import Grouping

Same as `COMPONENT.md`: Options API + `setup()`, same internal ordering and import-group headers/ordering from `GENERAL.md`.

## Related Components

Components that exist only to support one view live under `components/_views/<feature>/` and share that feature's name prefix (see `COMPONENT.md`). Keep a view's own template shallow — if a `<template>` block is doing more than composing named sections, that logic belongs in a child component or the view's store.

## Anti-patterns to flag, not silently fix

- A view with a large inline `<template>` that duplicates markup already available as a `_views` component.
- A dedicated store created for a view that has no actual state to hold (see the carve-out above) — flag as unnecessary indirection.
