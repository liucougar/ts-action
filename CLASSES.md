## Class-based action creator deprecation

In version 8 of `ts-action`, class-based action creators were deprecated in favour of function-based action creators. Function-based creators are simpler and don't require an unpleasant `Object.setPrototypeOf` call for the returned actions to be usable with `reactjs/redux`.

Class-based action creators can still be used, but the import has changed:

```ts
import { action } from "ts-action/classes";
```

Also, if you are using `ts-action-operators` with class-based action creators, you will need to change the import:

```ts
import { ofType } from "ts-action-operators/classes";
```

The documentation from the `README` that accompanied previous versions of `ts-action` follows.

## Usage

Action creators are declared using the `action` method:

```ts
import { action } from "ts-action/classes";
const Foo = action("FOO");
```

Actions are created using the action creator as a class:

```ts
store.dispatch(new Foo());
```

Although the actions are created as class instances, internally, the `prototype` is reset, so they are compatible with `reactjs/redux`, as they are considered to be plain objects.

For actions with payloads, the payload type is specified using the [`payload`](#payload) method:

```ts
import { action, payload } from "ts-action/classes";
const Foo = action("FOO", payload<{ foo: number }>());
```

and the payload value is specified when creating the action:

```ts
store.dispatch(new Foo({ foo: 42 }));
```

To have the properties added to the action itself - rather than a `payload` property - use the [`props`](#props) method instead. Or, for more control over the parameters accepted by the constructor, use the [`base`](#base) method.

Action creators have a `type` property that can be used to narrow an action's TypeScript type in a reducer.

The action types can be combined into a discriminated union and the action can be narrowed to a specific TypeScript type using a `switch` statement, like this:

```ts
import { action, payload, union } from "ts-action/classes";

const Foo = action("FOO", payload<{ foo: number }>());
const Bar = action("BAR", payload<{ bar: number }>());
const Both = union({ Foo, Bar });

interface State { foo?: number; bar?: number; }
const initialState = {};

function fooBarReducer(state: State = initialState, action: typeof Both): State {
  switch (action.type) {
  case Foo.type:
    return { ...state, foo: action.payload.foo };
  case Bar.type:
    return { ...state, bar: action.payload.bar };
  default:
    return state;
  }
}
```

Or, the package's `isType` method can be used to narrow the action's type using `if` statements, like this:

```ts
import { action, isType, payload } from "ts-action/classes";

const Foo = action("FOO", payload<{ foo: number }>());
const Bar = action("BAR", payload<{ bar: number }>());

interface State { foo?: number; bar?: number; }
const initialState = {};

function fooBarReducer(state: State = initialState, action: Action): State {
  if (isType(action, Foo)) {
    return { ...state, foo: action.payload.foo };
  }
  if (isType(action, Bar)) {
    return { ...state, bar: action.payload.bar };
  }
  return state;
}
```

Or, the package's [`reducer`](#reducer) method can be used to create a reducer function, like this:

```ts
import { action, on, payload, reducer } from "ts-action/classes";

const Foo = action("FOO", payload<{ foo: number }>());
const Bar = action("BAR", payload<{ bar: number }>());

interface State { foo?: number; bar?: number; }
const initialState = {};

const fooBarReducer = reducer<State>([
  on(Foo, (state, { payload }) => ({ ...state, foo: payload.foo })),
  on(Bar, (state, { payload }) => ({ ...state, bar: payload.bar }))
], initialState);
```

## API

* [action](#action)
* [empty](#empty)
* [payload](#payload)
* [props](#props)
* [base](#base)
* [union](#union)
* [isType](#isType)
* [guard](#guard)
* [reducer](#reducer)

<a name="action"></a>

### action

```ts
function action<T>(type: T)
function action<T, B extends {}, C extends Ctor<{}>>(type: T, config: {})
```

The `action` method returns an action creator. Action creators are classes and actions are be created using `new`:

```ts
const Foo = action("FOO");
const foo = new Foo();
console.log(foo); // { type: "FOO" }
```

The `type` argument passed to `action` must be a string literal or have a string-literal type. Otherwise, TypeScript will not be able to narrow actions in a discriminated union.

The `type` option passed to the `action` method can be obtained using the creator's static `type` property:

```ts
switch (action.type) {
case Foo.type:
  return { ...state, foo: action.payload.foo };
default:
  return state;
}
```

To define propeties, the `action` method can be passed a `config`. The `config` should be created using the `empty`, `payload`, `props` or `base` methods.

<a name="empty"></a>

### empty

```ts
function empty()
```

The `empty` method constructs the `config` for the `action` method. To declare an action without a payload or properties , call it like this:

```ts
const Foo = action("FOO", empty());
const foo = new Foo();
console.log(foo); // { type: "FOO" }
```

Actions default to being empty; if only a `type` is passed to the `action` call, an empty action is created by default:

```ts
const Foo = action("FOO");
const foo = new Foo();
console.log(foo); // { type: "FOO" }
```

<a name="payload"></a>

### payload

```ts
function payload<T>()
```

The `payload` method constructs the `config` for the `action` method. To declare action properties within a `payload` property, call it like this:

```ts
const Foo = action("FOO", payload<{ name: string }>());
const foo = new Foo({ name: "alice" });
console.log(foo); // { type: "FOO", payload: { name: "alice" } }
```

<a name="props"></a>

### props

```ts
function props<T>()
```

The `props` method constructs the `config` for the `action` method. To declare action properties at the same level as the `type` property, call it like this:

```ts
const Foo = action("FOO", props<{ name: string }>());
const foo = new Foo({ name: "alice" });
console.log(foo); // { type: "FOO", name: "alice" }
```

The `props` method is similar to the `payload` method, but with `props`, the specified properties are added to the action itself - rather than a `payload` property.

<a name="base"></a>

### base

```ts
function base<C extends Ctor<{}>>(BaseCtor: C)
```

The `base` method constructs the `config` for the `action` method. To declare a base class with properties, call it like this:

```ts
const Foo = action("FOO", base(class { constructor(public name: string) {} }));
const foo = new Foo("alice");
console.log(foo); // { type: "FOO", name: "alice" }
```

The `base` method offers more control over property defaults, etc. as the base class is declared inline.

<a name="union"></a>

### union

The `union` method can be used to infer a union of actions - for type narrowing using a discriminated union. It's passed an object literal of action creators and returns a value that can be used with TypeScript's `typeof` operator, like this:

```ts
const Both = union({ Foo, Bar });
function reducer(state: any = [], action: typeof Both): any {
  switch (action.type) {
  case Foo.type:
    return ... // Here the action will be narrowed to Foo.
  case Bar.type:
    return ... // Here the action will be narrowed to Bar.
  default:
    return state;
  }
}
```

<a name="isType"></a>

### isType

`isType` is a TypeScript [type guard](https://www.typescriptlang.org/docs/handbook/advanced-types.html) that will return either `true` or `false` depending upon whether the passed action is of the appropriate type. TypeScript will narrow the type when used with an `if` statement.

For example:

```ts
if (isType(action, Foo)) {
  // Here, TypeScript has narrowed the type, so the action is strongly typed
  // and individual properties can be accessed in a type-safe manner.
}
```

`isType` can also be passed multiple action creators:

```ts
if (isType(action, { Foo, Bar })) {
  // Here, TypeScript has narrowed the type to `typeof union({ Foo, Bar })`.
}
```

### guard

`guard` is a higher-order equivalent of `isType`. That is, it returns a TypeScript [type guard](https://www.typescriptlang.org/docs/handbook/advanced-types.html) that will, in turn, return either `true` or `false` depending upon whether the passed action is of the appropriate type. The `guard` method is useful when dealing with APIs that accept type guards.

For example, `Array.prototype.filter` accepts a type guard:

```ts
const actions = [new Foo(), new Bar()];
const filtered = actions.filter(guard(Foo));
// TypeScript will have narrowed the type of filtered, so the actions within
// the array are strongly typed and individual properties can be accessed in
// a type-safe manner.
```

`guard` can also be passed multiple action creators:

```ts
const actions = [new Foo(), new Bar(), new Baz()];
const filtered = actions.filter(guard({ Foo, Bar }));
```

<a name="reducer"></a>

### reducer

The `reducer` method creates a reducer function out of the combined, action-specific reducers specified in the array. The array should be populated with the results of calls to the `on` method.

The `on` method creates a reducer for a specific, narrowed action and returns an object - containing the created reducer and the types of one or more actions creators.

```ts
import { action, on, payload, reducer } from "ts-action/classes";

const Foo = action("FOO", payload<{ foo: number }>());
const Bar = action("BAR", payload<{ bar: number }>());
const FooError = action("FOO_ERROR", payload<{ foo: number, error: {} }>());
const BarError = action("BAR_ERROR", payload<{ bar: number, error: {} }>());

interface State { foo?: number; bar?: number; error?: {} }
const initialState = {};

const fooBarReducer = reducer<State>([
  on(Foo, (state, { payload }) => ({ ...state, foo: payload.foo })),
  on(Bar, (state, { payload }) => ({ ...state, bar: payload.bar })),
  on({ FooError, BarError }, (state, { payload }) => ({ ...state, error: payload.error }))
], initialState);
```