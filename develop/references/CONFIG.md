# CONFIG.md — Environment-Dependent Values

A config file holds only values that must differ between environments in a way that changes
behavior (e.g. sandbox vs. live payment gateway) — not a bag of default numbers just because they
feel configurable. If a value is the same everywhere, it's a constant (`CONSTANT.md`) or an enum
(`ENUM.md`), not config.

---

## Shape

Keys are `lower_snake_case` — unlike `ENUM.md`/`CONSTANT.md`, not
`SCREAMING_SNAKE_CASE`, so a config object's shape reads the same as the rest of a runtime
data payload rather than like a fixed category of options.

```js
const PaymentConfig = {
    payment_gateway_mode: import.meta.env.VITE_PAYMENT_GATEWAY_MODE,
    reserve_lock_timeout_seconds: Number(import.meta.env.VITE_RESERVE_LOCK_TIMEOUT_SECONDS)
};

export default Object.freeze(PaymentConfig);
```

## Rules

- File name is `<subject>.config.js`, kebab-case, singular — same suffix convention as every other
  layer (`GENERAL.md`).
- Default-export a single frozen object (`Object.freeze(...)`), one config file per feature/module.
- The exported object's own name always carries the `Config` suffix (`PaymentConfig`, not
  `Payment`) — every consumer imports and uses it under that exact same name
  (`PaymentConfig.payment_gateway_mode`), never re-imported under a shorter local alias (see
  `GENERAL.md`'s Naming rules).
- Every value must be read from an environment variable (`import.meta.env.*` / `process.env.*`),
  never hardcoded — the file only names and casts/parses the value (e.g. `Number(...)`); the actual
  value lives in `.env`/deployment config.
- No functions or conditional logic inside the file — resolving which value applies happens once,
  centrally, not as scattered `if` checks at call sites.
- Before adding a new config file, check whether the value is actually a constant or enum instead
  (`CONSTANT.md`/`ENUM.md`) — if you can't articulate what would break running the "wrong"
  environment's value, it isn't config.
- Keep config files flat under `config/` — no per-domain subfolders.