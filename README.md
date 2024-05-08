# @patrtorg/praesentium-doloremque-laudantium

Walk objects like this guy.

<img src="./chuck-norris.jpg" alt="Walker, Texas Ranger" width="200"/>

Map over an object in a preorder or postoder depth-first manner.
Also, provides functions for serializing and deserializng
self-referential objects using JSON pointer.

This library is designed to work well with functions that traverse
an object in the same way `JSON.stringify` and `JSON.parse` do. Namely,
preorder and postorder. To mimic that behavior entirely set the `jsonCompat`
option to `true`.

Custom traversal functions are supported for some functions. This allows you
to walk tree-like structures, such as a JSON schema, in a more efficient and
logical way. Prefer `walkEach` in these scenarios.

`map`, `walkEach`, `walkEachAsync`, `mapLeaves`, `compact`, and `truncate` support
the option `modifyInPlace` for in-place modification. Otherwise, the object is deep cloned.

```typescript
export interface MutationOption {
    /** Set to true to modify the object instead of returning a new object. */
    modifyInPlace?: boolean
}
```

## walker

```typescript
walker(obj: object, walkFn: WalkFn, options: Options = {}) => void
```

Generic walking fn that traverses an object in preorder (default) or postorder,
calling `walkFn` for each node. Can be used directly, but probably shouldn't.

```typescript
export interface Node {
    key: string | undefined
    val: any
    parents: any[]
    path: string[]
    isLeaf: boolean
    isRoot: boolean
}

export interface Options {
    postOrder?: boolean
    jsonCompat?: boolean
    traverse?(x: any): any
}
```

```typescript
import { walker } from '@patrtorg/praesentium-doloremque-laudantium'

const obj = {
    a: {
        b: 23,
        c: 24,
    },
    d: {
        e: 'Bob',
        f: [10, 20, 30],
    },
}

const nodes: Node[] = []
const walkFn = (node: Node) => {
    nodes.push(node)
}
walker(obj, walkFn, options)
nodes
```

Returns an array of nodes. Note this is how `walk` works, so prefer
that fn.

```typescript
[
  {
    key: undefined,
    parents: [],
    val: { a: { b: 23, c: 24 }, d: { e: 'Bob', f: [10, 20, 30] } },
    path: [],
    isRoot: true,
    isLeaf: false,
  },
  {
    key: 'a',
    val: { b: 23, c: 24 },
    parents: [{ a: { b: 23, c: 24 }, d: { e: 'Bob', f: [10, 20, 30] } }],
    path: ['a'],
    isLeaf: false,
    isRoot: false,
  },
  {
    key: 'b',
    val: 23,
    parents: [
      { b: 23, c: 24 },
      { a: { b: 23, c: 24 }, d: { e: 'Bob', f: [10, 20, 30] } },
    ],
    path: ['a', 'b'],
    isLeaf: true,
    isRoot: false,
  },
  ...
]
```

## walk

```typescript
walk(obj: object, options: WalkOptions = {}) => Node[]
```

Walk an object. Returns an array of all nodes in the object in either
preorder or postorder.

```typescript
export interface WalkOptions extends Options {
    leavesOnly?: boolean
}
```

```typescript
import { walk } from '@patrtorg/praesentium-doloremque-laudantium'

const obj = {
    a: {
        b: 23,
        c: 24,
    },
    d: {
        e: 'Bob',
        f: [10, 20, 30],
    },
}
walk(obj).map((x) => x.path)
```

Produces:

```json
[
    [],
    ["a"],
    ["a", "b"],
    ["a", "c"],
    ["d"],
    ["d", "e"],
    ["d", "f"],
    ["d", "f", "0"],
    ["d", "f", "1"],
    ["d", "f", "2"]
]
```

## walkEach

```typescript
walkEach(obj: object, walkFn: WalkFn, options: options?: WalkOptions & MutationOption) => object
```

```typescript
export type WalkFn = (node: Node) => void
```

Walk over an object calling `walkFn` for each node. The original
object is deep-cloned by default making it possible to simply mutate each
node as needed in order to transform the object. The cloned object
is returned if `options.modifyInPlace` is not set to true.

Below I want to walk a MongoDB JSON schema and set `additionalProperties` to `true`
wherever it exists. I traverse this tree using a custom `traverse` fn.
The original object is not modified.

