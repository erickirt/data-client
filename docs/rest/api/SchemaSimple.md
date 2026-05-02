---
title: SchemaSimple - Define data processing protocols
sidebar_label: SchemaSimple
description: Build custom schemas for normalization, denormalization, and query keys.
---

# SchemaSimple

`SchemaSimple` lets you teach `@data-client/rest` how to normalize,
denormalize, and query a response shape that is not covered by the built-in
schemas.

Most applications should start with [Entity](./Entity.md),
[Collection](./Collection.md), [Union](./Union.md), [Values](./Values.md), or
[Scalar](./Scalar.md). Reach for a custom schema when the schema needs runtime
logic the built-ins don't provide — for example, denormalized output that
depends on endpoint args, or bounded traversal of cyclic / deep entity graphs.

## Prefer built-ins for shape transformations

Plain objects and arrays are valid schemas; the [Object](./Object.md) shorthand
spreads input and only visits the keys you declare, so unmentioned fields pass
through unchanged. Most response wrappers do not need a custom schema:

```typescript
// Response: { data: { id: '5', name: 'Ada' }, requestId: 'abc123' }
const getUser = new RestEndpoint({
  path: '/users/:id',
  schema: { data: User },
});

// Response: { nextCursor: '...', results: [{...}, {...}] }
const getUsers = new RestEndpoint({
  path: '/users',
  schema: { results: [User] },
});
```

