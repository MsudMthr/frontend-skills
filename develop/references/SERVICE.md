# SERVICE.md — Business Logic Layer

A service holds business logic and calculations shared between multiple files. It never makes an
HTTP request itself (that's `REPOSITORY.md`) and never touches Vue reactivity (that's
`STORE.md`/`COMPOSABLE.md`). See `GENERAL.md` for how this layer fits into the overall
architecture.

---

## Two Shapes, Pick By State

### Singleton — the default

No constructor, no per-usage variation. Instance methods, default-exported as one already-`new`-ed
instance — the same convention as a repository (`REPOSITORY.md`). This is the right shape for
almost every service: pure calculations, classification/predicate helpers, and formatting derived
from data already fetched elsewhere.

```js
// Services
import { t } from '@/services/language.service';

// Utils
import JalaliDate from '@/utils/jalali-date';

class PropertyContractService {
    /**
     *
     * @param {Object} contract
     * @returns {boolean}
     */
    isDisabled(contract) {
        if (contract.cancelled_at !== null) {
            return true;
        }

        if (contract.deleted_at !== null) {
            return true;
        }

        const contractEndDate = JalaliDate.make().gregorian().parse('yyyy-MM-dd', contract.end_date);

        return contractEndDate.isPastInDay();
    }

    /**
     * @param {Object} contract
     * @returns {boolean}
     */
    isActive(contract) {
        return this.isDisabled(contract) === false;
    }

    /**
     * @param {Object} contract
     * @returns {String}
     */
    getStatusLabel(contract) {
        if (contract.cancelled_at !== null) {
            return t('canceled');
        }

        if (contract.deleted_at !== null) {
            return t('deleted');
        }

        const contractEndDate = JalaliDate.make().gregorian().parse('yyyy-MM-dd', contract.end_date);

        return contractEndDate.isPastInDay() ? t('Expired') : t('Current');
    }
}

export default new PropertyContractService();
```

Usage: `propertyContractService.isDisabled(contract)` — the module already returns the one shared
instance, so never `new PropertyContractService()` again elsewhere.

A singleton service may still hold shared cache/config via ordinary instance fields (`_payload`,
`_messages`) — a token service caching a decoded payload, or a language service caching loaded
messages — since the module only ever exports one instance, every caller already sees the same
value without needing `static`.

### Per-Usage Instance — only when state genuinely varies by usage

Give it a `constructor()`, private fields (`_underscorePrefixed`), and instance methods. Export
the class (not an instance); the *caller* decides when to `new` it, because the state is tied to
that one usage (a specific tracking context, a specific storage namespace) — not shared globally.

```js
class StorageService {
    _name;
    _driver;
    _data;

    constructor(name, driver = LocalStorageService) {
        this._name = name;
        this._driver = driver;
        this._data = this._initialize();
    }

    setItem(key, value) {
        this._data[key] = value;
        this._save();
    }
}

export default StorageService;
```

```js
// caller — e.g. another service or a store
this._storage = new StorageService('ecommerce-property');
```

Per-usage services can extend a shared base class to reuse constructor/state logic (e.g. a
per-domain tracking service extending a shared base tracking service). Private state and helper
methods use a leading underscore (`_context`, `_send()`, `_getContextData()`).

Only reach for this shape when different callers genuinely need different state at the same time
(two storage namespaces open together, two tracking contexts side by side) — if the whole app only
ever needs one instance, that's the singleton shape above, not this one.

---

## Rules

- One class per file. File name is `<subject>.service.js`, kebab-case, singular
  (`facility.service.js`, `property-reserve-form.service.js`).
- A method takes the plain data object it operates on as its first argument (`isDisabled(contract)`,
  `isActive(promotion)`) rather than reaching into a store. Services are decoupled from Vue/Pinia
  so they can be unit-tested with a plain object literal.
- Keep every service file flat under `services/` — don't create a per-domain subfolder
  (`services/ecommerce/`, `services/flight/`) to group a family of related services; the file
  name alone is enough (`ecommerce-property.service.js`, `flight.service.js`). Services are
  entity-based, not domain-based: one entity gets one service file, and everything relevant to
  that entity (`rate-plan.service.js` for rate plans, `facility.service.js` for facilities) is
  handled inside it. That's what keeps the layer flat.
- Document public methods with JSDoc (`@param`, `@returns`).
- Beyond `GENERAL.md`'s Dependency Direction rule, a service may also import enums, constants, and
  the translator function.
- Before adding a new method, check whether the calculation already exists on a related service —
  this is the layer most likely to accumulate duplicate one-off predicates.