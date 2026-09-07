# COMPONENT.md — Vue Components

This document covers reusable Vue components: both fully generic UI primitives (`components/form/VInput.vue`) and page-specific building blocks (`components/_views/<feature>/...`). Routed pages themselves are covered in `VIEW.md`.

## When to Create a Component

Extract a component when a section of a page carries real detail and logic of its own — a table, a page sidebar (e.g. a log/activity panel), or a form tied to a page or another component. These are self-contained units of behavior, not just markup, so they belong in their own file.

Don't extract a component just to shrink a template. Nesting components further inside an already-extracted component is only worth it when that inner piece independently meets the same bar (its own table/sidebar/form-level logic) — not by default.

Prefer props and emits to pass data down and events up between a component and its immediate parent. Avoid prop drilling: if a value has to pass through two or more intermediate components untouched just to reach a deeper one, that's a sign the component boundary is wrong or the data belongs in a store instead — see the Refactoring Rules in `GENERAL.md`.

## Script Style

Use the Composition API with a `setup()` function — **not** `<script setup>`:

```vue
<script>
export default {
    name: 'VInput',

    props: {
        modelValue: {
            default: null
        },
        type: {
            type: String,
            default: 'text',
            validator(value) {
                return /^(?:text|email|number)$/i.test(value);
            }
        }
    },

    emits: ['update:modelValue'],

    setup(props, { emit }) {
        // ...
    }
};
</script>
```

Internal ordering inside `setup()` follows `GENERAL.md`'s general script-ordering principle.

## Naming & Folders

- `components/<topic>/...` — fully generic, reusable anywhere (`components/form/VInput.vue`, `components/data-table/`). Prefix generic primitives with `V` (`VInput`, `VSelect`, `VCheckbox`) to distinguish them from feature components at a glance.
- `components/_views/<feature>/...` — components that exist to support one feature/view but aren't generic enough to live under a plain topic folder. The leading underscore marks "belongs to a view but isn't itself routed."
- Inside `_views/<feature>/`, name components by prefixing the feature name, matching the owning view: `PropertyCity*` lives under `_views/property/city/`, `FlightSearch*` lives under `_views/flight/search/`.
- Suffix conventions: `*Modal.vue` for a modal component, `*List.vue` for a list-rendering component, `*Bar.vue` for a toolbar/filter-bar, `*Banner.vue`/`*Banners.vue` for promotional banner blocks.
- Related components for one view share that view's namespace prefix so they sort and group together in the file tree and in autocomplete.

## Props & Emits

- Always declare props with the full options-object form (`type`, `default`, `validator` as needed) — never the bare array-of-names shorthand.
- Give array/object props a factory default (`default: () => []`, `default: () => ({})`), never a shared mutable literal.
- Use a real constructor (`String`, `Number`, `Boolean`, `Object`, `Array`) for `type` — never `{}` as a stand-in for "any object", that silently accepts anything and documents nothing; if the prop is genuinely open-shaped, use `Object`. Never add a JSDoc comment above a prop to document its shape — not here, not anywhere else in this codebase.
- Declare `emits` as a plain array of event-name strings (`emits: ['update:modelValue', 'submit']`). Add a validator function only when the payload shape genuinely needs runtime checking.
- A component that wraps a native form control and needs `v-model` follows the standard `modelValue` prop / `update:modelValue` emit pair.
- A prop name doesn't repeat a namespace the component's own file name already establishes — `FlightRuleDialog.vue`'s visibility prop is `show`, not `showFlightRules`; the file name already says "flight rule[s]." Same principle as `GENERAL.md`'s namespace rule, applied to props specifically.

## Composables Usage

A component that needs an initial async fetch wraps it the same way a view does — see the
project's own override file for the concrete helper, if one exists.

## Template Style

General Vue template syntax (quotes, wrapping, self-closing, attribute casing, `v-if`/`v-else`) follows `GENERAL.md`'s Vue Template Style section. Component-specific on top of that:

- Named slots: `<template #slotName="{ scopedProp }">`. Provide sensible fallback content inside a `<slot>` tag when the default rendering is usually fine as-is:

```vue
<slot name="price"> {{ formattedPrice }} </slot>
```

Class ordering and BEM conventions for template classes follow `STYLE.md`.

## Anti-patterns to flag, not silently fix

- Inconsistent self-closing-tag spacing (`<Foo/>` vs `<Foo />`) within the same file family — pick the first one and ask before repo-wide reformatting.
- A component reaching into a store it has no real ownership relationship with, instead of receiving data via props — flag per the Refactoring Rules in `GENERAL.md`.