```typescript
import { walkEach } from '@patrtorg/praesentium-doloremque-laudantium'

const obj = {
    bsonType: 'object',
    additionalProperties: false,
    required: ['name'],
    properties: {
        _id: {
            bsonType: 'objectId',
        },
        name: { bsonType: 'string' },
        addresses: {
            bsonType: 'array',
            items: {
                bsonType: 'object',
                additionalProperties: false,
                properties: {
                    address: {
                        bsonType: 'object',
                        additionalProperties: false,
                        properties: {
                            zip: { bsonType: 'string' },
                            country: { bsonType: 'string' },
                        },
                    },
                },
            },
        },
    },
}

const traverse = (x: any) => x.properties || (x.items && { items: x.items })
const walkFn = ({ val }: Node) => {
    if ('additionalProperties' in val) {
        val.additionalProperties = true
    }
}
walkEach(obj, walkFn, { traverse })
```

Produces:

```typescript
{
  bsonType: 'object',
  additionalProperties: true,
  required: ['name'],
  properties: {
    _id: { bsonType: 'objectId' },
    name: { bsonType: 'string' },
    addresses: {
      bsonType: 'array',
      items: {
        bsonType: 'object',
        additionalProperties: true,
        properties: {
          address: {
            bsonType: 'object',
            additionalProperties: true,
            properties: {
              zip: { bsonType: 'string' },
              country: { bsonType: 'string' },
            },
          },
        },
      },
    },
  },
}
```

## walkEachAsync

Like `walkEach` but awaits the promise returned by `walkFn` before proceeding to
the next node.

## map

Map over an object modifying values with a fn depth-first in a
preorder or postorder manner. The output of the mapper fn
will be traversed if possible when traversing preorder.

By default, nodes will be excluded by returning `undefined`.
Undefined array values will not be excluded. To customize
pass a fn for `options.shouldSkip`.

```typescript
map(obj: object, mapper: Mapper, options?: MapOptions & MutationOption) => object
```

```typescript
export type Mapper = (node: Node) => any

export type MapOptions = Omit<Options, 'traverse'> & {
    shouldSkip?(val: any, node: Node): boolean
}
```

```typescript
import { map } from '@patrtorg/praesentium-doloremque-laudantium'

const obj = {
    a: {
        b: 23,
        c: 24,
    },
    d: {
        e: 'Bob',
        f: [10, null, 30, [31, undefined, 32], 40],
    },
    g: [25, '', { h: [null, 26, 27] }],
    i: 'Frank',
}
map(obj, ({ val }) => (Array.isArray(val) ? _.compact(val) : val))
```

Produces:

```typescript
{
  a: { b: 23, c: 24 },
  d: { e: 'Bob', f: [10, 30, [31, 32], 40] },
  g: [25, { h: [26, 27] }],
  i: 'Frank',
}
```

Postorder

```typescript
const obj = {
    bob: {
        scores: ['87', 'x97', 95, false],
    },
    joe: {
        scores: [92, 92.5, '73.2', ''],
    },
    frank: {
        scores: ['abc', ''],
    },
}
const result = map(
    obj,
    ({ val, isLeaf }) => {
        if (isLeaf) {
            return parseFloat(val)
        }
        return Array.isArray(val) ? _.compact(val) : val
    },
    { postOrder: true }
)
```

Produces:

```typescript
{
  bob: { scores: [87, 95] },
  joe: { scores: [92, 92.5, 73.2] },
  frank: { scores: [] },
}
```

Custom `shouldSkip` fn

```typescript
const obj = {
    bob: {
        scores: ['87', 'x97', 95, false],
    },
    joe: {
        scores: [92, 92.5, '73.2', ''],
    },
    frank: {
        scores: ['abc', ''],
    },
}

const shouldSkip = (val: any, node: Node) =>
    _.isEmpty(val) && !parentIsArray(node)
const result = map(
    obj,
    ({ val, isLeaf }) => {
        if (isLeaf) {
            return parseFloat(val)
        }
        return Array.isArray(val) ? _.compact(val) : val
    },
    { postOrder: true, shouldSkip }
)
```

Produces:

```typescript
{
  bob: { scores: [87, 95] },
  joe: { scores: [92, 92.5, 73.2] },
}
```

## mapLeaves

```typescript
mapLeaves(obj: object, mapper: Mapper, options?: MapOptions) => object
```

Map over the leaves of an object with a fn. By default, nodes will be excluded
by returning `undefined`. Undefined array values will not be excluded. To customize
pass a fn for `options.shouldSkip`.

