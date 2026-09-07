# COMPOSABLE.md — Cross-Cutting Vue Logic

A composable is reusable, Vue-specific reactive logic shared across multiple stores/components/
views — the Vue equivalent of a util (`UTIL.md`), but one that is allowed to use
`ref`/`computed`/lifecycle hooks. Framework-independent logic that doesn't touch Vue reactivity
belongs in `UTIL.md` instead. As noted in `GENERAL.md`, composables are a cross-cutting support
layer, not a pipeline stage — any layer (store, component, another composable) may use one.

When a domain/feature has Vue-reactive logic in common across multiple of its own
components/stores/views — not just plain calculations — that shared logic belongs in a composable
dedicated to that domain (e.g. `property.composable.js`), not in the domain's service. A service
is framework-independent (`SERVICE.md`) and can't hold `ref`/`computed`/lifecycle hooks, so any
reactive logic a domain needs to share has nowhere to live but a composable.

---

## Shape

```js
import { ref, computed } from 'vue';

export function useLoading() {
    const loading = ref(0);
    const isLoading = computed(() => loading.value > 0);

    function startLoading() {
        loading.value++;
    }

    function endLoading() {
        loading.value--;
    }

    function startFakeLoading(delay = 500) {
        startLoading();

        return new Promise(function (resolve) {
            setTimeout(function () {
                endLoading();
                resolve(true);
            }, delay);
        });
    }

    return {
        isLoading,
        startLoading,
        endLoading,
        startFakeLoading
    };
}
```

## Rules

- File name is `<subject>.composable.js`; the exported function is `useX` and is a **named**
  export (a composable file may export more than one related hook, e.g. both a v1 and v2 variant
  of the same concern).
- Keep every composable file flat under `composables/` — don't create a per-domain subfolder
  (`composables/ecommerce/`, `composables/flight/`, `composables/property/`) to group a family of
  them; the file name alone is enough (`ecommerce-property.composable.js`,
  `flight-reserve-details.composable.js`).
- Model concurrency-sensitive state with a counter instead of a boolean when overlapping
  starts/stops must be supported (nested loading regions that shouldn't flicker `false` between
  overlapping requests) — an incrementing/decrementing ref, not a boolean toggle.
- Always clean up timers/listeners/abort controllers registered by a composable in
  `onBeforeUnmount`. A composable that starts an interval/listener and never tears it down is a
  bug, not a shortcut.