Use [Collection](./Collection.md) when a list needs identity across requests,
[Values](./Values.md) for keyed maps, [Union](./Union.md) for polymorphic
items, [Entity.process()](./Entity.md#process) for response-shape fixups, and
[Entity.indexes](./Entity.md#indexes) for non-pk lookups.

## Usage

The smallest custom schema implements `normalize()` and `denormalize()`. This
example illustrates the interface using a wrapper that preserves response
metadata; in practice you would use `schema: { data: User }` instead — see
[Prefer built-ins](#prefer-built-ins-for-shape-transformations) above.

```typescript
import { Entity, RestEndpoint } from '@data-client/rest';
import type {
  IDenormalizeDelegate,
  INormalizeDelegate,
} from '@data-client/rest';

class User extends Entity {
  id = '';
  name = '';

  static key = 'User';
}

class DataEnvelope {
  constructor(private readonly schema: any) {}

  normalize(
    input: any,
    _parent: any,
    _key: string | undefined,
    delegate: INormalizeDelegate,
  ) {
    return {
      ...input,
      data: delegate.visit(this.schema, input.data, input, 'data'),
    };
  }

  denormalize(input: any, delegate: IDenormalizeDelegate) {
    return {
      ...input,
      data: delegate.unvisit(this.schema, input.data),
    };
  }
}

const getUser = new RestEndpoint({
  path: '/users/:id',
  schema: new DataEnvelope(User),
});
```

`normalize()` replaces the nested `data` with the entity pk and stores the
`User` in the entity table. `denormalize()` runs the reverse, reconstructing the
envelope with `data` as a live `User` instance. See [Lifecycle](#lifecycle) for
the full traversal model.

## Members

Custom schemas typically implement `normalize()` and `denormalize()`.
Implement `queryKey()` when the schema should be readable from the store without
fetching, such as with [useQuery()](/docs/api/useQuery),
[Controller.get](/docs/api/Controller#get), or [Query](./Query.md).

### normalize(input, parent, key, delegate, parentEntity?) {#normalize}

Returns the normalized value to store at this position in the surrounding result.

```typescript
normalize(input, parent, key, delegate, parentEntity?)
```

Use `delegate.visit()` to recurse into nested schemas.

```typescript
class DataEnvelope {
  normalize(
    input: any,
    _parent: any,
    _key: string | undefined,
    delegate: INormalizeDelegate,
  ) {
    return {
      ...input,
      data: delegate.visit(this.schema, input.data, input, 'data'),
    };
  }
}
```

`parentEntity` is the nearest enclosing entity-like schema, when present. Most
custom schemas can ignore it.

### denormalize(input, delegate) {#denormalize}

Receives the normalized value and returns the denormalized value exposed to
hooks, [Controller](/docs/api/Controller), and endpoint resolution.

```typescript
denormalize(input, delegate)
```

Use `delegate.unvisit()` to recurse into nested schemas.

```typescript
class DataEnvelope {
  denormalize(input: any, delegate: IDenormalizeDelegate) {
    return {
      ...input,
      data: delegate.unvisit(this.schema, input.data),
    };
  }
}
```

### queryKey(args, unvisit, delegate) {#queryKey}

Computes the normalized key used to read from the store without fetching.

```typescript
queryKey(args, unvisit, delegate)
```

For wrapper schemas, `queryKey()` usually mirrors the normalized shape returned
by `normalize()`. The recursive `unvisit` argument asks the child schema to build
its own query key from the same endpoint args.

```typescript
class DataEnvelope {
  queryKey(
    args: readonly any[],
    unvisit: (schema: any, args: readonly any[]) => any,
  ) {
    const data = unvisit(this.schema, args);
    return data === undefined ? undefined : { data };
  }
}
```

Return `undefined` when the schema cannot build a valid store key. Return
`delegate.INVALID` when the schema can prove the result should be treated as
invalid.

## Lifecycle

### Normalize

During fetch resolution, the schema tree is walked from the endpoint's
`schema`. Each custom schema receives the raw value at its position and returns
the normalized value to place in the endpoint result.

```typescript
// Response
{ data: { id: '5', name: 'Ada' }, requestId: 'abc123' }

// Normalized endpoint result
{ data: '5', requestId: 'abc123' }
```

Nested [Entity](./Entity.md), [Collection](./Collection.md), [Union](./Union.md),
and other schemas should be reached through `delegate.visit()`, not by calling
their methods directly.

### Denormalize

When cached data is read, custom schemas receive the normalized input they
previously returned from `normalize()`.

```typescript
denormalize(input: any, delegate: IDenormalizeDelegate) {
  return {
    ...input,
    data: delegate.unvisit(this.schema, input.data),
  };
}
```

If denormalized output changes based on endpoint args, use
[`delegate.argsKey()`](#argsKey) so memoization tracks that dependency.

### Query Key

`queryKey()` runs when the schema is queried without a network response. This is
how `useQuery()`, `Controller.get()`, and `Query` discover the normalized input
that should be denormalized.

```typescript
queryKey(args, unvisit) {
  const pk = unvisit(User, args);
  return pk === undefined ? undefined : { data: pk };
}
```

## Delegate Interfaces

### INormalizeDelegate {#inormalizedelegate}

Passed to [`normalize()`](#normalize). Use this to recurse into nested schemas,
read endpoint args, and write entity-like results.

#### visit(schema, value, parent, key) {#visit}

Recursively normalizes `value` with another schema.

```typescript
delegate.visit(schema, value, parent, key)
```

`Array` uses `visit()` to normalize each item against the inner schema:

```typescript
return values.map(value => delegate.visit(schema, value, parent, key));
```

`EntityMixin` uses it to normalize each declared field on an entity:

```typescript
for (const key of Object.keys(this.schema)) {
  processedEntity[key] = delegate.visit(
    this.schema[key],
    processedEntity[key],
    processedEntity,
    key,
  );
}
```

#### args {#normalize-args}

Endpoint args for the current normalize operation.

```typescript
delegate.args
```

`EntityMixin.normalize()` reads `args` to compute the entity's primary key from
the incoming response and the original endpoint call:

```typescript
const args = delegate.args;
const processedEntity = this.process(input, parent, key, args);
const id = this.pk(processedEntity, parent, key, args);
```

#### meta {#meta}

Fetch metadata for the current normalize operation
(`fetchedAt`, `date`, `expiresAt`).

```typescript
delegate.meta
```

`Entity` merge lifecycles use this to decide whether an incoming row is newer
than what is in the store. Custom schemas usually do not need to read it.

#### mergeEntity(schema, pk, incomingEntity) {#mergeEntity}

Writes an entity through its merge lifecycle.

```typescript
delegate.mergeEntity(schema, pk, incomingEntity)
```

This is the final step of `EntityMixin.normalize()` after recursing into each
field:

```typescript
delegate.mergeEntity(this, id, processedEntity);
return id;
```

Most custom schemas should delegate entity work to `Entity`, `Collection`, or
another nested schema. Use `mergeEntity()` only when implementing entity-like
behavior yourself.

#### setEntity(schema, pk, entity, meta?) {#setEntity}

Writes an entity row by replacing the previous normalized value, skipping the
merge lifecycle that `mergeEntity()` runs.

```typescript
delegate.setEntity(schema, pk, entity, meta)
```

Use this when the incoming row should overwrite prior normalized data rather
than merge with it. `delegate.invalidate()` is implemented as a `setEntity()`
write of an `INVALID` symbol.

#### invalidate(schema, pk) {#invalidate}

Marks an entity result invalid, triggering Suspense for endpoints that need it.

```typescript
delegate.invalidate(schema, pk)
```

This is what the `Invalidate` schema does after computing the target entity's
pk:

```typescript
const processedEntity = entitySchema.process(input, parent, key, args);
const pk = `${entitySchema.pk(processedEntity, parent, key, args)}`;

delegate.invalidate(entitySchema, pk);
```

#### checkLoop(key, pk, input) {#checkLoop}

Returns `true` when `(entityKey, pk, input)` is being normalized again inside
its own subtree, so the schema should stop recursing.

```typescript
delegate.checkLoop(entityKey, pk, input)
```

`EntityMixin.normalize()` short-circuits before walking nested fields when a
cycle is detected:

```typescript
if (delegate.checkLoop(this.key, id, input)) return id;
```

### IDenormalizeDelegate {#idenormalizedelegate}

Passed to [`denormalize()`](#denormalize). Use this to recurse into nested
schemas and register args-dependent memoization.

#### unvisit(schema, input) {#unvisit}

Recursively denormalizes `input` with another schema.

```typescript
delegate.unvisit(schema, input)
```

`Array.denormalize()` uses `unvisit()` to denormalize each item with the inner
schema:

```typescript
return input.map(entityOrId => delegate.unvisit(schema, entityOrId));
```

`EntityMixin.denormalize()` uses it once per declared field, propagating
`INVALID` symbols when a required nested entity is missing:

```typescript
for (const key of Object.keys(this.schema)) {
  const value = delegate.unvisit(this.schema[key], input[key]);
  // ...
}
```

#### args {#denormalize-args}

Endpoint args for the current denormalize operation.

```typescript
delegate.args
```

`Query.denormalize()` reads `args` to forward them into a user-supplied
processor:

```typescript
const value = delegate.unvisit(this.schema, input);
return this.process(value, ...delegate.args);
```

Reading `args` directly does not contribute to cache invalidation. If
denormalized output changes based on endpoint args, use
[`argsKey()`](#argsKey) instead.

#### argsKey(fn) {#argsKey}

Registers an args-derived memoization key while denormalizing.

```typescript
delegate.argsKey(fn)
```

The function reference must be stable. Define it at module scope or bind it on
the schema instance.

`Scalar.denormalize()` uses a constructor-bound `lensSelector` to register the
current lens (e.g. `portfolio`, `currency`, `locale`) as a memoization
dimension, then looks up the matching cell:

```typescript
const lensValue = delegate.argsKey(this.lensSelector);
if (lensValue === undefined) return undefined;
const cellData = delegate.unvisit(
  this,
  `${input[2]}|${input[0]}|${lensValue}`,
);
```

### IQueryDelegate {#iquerydelegate}

Passed to [`queryKey()`](#queryKey). Use this to check whether store data exists
before returning a normalized key.

#### getEntity(key, pk) {#getEntity}

Reads one normalized entity row from the store.

```typescript
delegate.getEntity(entityKey, pk)
```

`EntityMixin.queryKey()` uses it to avoid returning a key for an entity that
the store does not currently have:

```typescript
if (!args[0]) return;
const pk = queryKeyCandidate(this, args, delegate);
if (pk && delegate.getEntity(this.key, pk)) return pk;
```

`Collection.queryKey()` does the same after computing the collection's pk from
endpoint args:

```typescript
const pk = this.pk(undefined, undefined, '', args);
if (delegate.getEntity(this.key, pk)) return pk;
```

#### getEntities(key) {#getEntities}

Reads all normalized rows for an entity key as an iterable view.

```typescript
delegate.getEntities(entityKey)
```

The single-schema branch of `All.queryKey()` returns every cached pk for its
entity:

```typescript
const entities = delegate.getEntities(this.schema.key);
if (!entities) return delegate.INVALID;
return [...entities.keys()];
```

#### getIndex(key, index, value) {#getIndex}

Looks up a primary key by an [Entity index](./Entity.md#indexes).

```typescript
delegate.getIndex(entityKey, indexName, value)
```

`EntityMixin.queryKey()` falls back to an index lookup when the first arg
matches an indexed field rather than a pk:

```typescript
const field = indexFromParams(args[0], schema.indexes);
if (!field) return;
const value = args[0][field];
return delegate.getIndex(schema.key, field, value);
```

#### INVALID {#invalid}

Sentinel returned when a query result should be treated as invalid.

```typescript
return delegate.INVALID;
```

`All.queryKey()` returns `INVALID` when no rows for the requested entity have
ever been cached, so the consumer knows to fetch rather than treat the empty
list as a hit:

```typescript
if (!entities) return delegate.INVALID;
```

Return `undefined` when the schema simply cannot build a store key yet. Return
`delegate.INVALID` when it can build the key but knows the cached result is
invalid.

## Entity-Like Schemas

Normalizr treats a schema as entity-like when it has a `pk` property.
Entity-like schemas are stored by `key` and primary key, denormalized through
entity caches, and tracked for cycle detection.

The minimal shape is:

```typescript
{
  key: string;
  pk(input, parent, key, args);
}
```

Additional members like `createIfValid`, `schema`, `indexes`, `cacheWith`, and
`maxEntityDepth` are used by [Entity](./Entity.md) and related schemas.
Prefer extending `Entity` unless you need a different entity protocol entirely.

```typescript
class UsernameEntity {
  static key = 'User';
  static indexes = ['username'];

  static pk(input: { id?: string }) {
    return input.id;
  }
}
```

`cacheWith` lets multiple schema instances share the same entity cache identity.
`maxEntityDepth` limits recursive denormalization depth for very deep entity
graphs.

## Examples

### Argument-Dependent Fields

If denormalized output changes based on endpoint args, register that dependency
with `delegate.argsKey()`. Reading `delegate.args` directly works for the current
call, but it does not give the memoized denormalization cache enough information
to invalidate when the relevant arg changes.

Define the selector at module scope or bind it once on the schema instance. The
function reference is part of the cache path.

```typescript
const localeKey = (args: readonly any[]) => args[0]?.locale;

class LocalizedText {
  normalize(input: Record<string, string>) {
    return input;
  }

  denormalize(
    input: Record<string, string>,
    delegate: IDenormalizeDelegate,
  ) {
    const locale = delegate.argsKey(localeKey) ?? 'en';
    return input[locale] ?? input.en;
  }

  queryKey() {
    return undefined;
  }
}

class Product extends Entity {
  id = '';
  name = '';

  static key = 'Product';
  static schema = {
    name: new LocalizedText(),
  };
}

const getProduct = new RestEndpoint({
  path: '/products/:id',
  searchParams: {} as { locale?: string },
  schema: Product,
});
```

With this schema, the normalized `Product.name` can store the full locale map,
while components receive the string for the current `locale` arg.

### Depth-limited Relationships

Large bidirectional graphs (parent/children, or `Department ↔ Building ↔ Room`)
can blow up eager denormalization. [Lazy](./Lazy.md) plus
[useQuery](/docs/api/useQuery) is the recommended fix because it keeps
re-render boundaries tight, but a custom schema can also bound traversal
in-place — useful as an interim while migrating, or for self-referential
hierarchies where you do want N levels resolved transparently.

The pattern uses the fact that one `delegate` object is reused for an entire
denormalize tree, so a `WeakMap<delegate, state>` gives per-call state that is
GC'd automatically.

`DepthLimited` caps how many levels of a specific relationship resolve. Once
the limit is hit, it returns the raw normalized value (the pks) instead of
recursing further:

```typescript
import type {
  IDenormalizeDelegate,
  INormalizeDelegate,
  Schema,
} from '@data-client/rest';

class DepthLimited<S extends Schema> {
  readonly schema: S;
  readonly maxDepth: number;
  private readonly _state = new WeakMap<
    IDenormalizeDelegate,
    { depth: number }
  >();

  constructor(schema: S, maxDepth: number) {
    this.schema = schema;
    this.maxDepth = maxDepth;
  }

  normalize(input: any, parent: any, key: any, delegate: INormalizeDelegate) {
    return delegate.visit(this.schema, input, parent, key);
  }

  denormalize(input: {}, delegate: IDenormalizeDelegate) {
    if (input == null || typeof input === 'symbol') return input;
    let cell = this._state.get(delegate);
    if (!cell) {
      cell = { depth: 0 };
      this._state.set(delegate, cell);
    }
    cell.depth++;
    try {
      if (cell.depth > this.maxDepth) return input;
      return delegate.unvisit(this.schema, input);
    } finally {
      cell.depth--;
    }
  }

  queryKey(): undefined {
    return undefined;
  }
}
```

```typescript
class Department extends Entity {
  static schema = {
    children: new DepthLimited([Department], 3),
    parent: new DepthLimited(Department, 1),
  };
}
```

`CycleDetect` stops as soon as the same entity type appears twice on the
ancestor path, so it adapts to any schema shape without tuning a depth number.
It pairs with an `Entity` subclass that records the ancestor stack:

```typescript
const _ancestors = new WeakMap<IDenormalizeDelegate, Map<string, number>>();

function getAncestors(delegate: IDenormalizeDelegate) {
  let m = _ancestors.get(delegate);
  if (!m) {
    m = new Map();
    _ancestors.set(delegate, m);
  }
  return m;
}

function extractEntityKeys(schema: any, out = new Set<string>()) {
  if (schema == null) return out;
  if (schema.pk !== undefined && schema.key) {
    out.add(schema.key);
    return out;
  }
  if (Array.isArray(schema) && schema.length === 1) {
    return extractEntityKeys(schema[0], out);
  }
  if (schema.schema !== undefined) extractEntityKeys(schema.schema, out);
  return out;
}

class CycleDetect<S extends Schema> {
  constructor(readonly schema: S) {}

  normalize(input: any, parent: any, key: any, delegate: INormalizeDelegate) {
    return delegate.visit(this.schema, input, parent, key);
  }

  denormalize(input: {}, delegate: IDenormalizeDelegate) {
    if (input == null || typeof input === 'symbol') return input;
    const path = getAncestors(delegate);
    for (const key of extractEntityKeys(this.schema)) {
      if (path.has(key)) return input;
    }
    return delegate.unvisit(this.schema, input);
  }

  queryKey(): undefined {
    return undefined;
  }
}

class CycleTrackingEntity extends Entity {
  static denormalize<T extends typeof CycleTrackingEntity>(
    this: T,
    input: any,
    delegate: IDenormalizeDelegate,
  ): any {
    if (typeof input === 'symbol') return input;
    const path = getAncestors(delegate);
    const k = this.key;
    const prev = path.get(k) ?? 0;
    path.set(k, prev + 1);
    try {
      return super.denormalize(input, delegate);
    } finally {
      if (prev === 0) path.delete(k);
      else path.set(k, prev);
    }
  }
}
```

```typescript
class Department extends CycleTrackingEntity {
  static schema = {
    buildings: new CycleDetect([Building]),
    children: new DepthLimited([Department], 3),
    parent: new DepthLimited(Department, 1),
  };
}

class Building extends CycleTrackingEntity {
  static schema = {
    departments: new CycleDetect([Department]),
  };
}
```

Neither wrapper has a `pk`, so they sit outside the entity hot path; consumers
who do not use them pay nothing. At the limit, they return raw normalized
values (the same shape `Lazy` produces). See discussion
[#3828](https://github.com/reactive/data-client/discussions/3828#discussioncomment-16456893)
for the original write-up and tradeoffs versus `Lazy` + `useQuery`.

## Related

- [Thinking in Schemas](./schema.md)
- [Entity](./Entity.md)
- [Collection](./Collection.md)
- [Scalar](./Scalar.md)