```typescript
import { mapLeaves } from '@patrtorg/praesentium-doloremque-laudantium'

const obj = {
    a: {
        b: 23,
        c: 24,
    },
    d: {
        e: 100,
        f: [10, 20, 30],
    },
}
mapLeaves(obj, ({ val }) => val + 1)
```

Produces:

```typescript
{
  a: { b: 24, c: 25 },
  d: { e: 101, f: [11, 21, 31] },
}
```

## findNode

```typescript
findNode(obj: object, findFn: FindFn, options?: Options) => Node | undefined
```

Search for a node and short-circuit the tree traversal if it's found.

```typescript
import { findNode } from '@patrtorg/praesentium-doloremque-laudantium'

const obj = {
    name: 'Joe',
    address: {
        city: 'New York',
        state: 'NY',
        zipCode: '10001',
    },
    likes: ['Stock Market', 'Running'],
}

findNode(obj, (node) => {
    return _.isEqual(node.path, ['address', 'zipCode'])
})
```

Produces:

```typescript
{
  key: 'zipCode',
  val: '10001',
  parents: [
    { city: 'New York', state: 'NY', zipCode: '10001' },
    {
      name: 'Joe',
      address: { city: 'New York', state: 'NY', zipCode: '10001' },
      likes: ['Stock Market', 'Running'],
    },
  ],
  path: ['address', 'zipCode'],
  isLeaf: true,
  isRoot: false,
}
```

## flatten

```typescript
flatten(obj: object, options?: WalkOptions & FlattenOptions) => object
```

```typescript
interface FlattenOptions {
    /** Defaults to '.' */
    separator?: string
    /** Flatten objects and not arrays */
    objectsOnly?: boolean
}
```

Flatten an object's keys. Optionally pass `separator` to determine
what character to join keys with. Defaults to '.'. If an array is
passed, an object of path to values is returned unless the `objectsOnly`
option is set.

```typescript
import { flatten } from '@patrtorg/praesentium-doloremque-laudantium'

const obj = {
    a: {
        b: 23,
        c: 24,
    },
    d: {
        e: 100,
        f: [10, 20, 30],
    },
}
flatten(obj)
```

Produces:

```typescript
{
  'a.b': 23,
  'a.c': 24,
  'd.e': 100,
  'd.f.0': 10,
  'd.f.1': 20,
  'd.f.2': 30,
}
```

## unflatten

```typescript
unflatten(obj: object, options?: UnflattenOptions) => object
```

```typescript
interface UnflattenOptions {
    /** Defaults to '.' */
    separator?: string | RegExp
}
```

Unflatten an object previously flattened. Optionally pass `separator`
to determine what character or RegExp to split keys with.
Defaults to '.'.

```typescript
import { unflatten } from '@patrtorg/praesentium-doloremque-laudantium'

const obj = {
    'a.b': 23,
    'a.c': 24,
    'd.e': 100,
    'd.f.0': 10,
    'd.f.1': 20,
    'd.f.2.g': 30,
    'd.f.2.h.i': 40,
}
unflatten(obj)
```

Produces:

```typescript
{
  a: {
    b: 23,
    c: 24,
  },
  d: {
    e: 100,
    f: [10, 20, { g: 30, h: { i: 40 } }],
  },
}
```

## compact

```typescript
compact(obj: object, options: CompactOptions & MutationOption) => object
```

```typescript
interface CompactOptions {
    removeUndefined?: boolean
    removeNull?: boolean
    removeEmptyString?: boolean
    removeFalse?: boolean
    removeNaN?: boolean
    removeEmptyObject?: boolean
    removeEmptyArray?: boolean
    compactArrays?: boolean
    removeFn?: (val: any, node: Node) => boolean
}
```

Compact an object, removing fields recursively according to the supplied options.
All option flags are `false` by default. If `compactArrays` is set to `true` arrays
will be compacted based on the enabled remove option flags.

```typescript
const obj = {
    a: {
        b: [null, null, 21, '', { b1: null }, { b2: 26 }],
    },
    c: [],
    d: [42, null],
    e: {
        f: {
            g: '',
            h: undefined,
            i: 'null',
        },
    },
}
const result = compact(obj, {
    removeUndefined: true,
    removeEmptyString: true,
    removeNull: true,
    compactArrays: true,
    removeEmptyArray: true,
    removeEmptyObject: true,
    removeFn: (val: any) => val === 'null',
})
```

