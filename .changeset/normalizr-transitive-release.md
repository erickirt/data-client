---
'@data-client/core': patch
'@data-client/react': patch
'@data-client/vue': patch
---

Include `@data-client/normalizr@0.16.6` performance improvements:

- [#3875](https://github.com/reactive/data-client/pull/3875) [`467a5f6`](https://github.com/reactive/data-client/commit/467a5f6f9d4cdaf0927fa7e22520c5d2c1462ff5) - Fix deepClone to only copy own properties

  `deepClone` in the immutable store path now uses `Object.keys()` instead of `for...in`, preventing inherited properties from being copied into cloned state.

- [#3877](https://github.com/reactive/data-client/pull/3877) [`e9e96f1`](https://github.com/reactive/data-client/commit/e9e96f1751895c17e046461a1c38bb4bb093c141) - Replace megamorphic computed dispatch in getDependency with switch

  `getDependency` used `delegate[array[index]](...spread)` which creates a temporary array, a computed property lookup, and a spread call on every invocation — a megamorphic pattern that prevents V8 from inlining or type-specializing the call site. Replaced with a `switch` on `path.length` for monomorphic dispatch.

- [#3876](https://github.com/reactive/data-client/pull/3876) [`7d28629`](https://github.com/reactive/data-client/commit/7d28629d07f6cade43e36f3cf1956f175f98d84f) - Improve denormalization performance by pre-allocating the dependency tracking slot

  Replace `Array.prototype.unshift()` in `GlobalCache.getResults()` with a pre-allocated slot at index 0, avoiding O(n) element shifting on every cache-miss denormalization.

- [#3884](https://github.com/reactive/data-client/pull/3884) [`7df6a49`](https://github.com/reactive/data-client/commit/7df6a49ee9fcdac10f9f24ec48c4df0931efa0b0) - Move entity table POJO clone from getNewEntities to setEntity

  Lazy-clone entity and meta tables on first write per entity type instead of eagerly in getNewEntities. This keeps getNewEntities as a pure Map operation, eliminating its V8 Maglev bailout ("Insufficient type feedback for generic named access" on `this.entities`).

- [#3878](https://github.com/reactive/data-client/pull/3878) [`98a7831`](https://github.com/reactive/data-client/commit/98a78318770feaa8433708693bec90b81cbcb1b2) - Avoid hidden class mutation in normalize() return object

  The normalize result object was constructed with `result: '' as any` then mutated via `ret.result = visit(...)`, causing a V8 hidden class transition when the property type changed from string to the actual result type. Restructured to compute the result first and construct the final object in a single step.
