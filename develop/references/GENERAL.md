# GENERAL.md — Cross-Project Frontend Principles

This document holds the coding rules that apply to **every** front-end project regardless of which UI framework/library (Bootstrap, Vuetify, or none) sits on top of Vue. Layer-specific rules live in their own files (`SERVICE.md`, `REPOSITORY.md`, `STORE.md`, etc.) and are indexed from `AGENTS.md`. Anything here can be narrowed or overridden by a project's own `PROJECT.md` (see `AGENTS.md`), but should not be contradicted without a stated reason.

These rules are derived from actual, recently-written code in real projects — not an aspirational style guide. Where a project's own code disagrees with a rule stated here, that's a signal to *ask whether to refactor*, not to silently copy the older pattern.

---

## Architecture

Data and behavior flow in one direction:

```text
Repository → Service → Store → View (→ Component)
```

- **Repository**: talks to the HTTP API. No business logic.
- **Service**: pure business logic / calculations, framework-independent, works on plain data.
- **Store**: framework state layer (Pinia or equivalent). Orchestrates repositories + services, owns reactive state.
- **View**: a routed page. Thin — composes components and reads/writes its store.
- **Component**: reusable UI, may belong to a specific view (`_views/`) or be fully generic.
- **Middleware**: a route guard (runs before entering a View's route) or an HTTP request/response
  interceptor (runs through the shared HTTP client). Never fetches data or touches a store, and
  never contains business logic specific to one repository/endpoint. Unlike the other layers above,
  middleware conventions (shape, naming, registration) are project-specific rather than defined
  here — see that project's own `PROJECT.md`.

Composables, Utils, Enums, and Constants are cross-cutting support layers used by any of the above.

Before writing new logic, **check whether an existing service, composable, repository, util, enum, or constant already provides it.** Reuse it instead of duplicating.

### Dependency Direction

Imports only flow one way, matching the arrow above — a layer never imports something that comes
after it or depends on it (no Service importing a Store or a Component, no Util importing a
Service). Concretely, and not always — only when the project actually has the thing in question:
- A **Service** may import Utils, and in some cases other Services.
- A **Repository** may import a generic Repository called `CrudRepository` if the project has one, and in
  some cases Utils as resource default values.
- A **Util** may import other Utils if needed.

Each layer's own doc (`SERVICE.md`, `REPOSITORY.md`, `UTIL.md`, etc.) states anything further it
specifically may or may not import.

---

## JavaScript

### General

- Prefer function declarations (`function doThing() {}`) over `const doThing = () => {}` unless the function is very short and fits comfortably in half a line.
- No space between the `function` keyword and the parameter parentheses — named or anonymous alike: `function doThing() {}`, `function(city) {}`, never `function doThing ()` or `function (city)`.
- In legacy Options-API files that rely on `this` (e.g. `methods` blocks), prefer arrow functions for short callbacks to avoid `this`-binding surprises. Plain `setup()`-style files have no such ambiguity.
- Semicolons are required everywhere they are syntactically valid.
- No trailing commas — in arrays, objects, or argument lists.
- Single quotes for strings in JS/JS-in-Vue logic. Templates use double quotes (see `COMPONENT.md`).
- Prefer a template literal over `+` string concatenation whenever a string is being built from static text and a variable/expression — including a single trailing/leading interpolation, not just multi-part strings: `` `enums.attribute_category.${ AttributeCategoryEnum.HOTEL }` ``, never `'enums.attribute_category.' + AttributeCategoryEnum.HOTEL`. Give the interpolation a space just inside the `${ }`, matching the object-brace spacing rule (see the Vue Template Style section below for the template-in-markup case). Reserve `+` for genuine arithmetic.
- Use `async/await`; avoid raw promise chaining (`.then()`).
- Every function that awaits a repository/service call wraps that call in `try`/`finally`, even when there's no loading flag to increment/decrement and the `finally` block ends up empty. This keeps the shape consistent everywhere and means adding loading state (or any other cleanup) later is a small diff instead of a restructure.

```js
// Good
async function fetchProperties() {
    try {
        const response = await PropertiesService.fetchWithSearch();

        return response.data.value.properties;
    } finally {}
}

// Bad — works today, but isn't in the standard shape
async function fetchProperties() {
    const response = await PropertiesService.fetchWithSearch();

    return response.data.value.properties;
}
```

When there *is* loading state, it goes in that same `finally` (see `STORE.md`'s own `fetch` example — `commit('DECREMENT_SENT_REQUEST_COUNT')`, `endLoading()`, etc.) — this rule just makes that the same shape even when that part doesn't apply yet.
- Do not add `catch {}` blocks purely to swallow errors — errors are handled globally (see the interceptor pattern in that project's own `PROJECT.md`). The only exception is when the code must inspect or react to something about the error (check an error code/type, emit an `'error'` event, clean up local state) before rethrowing. In that case: catch, react, and always `throw error;` again. Never swallow silently. This is a case-by-case necessity, not a default pattern to reach for.
- Insert a blank line after a group of unrelated variables/functions/declarations. Keep related refs, variables, and functions physically adjacent, with no blank line splitting them apart.
- Never write an arrow function whose only job is to return an object literal via implicit return:

```js
// Bad
const getData = () => ({ key: value });

// Good
function getData() {
    return { key: value };
}
```

### Explicit Conditions

Use explicit comparisons instead of relying on truthiness/falsiness coercion, especially for booleans and lengths.

```js
// Good
if (isVisible === true) { ... }
if (items.length === 0) { ... }
if (existing !== undefined && existing.items !== undefined) { ... }

// Bad
if (!isVisible) { ... }
if (!items.length) { ... }
if (existing?.items) { ... }
```

**Optional chaining (`?.`) and nullish coalescing (`??`) are never allowed, anywhere, with no exceptions** — not as a conditional shortcut, not inside a `computed`/`return` expression, not in a template interpolation or prop binding, and not even when a value is "deeply nested" or "genuinely optional." A value that can genuinely be absent still gets an explicit, spelled-out check at the level(s) where it can actually be missing — that's the whole guard, not a `?.`/`??` standing in for it. This is a hard rule, not a judgment call: if you find yourself reasoning about whether a particular `?.`/`??` is "safe" or "necessary" here, that reasoning itself means it must be rewritten as an explicit check instead.

```js
// Good
let logoUrl = null;
if (agencyStore.agency.logo !== undefined) {
    logoUrl = agencyStore.agency.logo.url;
}

// Bad
const logoUrl = agencyStore.agency.logo?.url !== undefined ? agencyStore.agency.logo.url : '';
const logoUrl = agencyStore.agency.logo?.url ?? '';
```

The same restriction applies to `||`/`&&` used to select or default a **value** rather than to combine booleans — `value || fallback`, `condition && 'result'` rely on the exact same truthiness coercion `?.`/`??` do, just spelled with a different operator, and are banned the same way. Check the absent/negative case explicitly first and return or assign early, then fall through to the real value — the same guard-clause shape a validation rule's early return already uses:

```js
// Good
function getPage(query, defaultPage) {
    if (query.page === undefined) {
        return defaultPage;
    }

    return query.page;
}

// Bad
const page = query.page || defaultPage;
const label = items.length && 'has items';
```

`||`/`&&` remain fine for combining two already-explicit boolean comparisons into a single condition — that's not truthiness coercion, both sides are already real booleans:

```js
// Still fine
if (isSignedIn === true && hasAccess === true) { ... }
if (input.type === InputTypeEnum.RADIO || input.type === InputTypeEnum.CHECKBOX) { ... }
```

This applies just as much inside a Vue template — pull the check into a `computed` in `setup()` rather than reaching for `?.` in the markup:

```js
// Good (setup())
const firstTicketCreatedAt = computed(function() {
    if (tickets.value[0] === undefined) {
        return null;
    }

    return tickets.value[0].created_at;
});
```

```html
<!-- Bad -->
<FlightVoucherHeader :reserve-created-at="tickets[0]?.created_at" />
```

This applies just as much to a predicate function's/computed's return value as to a plain boolean variable — don't rely on the call's own truthiness:

```js
// Good
if (canManageContract(item) === true) { ... }
if (remainingDays(item) !== null) { ... }

// Bad
if (canManageContract(item)) { ... }
```

When negating a multi-part condition as a whole, wrap the whole condition in its own parentheses before the comparison, so it reads as one unit being checked rather than a dangling `&&`/`||` chain:

```js
// Good
if ((promotion.start_at < now && now < promotion.end_at) === false) { ... }

// Bad
if (promotion.start_at < now && now < promotion.end_at === false) { ... }
```

A value's actual origin decides which presence check(s) it needs — never reach for
`!== null && !== undefined` together as a reflex pair "just in case." Trace where the value
actually comes from (an API field's documented type, a repository's `getDefault()`, a literal
passed at the call site) and check only the state it can really be in. A field typed/documented as
`array | null` (never `undefined`) only ever needs a `null` check; a plain object-property lookup
(`someMap[key]`) only ever needs an `undefined` check, since a missing key reads as `undefined`,
never `null`. Checking both when only one is actually reachable doesn't add safety — it just
obscures which state the code is really guarding against, and reads as if the value's shape were
never pinned down.

```js
// Bad — every call site passes this function either a real array or a literal `null`;
// `undefined` never occurs here, so checking for it adds nothing
function buildConditions(structure, existingConditions) {
    const existingByKey = existingConditions !== null && existingConditions !== undefined
        ? keyBy(existingConditions, 'key')
        : {};
    ...
}

// Good
function buildConditions(structure, existingConditions) {
    const existingByKey = existingConditions !== null
        ? keyBy(existingConditions, 'key')
        : {};
    ...
}
```

The exception is a nested/inner key that may genuinely be absent independent of its container's
own presence — verify this from the same source (the API doc, the shape's own definition), don't
assume it just because the key happens to be nested. A condition's `items` array is a real example:
per this project's own attribute API, only `checkbox`/`radio` conditions carry an `items` field at
all — a `text`/`number` condition's record omits it entirely, independent of whether the condition
itself was found. That's a genuinely, independently-optional inner key, so it gets its own check on
top of (not instead of) whatever check the outer lookup already needed:

```js
// Good — `existing` itself was already confirmed found; `existing.items` is a separate,
// genuinely-optional field (only checkbox/radio conditions carry one) that needs its own check
if (existing !== undefined && existing.items !== undefined) { ... }
```

The mirror mistake is checking a nested key that's actually guaranteed present whenever its
container is — that's the exact same over-checking the rule above warns about, just one level
deeper. If a shape's own definition says a field always exists once the object itself does, adding
an `undefined` check for that field on top of checking the object isn't "extra safe," it's noise:

```js
// Bad — the API documents `id` as always present on this entity; checking it on top of
// checking `attribute` itself adds nothing
if (attribute !== undefined && attribute.id !== undefined) { ... }

// Good
if (attribute !== undefined) { ... }
```

### Initial / Default Values

- `null` for most uninitialized scalar variables — not `undefined`.
- `[]` for arrays.
- `{}` only when the shape is genuinely open-ended — otherwise prefer explicit keys defaulted to `null`.
- `true`/`false` explicitly for booleans.
- Avoid an empty string (`''`) as an "unset" default — prefer `null` (or `undefined` when that's genuinely the more fitting absence value for the situation). The one common exception is a ref bound directly to a text input's `v-model` (or similar raw DOM-value binding), where `''` is the natural empty state of the control itself — that's a UI binding, not a domain-state placeholder. When a piece of state can be set back to `''` by normal user interaction (e.g. clearing a text field) as well as by a reset, treat both `null` and `''` as the "inactive" value at every read site rather than picking only one.

### Conditionals

- Prefer regular `if` statements over ternaries once a condition needs more than a single short expression.
- Ternaries are allowed only when the entire expression fits on one line.
- Use `switch` only when there are 3+ branches on the same discriminant; otherwise prefer `if`.
- Always use braces, even for one-line bodies.

```js
// Good
if (isVisible === true) {
    doSomething();
}

// Bad
if (isVisible === true)
    doSomething();
```

### Loops & Iteration

Prefer a plain loop (`for...of`, `for...in`, `.forEach`, a classic indexed `for`) over chained
array methods (`.map()`, `.filter()`, `.reduce()`) when it's genuinely faster and doesn't make the
code harder to follow — e.g. a single pass that needs to bucket items into more than one group, or
a hot path over a large collection. This is a judgment call, not a blanket ban on `.map()`/
`.filter()`: a short, single-purpose transform over a small/typical array is usually clearer as a
chain, and clarity wins when the two are close. Never write a loop inside a loop over the same or
related data when a single pass (or a lookup built once beforehand, e.g. `keyBy`) would do the same
work — that's an unprompted `O(n*m)` cost for no benefit.

### Strings & Magic Values

Avoid raw strings and magic numbers scattered through business logic. In order of preference:
1. Search for an existing enum/constant that already covers it.
2. A new **enum** (`ENUM.md`) if the value is part of a closed, single-concept category.
3. A new **constant** (`CONSTANT.md`) if it's a scattered fixed value tied to one file/subject rather than a category.

### Internal Ordering (`setup()`-shaped files)

Any file built around a `setup()` function — component, view, store, or composable — orders its
internals top to bottom in one consistent shape:

1. Other composables/stores/route this file depends on (`useLoading()`, `useXStore()`, `useRoute()`).
2. Static values / `ref`/`reactive` state.
3. `computed` derived values.
4. Functions (plain, then `async`).
5. `watch`.
6. The `return { ... }` object last, in three tiers, each its own group separated by a blank line:

    1. **First** — anything obtained directly from calling a composable (`const route = useRoute();`,
       `const { isLoading, startLoading, endLoading } = useLoading();`, `const { tableOptions, items,
      total } = useDataTable();`), if it's part of the return object at all. This goes first
       regardless of whether it's also used elsewhere in script or only in the template — `store`,
       `isLoading`, `route`, `router`, `tableOptions`, `formIsValid`/`validationRules` from a shared
       form composable, etc. all belong here. This tier is not exempt from the same relatedness
       grouping as the middle tier: results that came from the *same* composable call stay adjacent
       (`tableOptions`, `items`, `total` from one `useDataTable()`), but an unrelated composable's
       output (`isLoading` from a separate `useLoading()`) still gets its own blank-line-separated
       group within this tier, not folded into the first one just because both are composable-derived.
    2. **Middle** — the file's own locally-declared state/computed/functions (a plain `ref`/`reactive`
       created in this file, a `computed`, a function defined in this file) — grouped by actual
       semantic relatedness, not alphabetically and not simply by whether a value happens to be a
       `ref`, a `computed`, or a function. Two entries that describe or operate on the same concern
       sit adjacent with no blank line between them; a blank line separates that group from the next
       one, even when both fall in the same mechanical category (two unrelated `computed`s still get
       a blank line between them). Ask "does this value actually get used together with its neighbor,
       or read/change for the same reason?" — if not, it's a different group.
    3. **Last** — anything that is itself a direct import from another file (an enum, a util function,
       a constant — e.g. `import InputType from '@/enums/input-type.enum';` re-exposed to the
       template for a comparison like `input.type === InputType.RADIO`) **and** is read from the
       template. This goes last even when the same import is *also* read from a function in this
       file — the deciding factor is that the value's source is an external file, not this
       component's own state.

```js
// Good — composable-derived first, this file's own state/computed/functions grouped by concern,
// the imported InputType enum last because the template reads it directly
return {
    isLoading,
    formIsValid,
    validationRules,

    formData,
    inputs,

    isUpdateForm,
    title,
    submitButtonText,

    showDialog,
    close,

    submit,

    InputType
};

// Bad — isLoading buried at the end instead of leading, InputType mixed into the feature-specific
// group instead of trailing on its own, dialogShow/close split from each other
return {
    isUpdateForm,
    title,
    submitButtonText,
    showDialog,
    InputType,

    close,
    submit,

    isLoading
};
```

That returned object is always a plain object of refs/computed/functions — never a `reactive()`
wrapper directly (that breaks destructuring reactivity for the consumer).

### Composables Usage

Always destructure a composable's return value directly at the call site, in a component, view,
store, or another composable:

```js
const { startLoading, endLoading, isLoading } = useLoading();
```

---

## Naming

- An enum/constant/config/util file's default export (and any class-shaped util export, e.g.
  `DateTime`) is itself named with its layer suffixed (`ContractTypeEnum`, not `ContractType`;
  `CampaignConstant`, not `Campaign`; `PaymentConfig`; `DateTimeUtil`) — this is the name declared
  in the file itself, exported, and used identically by every consumer (`ContractTypeEnum.SIGN`,
  never re-imported under a shorter local name). This applies to the single default-exported
  object/class per file that each of `ENUM.md`/`CONSTANT.md`/`CONFIG.md` centers on, and to a
  util's class-shaped exception (`UTIL.md`) — it does not apply to a plain function exported from a
  util file (those keep their verb-based name per the Naming rules above, e.g.
  `normalizePersianText`, not `normalizePersianTextUtil`), nor to a repository/service/store/
  composable's own export, which already carries its layer as a file-name suffix, not an
  identifier suffix.
- Never repeat a namespace that the enclosing object/file already provides.

```js
// Bad
roomStore.fetchRooms();
roomStore.roomList;

// Good
store.fetch();
store.list;
```

This applies to function arguments too: if a file's own namespace is `property` and a function
inside it takes an argument that would otherwise be called `property_id`, name it `id` instead —
the enclosing file already established which entity the id belongs to.

```js
// Bad — property.repository.js
function getRoomTypes(property_id, config) { ... }

// Good — property.repository.js
function getRoomTypes(id, config) { ... }
```

The same applies to a loop/iteration callback's parameter when it's iterating a variable that already carries the domain name — don't re-name the parameter after the array:

```js
// Bad
properties.map(property => property.id)

// Good
properties.map(item => item.id)
```

- Omitting a namespace/qualifier only works when the *enclosing scope's own single purpose* is
  what actually explains the name — not whenever a short name is merely convenient. A View's
  `items`/`fetch`/`total` are self-evident because that View's whole reason for existing is "the
  list this page shows" — there's nothing else `items` could mean in that file. A dialog's own
  `show` prop / local `isShown` computed are self-evident the same way, because that file *is* the
  dialog (see `COMPONENT.md`'s `show`/`isShown` pattern) — reading `isShown` inside the dialog
  answers "is *this component* shown?" with no other candidate referent. That reasoning breaks the
  moment the value's real subject is a *child* the enclosing file merely renders, rather than the
  enclosing file's own concern: a View's `ref` that exists only to feed one specific child dialog's
  `show` prop describes that dialog, not the View — inside the View's own file, "is this View
  shown?" isn't even a meaningful question, so `isShown` there answers nothing and only reads
  correctly by mentally substituting in the child's name. Name it for what it actually toggles
  instead, so it stays legible without that substitution and so a second dialog added later can't
  collide with it.

```js
// Bad — PropertyAttributesView.vue; "isShown" describes nothing about this View's own domain,
// only about the dialog it's handed to
const isShown = ref(false);
...
<PropertyAttributeDialog v-model="isShown" .../>

// Good
const isCreateDialogShown = ref(false);
...
<PropertyAttributeDialog v-model="isCreateDialogShown" .../>
```

- Name the reactive value that a fetch/response result is stored into `items` when it holds an
  array — regardless of domain (`items`, not `properties`/`rooms`/`flights`), since the enclosing
  file/store/composable/component already gives the domain context. This applies wherever the
  pattern shows up (stores, composables, components), not just one layer.
- A boolean flag — a variable, ref, or predicate function/computed that only ever holds
  `true`/`false` — is prefixed `is`/`has` (`isLoading`, `isActive`, `hasParamsText`), never a bare
  adjective/noun (`loading`, `active`, `paramsText`).
- Every function name starts with a verb describing the action it performs (`fetchCityProperties`,
  `getStatusLabel`, `startLoading`, `normalizePersianText`) — never a bare noun. A predicate
  function's verb is `is`/`has` per the flag rule above (`isDisabled(contract)`).
- Choose semantic, descriptive names over short or clever ones. This applies to function parameters
  too, not just variables: a validation-rule function takes `value`, not `v`; an event handler
  takes `event`, not `e`. The one accepted exception is a classic indexed `for` loop counter
  (`i`/`j`), where the convention is already unambiguous.
- Name a `ref`/`reactive` after what it actually holds, not after its generic mechanism/type — a
  form's editable data is `formData` or such, never the
  bare word `form`, which reads as the `<form>`/`VForm` element itself and collides in meaning with
  a sibling `formRef`.
- A parameter that must be declared only to preserve position for a later parameter that *is* used, but is never itself referenced in the function body, is named `_` instead of its normal descriptive name:

```js
// Bad — props is declared but never referenced
setup(props, { emit }) { ... }

// Good
setup(_, { emit }) { ... }
```

If more than one leading parameter is unused, distinguish them (`_1`, `_2`, ...) — a function can't declare two parameters with the same name.
- Do not name a local variable (including a nested function's parameter or local `const`) the same as an outer `ref`/`computed`/`const` already declared in the same scope. It silently shadows the outer binding — valid JS, but misleading, and it makes the outer value unreachable by that name inside the inner scope. Rename the local instead.
- Avoid repeating a file's, object's, or function's own name inside its members.

---

## File Naming

Except for framework single-file-components (`.vue` files use PascalCase matching the exported component name — see `COMPONENT.md`/`VIEW.md`):

- Use singular nouns.
- Use lowercase, kebab-case.
- Include the file's role/layer as a suffix — this applies uniformly to every layer, with no
  per-layer exception (enums included: `contract-type.enum.js`, never `ContractType.js`).

```text
room-rate.store.js
rule.repository.js
facility.service.js
countdown.composable.js
contract-type.enum.js
promotions.constant.js
payment.config.js
date-time.util.js
string.filter.js
check-tour-search-route.middleware.js
```

## No Domain Subfolders

Enums, constants, repositories, composables, services, stores, utils, and middleware all stay
**flat** inside their layer's root folder (`services/facility.service.js`, not
`services/property/facility.service.js`). Don't create a subfolder to group files by domain/feature
(`services/ecommerce/`, `stores/property/`, `composables/flight/`, `enums/property/`,
`router/middleware/flight/`) — every file of these types lives directly under its layer folder,
distinguished by its (kebab-case, prefixed) file name alone. This applies to new files regardless
of what an existing project's older code already does.

---

## Import Ordering

Group imports under short comment headers, with prioritizing base files(the global ones or not customized for the target file) and then relative files in this order (omit any group with no members — never leave an empty header):

```js
// Vue
// Store
// Composables
// Components
// Directives
// Libraries
// Utils
// Services
// Repositories
// Enums
// Constants
// Config
```

- Prefer path-alias imports (`@/...`) over long relative chains whenever an alias is configured for that path.
- Keep the ordering consistent across the codebase. If a file's imports drift from this order, that's a refactor candidate for its own layer's doc — ask before batch-fixing unrelated files, per the Refactoring Rules below.

---

## Object Formatting

- Keys: lowercase, snake_case, so the shape matches the API payload end-to-end (backend → repository → store → template).
- Multi-line once an object has more than one key:

```js
{
    key_1: valueOne,
    key_2: valueTwo
}
```

- Always put a space just inside non-empty braces: `{ key: value }`, never `{key: value}`.

---

## Vue Template Style

- Double quotes for all attributes.
- An element with more than 1 attribute goes multi-line, one attribute per line, with the closing `>` (or self-closing `/>` for the ones that is standard like `input`) on its own line:

```vue
<span
    v-if="isLoading"
    class="spinner-border spinner-border-sm text-light"
></span>
```

- Elements that fit on one line stay on one line — don't wrap for the sake of wrapping:

```vue
<span v-else> {{ $t('delete') }} </span>
```

- Self-close Vue components and views with no content (`<PropertySort/>`), but never self-close native HTML elements (`<div></div>`, not `<div/>`).
- Attribute names are kebab-case (`:items-length="total"`).
- A template literal's interpolation inside a Vue template expression gets a space just inside the `${ }`, matching the object-brace spacing rule: `` `enums.age_range.${ TypeOfPassengerEnum.ADULT }` ``, never `` `enums.age_range.${TypeOfPassengerEnum.ADULT}` ``.
- Use `v-if`/`v-else` for true either/or rendering; reach for `v-else-if` only when there are 3+ mutually exclusive branches, and prefer nested `v-if` blocks or a computed class/value over long `v-else-if` chains when the branches themselves get complex.

---

## Formatting / Indentation

| File type | Indent |
|---|---|
| JavaScript / TypeScript | 4 spaces |
| Vue templates | 4 spaces |
| SCSS / YAML | 2 spaces |

Enforce via `.editorconfig` (UTF-8, LF line endings, trim trailing whitespace, final newline) at minimum. No project has (or needs) an ESLint/Prettier setup enforcing the style rules across this document family by default — unless the project's own `PROJECT.md` says otherwise, treat every rule here as team convention, not a tooling guarantee: follow it by reading surrounding code, flag drift rather than silently "fixing" unrelated files, and don't claim a rule is "enforced" when it's actually convention-only — say so plainly when asked.

No single trailing newline at end of file is allowed.

---

## Refactoring Rules

Whenever logic is duplicated across files:

1. Move it into the correct layer (Service, Composable, Util, Enum, Constant, etc. — pick by the definitions in each layer's doc).
2. Follow that layer's conventions for the new shared implementation.
3. Ask whether existing call sites should be refactored to use it. Don't refactor unrelated files unprompted.

Whenever any rule in these documents is violated in code you're touching (including import ordering), ask whether the surrounding code should be brought into line — don't silently rewrite things outside the scope of the current task, and never leave two competing styles behind in the same area.

---