# UTIL.md — Utils & Filters

Utils hold **framework-independent, product-independent** logic — code that would work unchanged if pasted into a different project entirely. If the logic touches Vue reactivity or lifecycle, it belongs in `COMPOSABLE.md` instead.

## Shape

- Export **plain functions**, not classes — the vast majority of utils are single-purpose pure functions: `debounce`, `deepCloneObject`, `keyBy`, `sortByDesc`, `isEmpty`, `groupBy`.
- The only time a util is class-shaped is when it wraps a genuine multi-method, stateful API that doesn't make sense as separate functions — e.g. a `DateTimeUtil` wrapper around a calendar library that holds an internal date value across chained calls (`DateTimeUtil.make().gregorian().addDay(1).format(...)`). Reach for a class here only when chaining/internal state is the actual point, not as a default style. This class-shaped export's own name always carries the `Util` suffix (`DateTimeUtil`, not `DateTime`) — every consumer imports and uses it under that exact same name, never re-imported under a shorter local alias (see `GENERAL.md`'s Naming rules). A plain function util does **not** get this suffix — it keeps its own verb-based name (`normalizePersianText`, not `normalizePersianTextUtil`); the `Util` identifier suffix applies only to a file's single class-shaped default export, not to every named function a util file exports.
- A barrel `utils/index.js` re-exports the small, frequently-used pure functions instead of one import per file. Larger or more specialized utilities (the `DateTimeUtil`-style class, DOM-heavy helpers) stay in their own file and are imported directly, not necessarily re-exported from the barrel.
- One concern per file when a util grows beyond a couple of functions (`cookies.util.js`, `query.util.js`, `serializer.util.js`, `breakpoints.util.js`) rather than piling everything into the barrel file itself.

```js
// utils/text-comparison.util.js
function normalizePersianText(value) {
    return value.replace(/ي/g, 'ی').replace(/ك/g, 'ک');
}

export {
    normalizePersianText
};
```

Declare each function without an inline `export` keyword, then export all of a file's public
functions together in one multi-line named-export block at the bottom (one name per line, even
when there's only one) — this matches the barrel file's own shape and keeps every file's public
surface visible in one place at a glance, rather than scattered across inline `export function`
declarations.

## Filters

A "filter" here means a small, schema-driven, stateful transform object grouped by data type — distinct from the plain-function utils above. Use this shape when you need a family of interchangeable value-handlers with a shared contract (e.g. a query/search filter that knows its own default value, how to read/write itself from a query string, and whether it's currently active):

```js
class StringFilter extends BaseFilter {
    valueOf() {
        return String().valueOf();
    }
}
```

- One file per data type (`array.filter.js`, `boolean.filter.js`, `number.filter.js`, `object.filter.js`, `string.filter.js`), each extending a shared `base.filter.js` that defines the common contract (`getFilterValue`, `getQueryValue`, `isActive`, `isMatch`, `getLabel`).
- Don't reach for this pattern for a one-off transform — that's a plain function util. Reserve it for genuinely interchangeable, schema-driven handlers.

## `utils/` vs a framework-coupled `utils/` folder

If the project also has a second, framework-coupled utils folder (e.g. `vue/utils/` sitting next to the framework-independent `utils/`), the split is:

- **Framework-independent `utils/`**: no import of `vue`, no VNode/slot access, works in plain Node or any other frontend stack.
- **Framework-coupled utils**: helpers that operate directly on framework internals (e.g. reading VNode/slot data) or that are only ever consumed from framework code, even if the function body itself doesn't import the framework package (e.g. a validator class that's technically framework-agnostic but is only wired up through framework-specific composables/directives).

When in doubt, ask: "would this file still make sense in a plain script with no UI framework loaded at all?" If yes, top-level `utils/`. If no, the framework-coupled folder.

## Rules

- File name is `<subject>.util.js` (or `<subject>.filter.js` for the Filters family), kebab-case,
  singular.
- Beyond `GENERAL.md`'s Dependency Direction rule, a util may also import a very low-level constant where genuinely unavoidable.
- Prefer pure functions with no side effects and no shared mutable module-level state. If a util needs to cache something across calls, make that explicit and document why (e.g. `cachePromise`), don't default to it.
- Use large-number separators for readability in constants passed through utils (`500_000`, not `500000`).
- Before writing a new util, search the barrel file and existing util files for an equivalent — this layer accumulates near-duplicate helpers (`unique` vs `uniqueBy`, `sortBy` vs `sortByDesc`) faster than any other, so check both singular and "by key" variants already exist before adding a new one.