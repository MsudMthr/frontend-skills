# STYLE.md — CSS / SCSS

This document covers general CSS authoring rules that hold regardless of which component/utility framework (Bootstrap, Vuetify, none) a project layers on top. Framework-specific setup — which utility classes exist, how theme variables are overridden, build/import order — belongs in that project's own `PROJECT.md`.

## Indentation

2 spaces for SCSS (see `GENERAL.md`).

## Naming Convention

Follow a BEM-derived convention (`block__element--modifier`) for component-owned classes, with one
deliberate deviation from strict BEM: an element name may itself be chained from a parent element
when it's a sub-part of that element specifically, rather than a sibling of it —
`property-search__fast-reserve__placeholder` (a placeholder that belongs to the `fast-reserve`
element, not to the block directly) is valid here, not an error to flatten. Nest `&`-selectors to
match:

```scss
.property-card-horizontal {
    &__image { ... }
    &__special-offer { ... }
}

.property-search {
    &__fast-reserve {
        &__placeholder { ... }
    }
}
```

Use judgment on when a sub-element chain is warranted versus just a new top-level element on the
same block — chain it when the element is conceptually *part of* another element (and would make
no sense detached from it), keep it a flat sibling element otherwise.

- A fully generic, reusable-anywhere component gets its own SCSS partial, named to match the component (`components/property/property-card-horizontal.scss` for `PropertyCardHorizontal.vue`).
- A component that instead belongs to one specific view (rendered only by that view, not meant to be reused elsewhere) does **not** get a separate partial of its own — no one-file-per-component sprawl for view-owned pieces. Its styles fold into the owning view's existing partial as a new BEM element, chained under the view's own block name the same way the sub-element chaining above works:

```scss
// property-search-view.scss — owns PropertySearchFastReserveModal.vue, which it alone renders
.property-search-view {
    &__fast-reserve-modal {
        &__close-button { ... }
    }
}
```

- Utility classes (layout, spacing, typography helpers provided by the underlying framework or a shared utilities partial) are used directly in templates rather than re-declared per component — don't wrap a plain utility combination in a new BEM class just to give it a name.
- A static `style` attribute (a fixed value, not a `:style` binding) belongs in a class instead of inline — the component's own BEM element (or, for a view-owned component, the owning view's element per the rule above). Reserve inline `:style` bindings for values genuinely computed at runtime that can't be expressed as a static class.

## Class Order (in templates)

When a template element has both a component-specific class and utility classes, order them:

1. The component's own BEM class.
2. Display/layout classes.
3. Other utility classes.
4. Typography classes.
5. Spacing classes.

```html
<div class="property-card-horizontal d-flex align-items-center body-1 mb-2">
```

## File Organization

Organize partials by role, imported in a deliberate, dependency-respecting order from one entry stylesheet:

1. Functions
2. Variables
3. Mixins
4. Framework-level resets/typography/utilities
5. Shared helpers
6. Components (one partial per component/family)
7. Layout (header/footer/shell)
8. Pages (one partial per page/view, for page-specific overrides that don't belong to a single component)

Don't add a new global helper/mixin for something only one component uses — keep it local to that component's partial until a second consumer actually appears, or it's obvious that would be converted to a repeated one in the future.

## RTL / i18n-sensitive styling

If the project supports right-to-left languages, isolate any styles that must **not** be mirrored (icons, specific directional overrides) inside an explicit ignore block understood by the RTL build tool, and keep that block as small and well-commented as possible — see the project's own `PROJECT.md` for the exact tool/syntax in use.

## Anti-patterns to flag, not silently fix

- A component-specific BEM class defined but never actually used because the template only uses utility classes — dead CSS.
- An element chained more than one level deep for no real reason (a sub-element chain should stop being useful reading as "part of its parent element" — if it does, that's a signal that it should be flattened to a plain sibling element instead).
- A property in a custom SCSS rule that duplicates a utility class already available for it — whether that class comes from the underlying framework (Bootstrap's own `rounded-circle`, `bg-danger-subtle`, etc.) or from this project's own generated utilities (a `bg-<name>`/`text-<name>` pair already generated from a shared color map). Apply the class in the template and drop the property from the SCSS rule instead of restating it. Hard-coded colors/spacing values duplicating an existing variable/utility class fall under the same rule.
- Using Style element inside views or components.