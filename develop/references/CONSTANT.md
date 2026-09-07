# CONSTANT.md — Scattered Fixed Values

A constant file holds scattered, loosely-related fixed values that all just happen to belong to
one narrow topic/subject — not a categorical "type" enumerating every option of a single concept
(`ENUM.md`), and not a tunable per-environment default (`CONFIG.md`). Values don't need to share a
"kind" the way enum members do; they only need to share a subject.

---

## Shape

Structurally identical to `ENUM.md` — declare a named object, then default-export it wrapped in
`Object.freeze(...)`, even for a single value:

```js
const CampaignConstant = {
    TIERED_DISCOUNT_CAMPAIGN_ID: 39,
    POPUP_DISCOUNT_CAMPAIGN_ID: 33
};

export default Object.freeze(CampaignConstant);
```

Unlike an enum, the values don't need to be interchangeable "kinds" of the same thing:

```js
const SearchConstant = {
    MIN_QUERY_LENGTH: 2,
    DEBOUNCE_MS: 300,
    MAX_RESULTS: 20
};

export default Object.freeze(SearchConstant);
```

## Rules

- File name is singular kebab-case with a `.constant.js` suffix, describing the subject
  (`campaign.constant.js`).
- The exported object's own name always carries the `Constant` suffix (`CampaignConstant`, not
  `Campaign`) — every consumer imports and uses it under that exact same name
  (`CampaignConstant.TIERED_DISCOUNT_CAMPAIGN_ID`), never re-imported under a shorter local alias
  (see `GENERAL.md`'s Naming rules).
- Use large-number separators for readability (`500_000`, `44_000`), not `500000`.
- Keys must be `SCREAMING_SNAKE_CASE`; values must be `snake_case` strings, and also numbers.
- Don't let a constants file grow into a second enum: if a value represents "one of several kinds
  of X" — i.e. every value could stand in for every other one in the same code path — it belongs
  in an enum instead (`ENUM.md`).