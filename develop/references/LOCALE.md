# LOCALE.md — i18n / Translations

All translations live under one `locales/<lang>/` folder per language. A project may support exactly three files per language, assembled into one export by an `index.js`:

```js
// locales/fa/index.js
import Messages from './messages';
import Enums from './enums';
import Validation from './validation';

export default Object.assign(Messages, {
    enums: Enums,
    validation: Validation
});
```

- `messages.js` — general UI copy.
- `enums.js` — translated labels for enum values, nested under an `enums.` namespace at lookup time (`t('enums.property_type.hotel')`).
- `validation.js` — validation error message templates, nested under a `validation.` namespace (`t('validation.required')`).

Any other file or folder under a language directory is non-conforming — ignore it if found, don't add new ones without a stated reason.

## Adding New Entries

- Append new entries at the **end** of the file — never insert mid-file. This applies to
  `messages.js` keys, `enums.js` per-enum sections, and `validation.js` rules alike.
- Add the exact same entry, in the same order, to every language's file at the same time — never
  land one in only a single language. Treat the reference/default language's file as the source of
  truth for "what's new and in what order" if the files ever drift out of sync.

## `messages.js`

- New keys are **snake_case**.

```js
{
    room_min_stay_error_message: 'این اتاق برای رزرو های بیشتر از {day} شب در دسترس است',
    room_price_calendar: 'تقویم قیمتی اتاق'
}
```

- Older keys in a long-lived file may not follow this (legacy PascalCase/camelCase keys like `Star`, `SimilarProperty`). Leave them as-is.

### Backward Compatibility

If an existing key doesn't follow the snake_case convention:
- Create a new key following the convention for new usages.
- Keep the old key in place — do not remove it unless explicitly instructed. Removing it requires refactoring every call site that still references it.

### Before Adding a Key

- Check whether the same meaning already exists elsewhere in the file and reuse it instead of duplicating — but only when that existing key already follows the snake_case convention (see Backward Compatibility above for the alternative). This check is forward-looking only — don't refactor it.
- Namespace keys tied to a specific feature/page with that feature/page's name (e.g. `contract_lead_description`, `property_map_slider_not_fount`) whenever a generic word (`title`, `code`, `status`) could plausibly mean something different in another feature's context. Truly generic, unambiguous words don't need a namespace prefix.
- If a new key would otherwise need the same name as an existing key but with a different meaning, give it a distinguishing name instead of overloading the existing one — then ask whether call sites using the old key should be migrated to the new one and the old key retired. Don't perform that migration without asking first.
- A key's name must describe what its **value** actually says, not what the surrounding feature happens to be about. `attributes_management: "ویژگی‌های اقامتگاه"` (a plain "Property Attributes" label, no "management" concept anywhere in the value) is a real example of this drifting — the key was renamed to `property_attributes` to match what the value actually says, rather than leaving a name that implies semantics the value doesn't carry.
- The namespacing rule above cuts both ways: don't bake a feature/page name into a key whose value is actually a generic, reusable phrase just because its first (and only) use happens to live in that one feature. `complete_starred_attributes_message: "ویژگی‌های ستاره‌دار (*) را تکمیل کنید"` was really a generic "complete the starred/required fields" validation banner, wrongly named and worded as if it only ever meant attributes — renamed (and reworded) to `complete_starred_fields_message` so it reads correctly if any other required-field validation UI reuses it later.
- A key's naming vocabulary should match the term the UI itself already uses for a concept, not an internal/backend field name that differs from it. A backend `structure` array that every other label in the same feature already calls "details" in the UI (`attribute_details`, a "جزئیات" button) belongs in a key named `attribute_details_count`, not `attribute_structure_count` — using the raw API field name in the key just reintroduces the exact word mismatch the UI copy already resolved.
- Drop a suffix (`_column`, `_label`, `_text`) that adds no distinguishing meaning once the base phrase alone is already unambiguous — `recorded_values` over `recorded_values_column` when there's no sibling `recorded_values` key it would collide with.

## `enums.js`

Maps an enum's values to translated labels, one top-level key per enum, imported directly from the enum file it labels:

```js
import PropertyTypeEnum from '@/web/enums/property-type.enum.js';

export default {
    'property_type': {
        [PropertyTypeEnum.MOTEL]: 'متل',
        [PropertyTypeEnum.HOTEL]: 'هتل',
        [PropertyTypeEnum.VILLA]: 'ویلا'
    }
};
```

- The top-level key is the snake_case name of the enum's own concept, not its filename
  (`flight-layover.enum.js` → key `flight_layover`).
- Every case must key off the imported enum's members (`[PropertyTypeEnum.MOTEL]: ...`), never a hand-typed copy of the same string — so a renamed/removed enum member breaks visibly instead of silently going stale.

## `validation.js`

Validation message templates, keyed by the validation rule name (`required`, `digits`, `email`, `min`, `max`). Rules that vary by target type nest one level (`min: { string: '...' }`) to support future non-string variants (`min: { numeric: '...' }`) without renaming the top-level key.