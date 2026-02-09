# object-graph-delta

> Structural diffs for object graphs with cycle safety and configurable array semantics.

`object-graph-delta` computes deterministic `CREATE` / `CHANGE` / `REMOVE` events between two object/array graphs.
It is designed for scenarios where you need path-based change events instead of just boolean equality.

## Features

- **Deterministic diff events:** Emits `REMOVE` and `CHANGE` first, then `CREATE`, in stable key/index order.
- **Operational array policies:** Choose between `diff`, `atomic`, and `ignore` array handling.
- **Cycle-safe traversal:** Optional pair-based circular reference tracking to prevent infinite recursion.
- **Rich-type equality:** Correct value comparison for `Date`, `RegExp`, and boxed primitives.
- **Object key filtering:** Skip specific object keys via `keysToSkip` without affecting array indices.
- **TypeScript-friendly API:** Strongly typed paths and diff result unions.

## Why Use It?

| Need | What this package gives you |
| --- | --- |
| Patch-like change events | `CREATE` / `CHANGE` / `REMOVE` with explicit paths |
| Fine control over array behavior | Index diffing, atomic compare, or full ignore |
| Safe comparison of cyclic graphs | Built-in cycle tracking (`trackCircularReferences`) |
| Correct treatment of tricky JS values | `Object.is` semantics, rich-type handling, boxed primitive equality |

## Diff Model

Each diff entry includes a `path` (`(string | number)[]`) and one of:

- `CREATE`: `{ type: 'CREATE', path, value }`
- `REMOVE`: `{ type: 'REMOVE', path, oldValue }`
- `CHANGE`: `{ type: 'CHANGE', path, value, oldValue }`

Example path:

- `['users', 0, 'name']` means `root.users[0].name`

## Installation

```bash
npm install object-graph-delta
# or
pnpm add object-graph-delta
# or
yarn add object-graph-delta
```

## Quick Start

```ts
import { diff } from 'object-graph-delta';

const previous = { a: 1, b: 2 };
const current = { b: 3, c: 4 };

const changes = diff(previous, current);
console.log(changes);
```

Output:

```ts
[
  { type: 'REMOVE', path: ['a'], oldValue: 1 },
  { type: 'CHANGE', path: ['b'], value: 3, oldValue: 2 },
  { type: 'CREATE', path: ['c'], value: 4 }
]
```

## Operational Policies

Array handling is the core runtime policy switch.

### Array Policy Matrix

| `arrayPolicy` | Behavior | Emits |
| --- | --- | --- |
| `'diff'` | Traverses arrays by index recursively | Per-index `CREATE` / `CHANGE` / `REMOVE` |
| `'atomic'` | Treats each array as one leaf value | At most one `CHANGE` at the array path |
| `'ignore'` | Skips array comparison entirely | Nothing for arrays |

### Atomic Equality Modes

Used only when `arrayPolicy: 'atomic'`:

| `arrayEquality` | Equality rule |
| --- | --- |
| `'reference'` | Equal only if `a === b` |
| `'shallow'` | Equal if same length and each element matches via `Object.is` |

### Default Policy

Default options are:

```ts
{
  trackCircularReferences: true,
  arrayPolicy: 'diff',
  keysToSkip: []
}
```

When `arrayPolicy` is `'atomic'` and `arrayEquality` is omitted, `'shallow'` is used.

`arrayEquality` is only relevant in `atomic` mode. Set it explicitly when using `arrayPolicy: 'atomic'`.

## Examples (Simple to Complex)

### 1. Basic Value Change

```ts
import { diff } from 'object-graph-delta';

diff({ a: 1 }, { a: 2 });
// [{ type: 'CHANGE', path: ['a'], value: 2, oldValue: 1 }]
```

### 2. Nested Array Diff (Default `arrayPolicy: 'diff'`)

```ts
import { diff } from 'object-graph-delta';

diff({ list: [1, 2, 3] }, { list: [1, 3] });
// [
//   { type: 'CHANGE', path: ['list', 1], value: 3, oldValue: 2 },
//   { type: 'REMOVE', path: ['list', 2], oldValue: 3 }
// ]
```

### 3. Atomic Arrays (`reference` vs `shallow`)

```ts
import { diff } from 'object-graph-delta';

const previous = { a: [1, 2] };
const current = { a: [1, 2] };

diff(previous, current, { arrayPolicy: 'atomic', arrayEquality: 'reference' });
// [{ type: 'CHANGE', path: ['a'], value: [1, 2], oldValue: [1, 2] }]

diff(previous, current, { arrayPolicy: 'atomic', arrayEquality: 'shallow' });
// []
```

### 4. Ignore Arrays Entirely

```ts
import { diff } from 'object-graph-delta';

diff(
  { items: [1, 2, 3], version: 1 },
  { items: [9, 8, 7], version: 2 },
  { arrayPolicy: 'ignore' }
);
// [{ type: 'CHANGE', path: ['version'], value: 2, oldValue: 1 }]
```

### 5. Skip Keys and Handle Cycles

```ts
import { diff } from 'object-graph-delta';

const previous: Record<string, unknown> = { a: 1, meta: { changed: false } };
previous.self = previous;

const current: Record<string, unknown> = { a: 2, meta: { changed: true } };
current.self = current;

diff(previous, current, {
  trackCircularReferences: true,
  keysToSkip: ['meta']
});
// [{ type: 'CHANGE', path: ['a'], value: 2, oldValue: 1 }]
```

## How to Use

### Step 1: Provide Two Root Containers

The public API compares two root containers (objects or arrays):

```ts
import { diff } from 'object-graph-delta';

const previous = { user: { name: 'Alice' } };
const current = { user: { name: 'Bob' } };
```

### Step 2: Choose Options for Your Use Case

```ts
const result = diff(previous, current, {
  arrayPolicy: 'diff',
  trackCircularReferences: true,
  keysToSkip: ['_internal']
});
```

### Step 3: Consume Diff Events

```ts
for (const change of result) {
  if (change.type === 'CHANGE') {
    // change.path, change.oldValue, change.value
  }
}
```

## API Reference

### Function

```ts
diff<V>(
  previous: Container<V>,
  current: Container<V>,
  options?: DiffOptions
): DiffResult<Node<V>>[]
```

### Options

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `trackCircularReferences` | `boolean` | `true` | Prevents infinite recursion by tracking compared container pairs on the current traversal path |
| `arrayPolicy` | `'diff' \| 'atomic' \| 'ignore'` | `'diff'` | Controls array handling mode |
| `arrayEquality` | `'reference' \| 'shallow'` | `'shallow'` (when `arrayPolicy: 'atomic'` and omitted) | Used in `atomic` mode to decide array equality |
| `keysToSkip` | `readonly string[]` | `[]` | Skips object keys (does not apply to array indices) |

## Behavioral Notes (Backed by Tests)

- Equality is based on `Object.is` for leaf values.
- `NaN` equals `NaN`; `+0` and `-0` are different.
- `Date` values are compared by timestamp.
- `RegExp` values are compared by string form (`/pattern/flags`).
- Boxed primitives (`new Number`, `new String`, `new Boolean`, `Object(1n)`) compare by underlying value.
- Symbol keys and non-enumerable keys are ignored (iteration is based on `Object.keys`).
- Inherited keys can affect presence/value checks because key existence uses the `in` operator.
- Alias paths in cyclic structures can yield multiple diffs for the same logical value change at different paths.