Produces:

```typescript
{
  a: { b: [21, { b2: 26 }] },
  d: [42],
}
```

## truncate

```typescript
truncate(obj: object, options: TruncateOptions & MutationOption) => object
```

```typescript
export interface TruncateOptions {
    /** Max allowed depth of objects/arrays. Default to Infinity */
    maxDepth?: number
    /** What to replace an object/array at the maximum depth with. Defaults to '[Truncated]' */
    replacementAtMaxDepth?: any
    /** Max allowed length of a string. Defaults to Infinity */
    maxStringLength?: number
    /** What to replace the last characters of the truncated string with. Defaults to '...' */
    replacementAtMaxStringLength?: string
    /** Max allowed length of an array. Defaults to Infinity */
    maxArrayLength?: number
    /** Transform instances of Error into plain objects so that truncation can be performed. Defautls to false */
    transformErrors?: boolean
}
```

Truncate allows you to limit the depth of nested objects/arrays,
the length of strings, and the length of arrays. Instances of Error
can be converted to plain objects so that the enabled truncation options
also apply to the error fields. All truncation methods are opt-in.

Note: For the best performance you should consider setting `modifyInPlace`
to `true`.

```typescript
const obj = {
    a: {
        b: 'Frank',
        c: {
            d: 'Joe',
        },
        e: null,
    },
    f: 42,
}
truncate(obj, { depth: 2 })
```

Produces:

```typescript
{
  a: {
    b: 'Frank',
    c: '[Truncated]',
    e: null,
  },
  f: 42,
}
```

## size

```typescript
size(val: any) => number
```

Estimate the size in bytes.

```typescript
const obj = {
    a: {
        b: 'hello',
    },
    c: Symbol('hello'),
    d: {
        e: [true, false],
    },
    f: [42, 10n],
}
size(obj)
// 44
```

## Helper fns

These helper fns are exported for your convenience.

```typescript
export const isObjectOrArray = _.overSome([_.isPlainObject, _.isArray])

export const defShouldSkip = (val: any, node: Node) =>
    val === undefined && !parentIsArray(node)

export const parentIsArray = (node: Node) => {
    const parent = node.parents[0]
    return Array.isArray(parent)
}

export const defTraverse = (x: any) => isObjectOrArray(x) && !_.isEmpty(x) && x
```

## addRefs

```typescript
addRefs(obj: object, options?: RefOptions): object
```

Replace duplicate objects refs with pointers to the first
object seen.

```typescript
import { addRefs } from '@patrtorg/praesentium-doloremque-laudantium'

const apiOutput = {
    1: 'foo',
    2: 'bar',
    3: 'baz',
}

const detailsOutput = {
    1: 'bla',
    2: 'bla',
    3: 'bla',
}

const obj = {
    api: {
        input: [1, 2, 3],
        output: apiOutput,
    },
    details: {
        input: apiOutput,
        output: detailsOutput,
    },
    writeToDB: {
        input: detailsOutput,
    },
}
addRefs(obj)
```

Produces:

```typescript
{
  api: {
    input: [1, 2, 3],
    output: { '1': 'foo', '2': 'bar', '3': 'baz' },
  },
  details: {
    input: { $ref: '#/api/output' },
    output: { '1': 'bla', '2': 'bla', '3': 'bla' },
  },
  writeToDB: { input: { $ref: '#/details/output' } },
}
```

## deref

```typescript
deref(obj: object, options?: RefOptions): object
```

Rehydrate objects by replacing refs with actual objects.

```typescript
import { deref } from '@patrtorg/praesentium-doloremque-laudantium'

const obj = {
    api: {
        input: [1, 2, 3],
        output: { '1': 'foo', '2': 'bar', '3': 'baz' },
    },
    details: {
        input: { $ref: '#/api/output' },
        output: { '1': 'bla', '2': 'bla', '3': 'bla' },
    },
    writeToDB: { input: { $ref: '#/details/output' } },
}
deref(obj)
```

Produces:

```typescript
{
  api: {
    input: [1, 2, 3],
    output: { '1': 'foo', '2': 'bar', '3': 'baz' },
  },
  details: {
    input: { '1': 'foo', '2': 'bar', '3': 'baz' },
    output: { '1': 'bla', '2': 'bla', '3': 'bla' },
  },
  writeToDB: { input: { '1': 'bla', '2': 'bla', '3': 'bla' } },
}
```
