# REPOSITORY.md — API Access Layer

A repository is the only layer allowed to make an HTTP request. It has no business logic — every
method just names an endpoint for the shared HTTP client to call. See `GENERAL.md` for how this
layer fits into the overall architecture.

---

## Shape

When the project has a `CrudRepository`, extend it for the standard operations and only add methods
for endpoints it doesn't cover:

```js
// Repositories
import ApiRepository from '@/repositories/api.repository';
import CrudRepository from '@/repositories/crud.repository';

class PropertyRepository extends CrudRepository {
    /**
     * url
     * @returns {string}
     */
    get url() {
        return '/admin/v1/properties';
    }

    /**
     * @param id
     * @param data
     * @param config
     * @returns {Promise}
     */
    restore(id, data, config) {
        return ApiRepository.post(`${ this.url }/${ id }/restore`, data, config);
    }

    /**
     * @param id
     * @param config
     * @returns {Promise}
     */
    getRoomTypes(id, config) {
        return ApiRepository.get(`${ this.url }/${ id }/room-types`, config);
    }
}

export default new PropertyRepository();
```

## Rules

- Extend `CrudRepository` when the project has one and the resource needs any of its standard
  operations (`getAll`, `search`, `getOneById`, `create`, `update`, `delete`) — implement only the
  `url` getter plus whatever endpoint-specific methods `CrudRepository` doesn't cover, rather than
  hand-rolling standard CRUD again.
- File name is `<subject>.repository.js`, kebab-case, singular (`contract.repository.js`,
  `flight-reserve.repository.js`).
- Default-export an **instance**, not the class (`export default new ReserveRepository()`).
- Every method delegates to the shared HTTP client — never `fetch`/`axios` directly, never inline
  URL-building logic beyond simple template-literal interpolation of an id/code into the path.
- Document every method with JSDoc: `@param` for each argument, `@returns`.
- GET methods: `(...pathParams, config)` — always include `config` as the last parameter, even
  when this particular endpoint has no need for custom options.
- POST/PUT/DELETE methods: `(...pathParams, data, config)` — `data` always comes immediately
  before `config`, and both are always present even when the endpoint doesn't conceptually need a
  body (e.g. `cancel(id, data, config)`).
- Never give a parameter a default value in the method signature — not even `config = {}`. A
  missing `data`/`config` is simply forwarded as `undefined` to the HTTP client, never silently
  substituted.
- Never add a separate `params` argument — query params live inside `config` (`config.params`),
  not as a parameter of their own.
- Only pass `config` through to the HTTP client for request options (headers, params). Do not
  invent a separate ad-hoc `options` shape per method.
- Keep versioned endpoints for the same resource in the same repository as it evolves
  (e.g. `tour.repository.js` next to `tour.repository.v2.js`) rather than scattering versions
  across unrelated files.
- Beyond `GENERAL.md`'s Dependency Direction rule, a repository imports only the shared HTTP
  client.

## The Shared HTTP Client

Exactly one shared client resolves the base URL (including any client/server split for SSR) and is
the only place `get`/`post`/`put`/`delete` and middleware (auth, error interception) are defined.
Never instantiate a second client — if a request needs different behavior (a different base URL,
auth scheme), extend the shared client via a middleware or constructor option instead of bypassing
it. Middleware conventions (shape, naming, registration) are project-specific — see that project's
own `PROJECT.md`.