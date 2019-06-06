# Constant Value Types

ECMAScript proposal for constant and value types (also known as immutable types).

**Authors:** Robin Ricard (Bloomberg), Philipp Dunkel (Bloomberg)

**Champions:** TBD

**Stage:** 0

## Overview

The goal of this proposal is to introduce constant/immutable value types to JavaScript. It has multiple objectives:

- Introducing efficient data structures that makes copying and changing them cheap and will allow programs avoiding mutation of data to run faster (pattern heavily used in Redux for instance).
- Add guarantees in strict equality when comparing data. This is only possible because those data structures are deeply constant (comparing props fast is essential for efficient virtual dom reconciliation in React apps for instance)
- Be easily understood by external typesystem supersets such as TypeScript or Flow.
- Offers the possibility to improve structured cloning efficiency when messaging across workers.

This proposal presents 2 main additions to the language:

- Const Objects
- Const Arrays

Once you create one of those structures, the only accepted sub-structures will only be one of those const structures and normal value types such as `number`, `string`, `symbol` or `null`.

### Prior work on immutable data structures in JavaScript

As of today, a few libraries are actually implementing similar concepts such as [Immutable.js](https://immutable-js.github.io/immutable-js/) or [Immer](https://github.com/mweststrate/immer) that have been covered by [a previous proposal attempt](https://github.com/sebmarkbage/ecmascript-immutable-data-structures). However, the main influence to that proposal is [constant.js](https://github.com/bloomberg/constant.js/) that forces data structures to be deeply constant.

Using libraries to handle those types has multiple issues: we have multiple ways of doing the same thing that do not interoperate with each other, the syntax is not as expressive as it could be if it was integrated in the language and finally, it can be very challenging for a type system to pick up what the library is doing.

## Examples

#### Simple map

```js
const map1 = const {
    a: 1,
    b: 2,
    c: 3,
};

const map2 = map1 with .b = 5;

assert(map1 !== map2);
assert(map2 === const { a: 1, b: 5, c: 3});
```

#### Simple array

```js
const array1 = const [1, 2, 3];

const array2 = array1 with [0] = 2;

assert(array1 !== array2);
assert(array1 === const [2, 2, 3]);
```

#### Computed access

```js
const map = const { a: 1, b: 2, c: 3 };
const array = const [1, 2, 3];

const k = "b";
const i = 0;

assert((map with [k] = 5) === const { a: 1, b: 5, c: 3});
assert((array with [i] = 2) === const [2, 2, 3]);
```

#### Nested structures

```js
const marketData = const [
    { ticker: "AAPL", lastPrice: 195.855 },
    { ticker: "SPY", lastPrice: 286.53 },
];

const updatedData = marketData
    with [0].lastPrice = 195.891,
         [1].lastPrice = 286.61;

assert(updatedData === const [
    { ticker: "AAPL", lastPrice: 195.891 },
    { ticker: "SPY", lastPrice: 286.61 },
]);
```

#### Key ordering

```js
const map1 = const {
    a: 1,
    b: 2,
    c: 3,
};

const map2 = const {
    b: 2,
    a: 1,
    c: 3,
};

assert(map1 !== map2);
assert(map1 === {} with .a = 1, .b = 2, .c = 3);
assert(map1 !== {} with .b = 2, .a = 1, .c = 3);

```

#### Forbidden cases

```js
const instance = new MyClass();
const immutableContainer = const {
    instance: instance
};
// TypeError: Can't use a non-immutable type in an immutable declaration

const immutableContainer = const {
    instance: null,
};
immutableContainer with .instance = new MyClass();
// TypeError: Can't use a non-immutable type in an immutable operation

const array = const [1, 2, 3];

array.map(x => new MyClass(x));
// TypeError: Can't use a non-immutable type in an immutable operation

// The following should work:
Array.from(array).map(x => new MyClass(x))
```

#### More assertions

```js
assert(const [] with .push(1), .push(2) === const [1, 2]);
assert((const {} with .a = 1, .b = 2) === const { a: 1, b: 2 });
assert((const []).push(1).push(2) === const [1, 2]);
assert((const [1, 2]).pop().pop() === const []);
assert((const [ {} ] with [0].a = 1) === const [ { a: 1 } ]);
assert((x = 0, const [ {} ] with [x].a = 1) === const [ { a: 1 } ]);
```

## Syntax

This defines the new pieces of syntax being added to the language with this proposal.

### Const expressions and declarations

We define _ConstExpression_ by using the `const` modifier in front of otherwise normal expressions and declarations.

_ConstExpression_:

> `const` _ObjectExpression_

> `const` _ArrayExpression_

#### Examples

```js
const {}
const { a: 1, b: 2 }
const { a: 1, b: [2, 3, { c: 4 }] }
const []
const [1, 2]
const [1, 2, { a: 3 }]
```

#### Runtime verification

At runtime, if a non-const data structure is passed in a const expression, it is a Type Error. That means that the object or array expressions can't contain a Reference Type or call a function that returns a Reference Type.

### Const update expression

_ConstAssignment_:

> `.`_Identifier_ `=` _Expression_

> `[`_Expression_`] =` _Expression_

> `.`_MemberExpression_ `=` _Expression_

> `[`_Expression_`]`_MemberExpression_ `=` _Expression_

_ConstCall_:

> `.`_CallExpression_

_ConstUpdatePart_:

> _ConstAssignment_

> _ConstCall_

> _ConstUpdatePart_`,` _ConstUpdatePart_

_ConstUpdateExpresion_:

> _Identifier_ `with` _ConstUpdatePart_

#### Examples

```js
constObj with .a = 1
constObj with .a = 1, .b = 2
constArr with .push(1), .push(2)
constArr with [0] = 1
constObj with .a.b = 1
constObj with ["a"]["b"] = 1
constObj with .arr.push(1)
```

#### Runtime verification

The same runtime verification will apply. It is a Type Error when a const type gets updated with a reference type in it.

## Prototypes

### Const object prototype

In order to keep this new structure as simple as possible, the const object prototype is `null`. The `Object` namespace and the `in` should however be able to work with const objects and return const values. For instance:

```js
assert(Object.keys(const { a: 1, b: 2 }) === const ["a", "b"]);
assert("a" in const { a: 1, b: 2 });
```

### Const array prototype

The const array prototype is a const object that contains the same methods as Array with a few changes:

- `ConstArray.prototype.pop()` and `ConstArray.prototype.shift()` do not return the removed element, they return the result of the change
- `ConstArray.prototype.first()` and `ConstArray.prototype.last()` are added to return the first and last element of the const array

## FAQ

### Const classes?

Const classes are being considered as a followup proposal that would let us associate methods to const objects.

You can see an attempt at defining them in an [earlier version of this proposal](./history/with-classes.md).

### `const` vs `const` ?

`const` variable declarations and `const` value types are completely orthogonal features.

`const` variable declarations force the reference or value type to stay constant for a given identifier in a given lexical scope.

`const` value types makes the value deeply constant and unchangeable.

Using both at the same time is possible, but using a non-const variable declaration is also possible:

```js
const obj = const { a: 1, b: 2 };
let obj2 = obj with .c = 3;
obj2 = obj2 with .a = 3, .b = 3;
```

### Const equality vs normal equality

```js
assert(const { a: 1 } === const { a: 1 });
assert(Object(const { a: 1 }) !== Object(const { a: 1 }));
assert({ a: 1 } !== { a: 1 });
assert(const { a: 1, b: 2 } !== const { b: 2, a: 1 });
```

Since we established that value types are completely and deeply constant, if they have the same values stored and **inserted in the same order**, they will be considered strictly equal.

It is not the case with normal objects, those objects are instantiated in memory and strict comparison will see that both objects are located at different addresses, they are not strictly equal.

### Are there any follow up proposals being considered?

As this proposal adds a new concept to the language, we expect that other proposals might use this proposal to extend an another orthogonal feature.

We consider exploring the following proposals once this one gets considered for higher stages:

- Const classes
- ConstSet and ConstMap, the const versions of [Set](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set) and [Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map)

A goal of the broader set of proposals (including [operator overloading](https://github.com/littledan/proposal-operator-overloading/) and [extended numeric literals](https://github.com/tc39/proposal-extended-numeric-literals) is to provide a way for user-defined types to do the same as [BigInt](https://github.com/tc39/proposal-bigint).

If const classes are standardized, features like [Temporal Proposal](https://github.com/tc39/proposal-temporal) which might be able to express its types using const classes. However, this is far in the future, and we do not encourage people to wait for the addition of const classes.

### What is different with this proposal than with [previous attempts](https://github.com/sebmarkbage/ecmascript-immutable-data-structures)?

The main difference is that this proposal has a proper assignment operation using `with`. This difference makes it possible to handle proper type support, which was not possible with the former proposal.

### Would those matters be solved by a library and operator overloading?

Not quite since the `with` operation does some more advanced things such as being able to deeply change and return a value, for instance:

```js
const newState = state with .settings.theme = "dark";
```

Even with operator overloading we wouldn't be able to perform such operation.

## Glossary

#### Immutable Data Structure

A Data Structure that doesn't accept operations that change it internally, it has operations that return a new value type that is the result of applying that operation on it.

In this proposal `const object` and `const array` are immutable data structures.

#### Strict Equality

In this proposal we define strict equality as it is broadly defined in JavaScript. The operator `===` is a strict equality operator.

#### Structural Sharing

Structural sharing is a technique used to limit the memory footprint of immutable data structures. In a nutshell, when applying an operation to derive a new version of an immutable structure, structural sharing will attempt to keep most of the internal structure instinct and used by both the old and derived versions of that structure. This greatly limits the amount to copy to derive the new structure.

#### Value type

In this proposal it defines any of those: `boolean`, `number`, `symbol`, `undefiened`, `null`, `const object` and `const array`.

Value types can only contain other value types: because of that, two value types with the same contents are strictly equal.

