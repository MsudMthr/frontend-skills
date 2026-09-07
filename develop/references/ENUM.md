# ENUM.md — Closed-Category Values

An enum represents a single closed category of related values — every valid option for one
concept, and nothing else. See `GENERAL.md`'s "Strings & Magic Values" section for when to reach
for an enum versus a constant.

---

## Shape

```js
const ContractTypeEnum = {
    SIGN: 'sign',
    AFFILIATE: 'affiliate',
    WELFARE: 'welfare',
    IPG: 'ipg'
};

export default Object.freeze(ContractTypeEnum);
```

## Rules

- File name is singular kebab-case with an `.enum.js` suffix, matching the enum's own name
  (`contract-type.enum.js`, exporting `ContractTypeEnum`).
- The exported object's own name always carries the `Enum` suffix (`ContractTypeEnum`, not
  `ContractType`) — every consumer imports and uses it under that exact same name
  (`ContractTypeEnum.SIGN`), never re-imported under a shorter local alias (see `GENERAL.md`'s
  Naming rules).
- Default-export the object wrapped in `Object.freeze(...)` — always frozen, no exceptions.
- Keys are `SCREAMING_SNAKE_CASE`; values are whatever the backend/domain actually uses
  (usually `snake_case` strings, sometimes numbers) — the value is not required to match the key's
  casing, it must match the wire format.
- One enum = one category. A `ContractTypeEnum` must only ever contain contract types; if a new,
  unrelated category of values shows up, it gets its own enum file, never appended to an
  unrelated existing one.
- Keep every enum file flat under `enums/` — don't create a per-domain subfolder
  (`enums/property/`) to group them; the file name alone (`property-type.enum.js`,
  `property-grade.enum.js`) is enough to keep related enums next to each other alphabetically.
- Enums that mirror a value defined and owned by the backend (rather than a frontend-only
  concept) go in `enums/backend/` so it's obvious at a glance which enums must be kept in sync
  with the API contract vs. which are purely presentational (sizes, colors, positions). This is
  the one accepted subfolder for this layer — it's a sync-with-API marker, not a domain grouping.
- Before adding a new enum member or a new enum file, check whether the same category already
  exists under a different name.