# Design Choices

This document records non-obvious design decisions in `@data-client/rest` and the
trade-offs behind them. It is meant for contributors deciding whether to change a
behavior — not for end users (those should use the [docs](https://dataclient.io/rest)).

## `Resource.create` does not track `getList` overrides

### Symptom

Given a resource whose `getList` is customized via `extend`:

```ts
const MyResource = resource({ path: '/todos/:id', schema: Todo }).extend(
  Base => ({
    getList: Base.getList.extend({
      path: '/todos/:user/',
      body: {} as WeirdBody,
    }),
  }),
);

MyResource.create({ user: 'bob' }, { title: 'get it done' } as WeirdBody);
```

`MyResource.create`:

- **Runtime**: still POSTs to the original `getList` path (the shortened form of
  `/todos/:id`), not `/todos/:user/`.
- **Types**: still requires the original `Partial<Denormalize<Todo>>` body and the
  original path args, ignoring `WeirdBody` and `:user`.

### Root cause

`create` is materialized eagerly at `resource()` construction time and is never
re-derived when `getList` is replaced.

```ts
// packages/rest/src/resource.ts
const ret = {
  get,
  getList,
  // TODO(deprecated): remove this once we remove creates
  create: getList.push.extend({ name: getName('create') }),
  ...
};
```

The type definition mirrors this: `create` is computed from the original
`ResourceGenerics` `O`, not from `R['getList']`.

```ts
// packages/rest/src/resourceTypes.ts
/** @deprecated use Resource.getList.push instead */
create: 'searchParams' extends keyof O ?
  MutateEndpoint<{
    path: ShortenPath<O['path']>;
    schema: schema.Collection<[O['schema']]>['push'];
    body: 'body' extends keyof O ? O['body']
    : Partial<Denormalize<O['schema']>>;
    searchParams: O['searchParams'];
  }>
  : ...;
```

All three forms of `extend` (`string`, object-options, and function callback)
produce result types that preserve `create` from the original `R` and only swap
in the keys explicitly overridden:

```ts
// packages/rest/src/resourceExtensionTypes.ts
export type ExtendedResource<
  R extends ResourceInterface,
  T extends Record<string, EndpointInterface>,
> = Omit<R, keyof T> & T;
```

### Why we don't fix it

`create` is `@deprecated` in favor of `getList.push`, and the source carries a
`TODO(deprecated): remove this once we remove creates`. `getList.push` already
has the desired behavior for free because it lives on the `RestInstance`
itself — when `getList` is extended, its `push` method's types are derived
from the new instance:

```ts
// packages/rest/src/RestEndpointTypes.ts (RestInstance)
push: AddEndpoint<
  F,
  ExtractCollection<S>['push'],
  Omit<O, 'body' | 'method'> & {
    body:
      | OptionsToAdderBodyArgument<O, ExtractCollection<S>['push']>
      | OptionsToAdderBodyArgument<O, ExtractCollection<S>['push']>[]
      | FormData;
  }
>;
```

So `MyResource.getList.push({ user: 'bob' }, { title: 'get it done' } as WeirdBody)`
works correctly with no library changes.

### What a fix would look like (if we ever decide to do it)

For completeness, the patch would touch four places. None of them is large; the
reason it hasn't been done is value-to-cost relative to a method already slated
for removal.

**Runtime** (`packages/rest/src/resource.ts`): re-derive `create` in each
`extend` branch when `getList` was overridden and `create` was not. Example for
the function-form branch:

```ts
} else if (typeof args[0] === 'function') {
  const extended = args[0](this);
  const merged = { ...this, ...extended };
  if ('getList' in extended && !('create' in extended)) {
    merged.create = merged.getList.push.extend({ name: getName('create') });
  }
  return merged;
}
```

A `defineProperty` getter on the original `ret` is tempting but does not
survive the `{ ...this }` spreads used throughout `extend`, so explicit
re-binding is simpler.

**Types** (three locations):

1. `ExtendedResource<R, T>` — when `'getList' extends keyof T`, intersect in
   `{ create: T['getList']['push'] }` and add `'create'` to the `Omit` from `R`.
2. `CustomResource<R, O, Get, GetList, ...>` — when `GetList` is non-default,
   re-derive `create` from the rewritten `getList`.
3. `ResourceExtension<R, ExtendKey, O>` — when `ExtendKey extends 'getList'`,
   also rewrite `create`.

`Resource<O>` itself does not need to change; it is only the extend results
that mis-track today.

A cleaner-looking attempt using polymorphic `this` (`create: this['getList']['push']`)
does **not** work — TypeScript's `this` type is preserved through method
signatures, but in property type positions the lookup re-resolves to the
declaring interface, not the intersected result of `Omit<R, keyof T> & T`.
Explicit re-derivation in each extend result type is the only working approach.

### Edge case: user-supplied `create`

Any fix must not clobber a user-provided `create`:

```ts
resource(...).extend(Base => ({
  getList: Base.getList.extend({ path: '/todos/:user/' }),
  create: someCustomEndpoint, // do not overwrite this
}));
```

The runtime check is `'getList' in extended && !('create' in extended)`. The
type-level equivalent is conditioning on `'create' extends keyof T` before
re-deriving.

### Type-checking performance considerations

The proposed type changes short-circuit cleanly on the paths that don't use the
scenario:

- Base `Resource<O>` is unchanged — every `resource(...)` call costs the same.
- `.extend(...)` calls that don't touch `getList` add one short-circuiting
  conditional (`'getList' extends keyof T ? ... : {}`).
- `.extend(...)` calls that override `getList` evaluate
  `RestExtendedEndpoint<...>['push']`, but anyone who reads
  `MyResource.getList.push` already pays this cost; TypeScript caches indexed
  access on a given type.

The most user-visible cost would be in editor hover and error rendering, since
`Resource` and `CustomResource` already produce verbose tooltips. Anyone
attempting this change should compare `tsc --noEmit --extendedDiagnostics`
(or `--generateTrace`) before and after on a representative consumer project.
Anything under 5% on `Check time` / `Instantiations` is in the noise; over 15%
is a regression worth pushing back on.

### Recommendation

Use `MyResource.getList.push(...)` instead of `MyResource.create(...)`. If
`create` is ever revived as a non-deprecated API, prefer removing the eager
materialization and adding the four type-level re-derivations above, rather
than special-casing the override scenarios.
