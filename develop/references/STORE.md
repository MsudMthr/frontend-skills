# STORE.md — State Layer (Pinia)

A store holds Pinia state wherever the same data needs to be shared between views,
components, composables, or other stores. Every view has its own dedicated store holding the
reactive state for that page/feature; when a component/view consumes its own store, name the
instance plainly `store` (`const store = usePropertyLandingFreeCancellationStore();`). Other
stores are borrowed only when actually needed (e.g. a shared user store, a global modal store) and
are given their own instance variable named after that store's own convention instead
(`const usersStore = useUsersStore();`) — don't rename a borrowed store's variable to match the
local file's namespace. See `GENERAL.md` for how this layer fits into the overall architecture.

---

## Shape

Use Pinia's **setup-style** store (a function returning refs/computed/functions), not the
options-style `{ state, getters, actions }` object. This mirrors Composition-API components and
lets a store use composables directly.

```js
import { defineStore } from 'pinia';

// Vue
import { computed, reactive, ref } from 'vue';

// Composables
import { useLoading } from '@/web/composables/loading.composable.js';

// Repositories
import PropertyRepository from '@/web/repositories/property.repository.js';

// Enums
import SortTypeEnum from '@/web/enums/sort-type.enum.js';

export const usePropertyLandingFreeCancellationStore = defineStore('property-landing-free-cancellation', function() {
    const { startLoading, endLoading, isLoading } = useLoading();

    const items = ref([]);
    const sortType = ref(SortTypeEnum.DEFAULT);

    const categoryProperties = computed(function() {
        // derived list
    });

    async function fetchCityProperties(city) {
        startLoading();

        try {
            const response = await PropertyRepository.getByFreeCancellationPromotion(city.id);
            items.value = response.data.value.properties;

            return response;
        } finally {
            endLoading();
        }
    }

    return {
        isLoading,
        items,
        categoryProperties,
        sortType,
        fetchCityProperties
    };
});
```

## Rules

- File name is `<subject>.store.js`, kebab-case, singular; the exported composable is
  `useXStore`; the Pinia store id string passed to `defineStore(...)` matches the filename's
  subject.
- Keep every store file flat under `stores/` — don't create a per-domain subfolder
  (`stores/property/`, `stores/property/reserve/`, `stores/flight/`, `stores/user/`) to group a
  family of them; the file name alone is enough (`property-reserve-form.store.js`,
  `flight-search.store.js`).
- Internal ordering inside the setup function follows `GENERAL.md`'s general script-ordering
  principle.
- For a store with several distinct concerns (loading, reserve data, payment, polling), separate
  each concern with a banner comment so the file stays scannable:

  ```js
  /**
   * ------------------------------------------------------------------------
   * Reserve Polling
   * ------------------------------------------------------------------------
   */
  ```

  Skip this for a small, single-concern store — banner comments are for scanability in a large,
  multi-concern file, not decoration for something already short.
- Wrap every repository call in `try { ... } finally { startLoading(); ... endLoading(); }` via
  the `useLoading` composable (`COMPOSABLE.md`) — never an ad-hoc `ref(false)` loading flag. An
  empty `finally {}` with no `catch` is expected: errors are handled globally (`GENERAL.md`); the
  `finally` only exists to guarantee `endLoading()` runs.
- A fetch function must `return` the repository's response (`return response;`), even though it
  also assigns the relevant piece of it into reactive state — callers that need the raw response
  (e.g. to read pagination metadata) shouldn't have to re-fetch just because the store already
  extracted its own slice of it.
- A store may call another store's actions/state directly (`otherStore.show(...)`,
  `userStore.isLoggedIn`) and may `watch()` another store's state to react to it.
- A store may instantiate a stateful service inline when the action needs one
  (`new EcommercePropertyService({...})`), per `SERVICE.md`'s instantiable-service pattern.
- Do not put presentation logic (formatting for display, DOM measurements) in a store — that
  belongs in the component/view or a util/filter. A store's computed values should still be
  domain-shaped data, not markup-ready strings, unless every consumer needs the exact same
  formatting.