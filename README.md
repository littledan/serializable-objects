# User-defined structured clone for JavaScript objects

This proposal defines a mechanism such that JavaScript developers can define their own behavior for objects when they undergo [HTML serialization/deserialization](https://html.spec.whatwg.org/#safe-passing-of-structured-data), e.g., in [`postMessage`](https://html.spec.whatwg.org/#window-post-message-steps) or [IndexedDB](https://w3c.github.io/IndexedDB/#object-store-storage-operation).

The HTML serialization (historically called "structured clone") algorithm includes meaningful serialization and transfer semantics for certain built-in JavaScript and web platform datatypes. However, user-defined objects are cloned by the rough equivalent of `{...obj}`--that is, copying only own properties. This proposal allows classes to define their own serialization and transfer algorithms.

## Motivation

### Ergonomics in working with Workers

[WebWorkers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers) are based on a `postMessage` interface, and new API proposals for improved usability of workers such as [TaskWorklet](https://github.com/developit/task-worklet) are as well. Instances of user-defined classes can't be directly sent and received by `postMessage`, but require explicit translation before calling `postMessage` and in the event handler to decode. User-defined serialization could make the translation clearer, by indicating it in the definition of the data type instead. Methods can be used in a natural way on both sides of the `postMessage` boundary.

### Explaining the Web Platform

Many built-in web platform features have defined serialization semantics. The [Extensible Web Manifesto](https://extensiblewebmanifesto.org/) encourages exposing fundamental capabilities of the web platform to developers, and the [Layered APIs proposal](https://github.com/drufball/layered-apis) doubles down on this by encouraging some new  features to be specified exclusively in terms of what could be expressed in JavaScript. This proposal attempts to fill in one of these fundamental primitives.

### Building block for future JS/web features

This proposal includes a mechanism for identifying "the same class" across different [JavaScript Realms](https://tc39.github.io/ecma262/#realm), possibly including across processes and in persistent storage. Such an identification of same-ness could be useful for several other language features, including
- **A multithreaded, immutable heap of value types**: Value Types are an idea for permitting user-defined primitive types, which would be deeply immutable values. With deep immutability, it may be possible to expose sharing of value objects across different threads, without copying, on a shared heap. However, with the corresponding JavaScript object heaps still not shared, a separate value type would need to be identified within the other thread. The correspondence of these types could use the same mechanism as the correspondence of classes in serialization.
- **A multithreaded, mutable heap of typed objects**: The [Typed Objects proposal](https://github.com/tschneidereit/proposal-typed-objects/blob/master/explainer.md) provides mutable objects with a consistent shape, and is designed to interact well with the [WebAssembly GC proposal](https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md). An early idea for a follow-on proposal here is to expose some of these objects across multiple WebAssembly threads, and possibly in JavaScript workers. This will require multiple JavaScript typed object definitions to correspond to the same instance, similarly to the value types use case.
- **Cross-realm brand checks**: Internal slots in the ECMAScript standard, as well as WebIDL platform object interface checks, are shared across JavaScript Realms, in the sense that methods from one realm can be successfully applied to objects from another realm, and they will find that internal slots/brand checks match across them. For JavaScript APIs were to achieve this, we would need to be able to associate "the same class declaration" across multiple evaluations of the same code in different Realms (in addition to some way to rebind private names, e.g., through [rebind decorators](https://github.com/tc39/proposal-decorators/issues/17)).

## Example code

To implement [HTML's Serializable example](https://html.spec.whatwg.org/#deserialization-steps) in JavaScript:

```js
import { Serializable } from "@std/structured-clone";

class Person extends Serializable {
  #name;
  #bestFriend;
  constructor(name, bestFriend) {
    this.#name = name;
    this.#bestFriend = bestFriend;
  }
  serialize() {
    return { name: this.#name, bestFriend: this.#bestFriend };
  }
  static deserialize(payload) {
    return new Person(payload.name, payload.bestFriend);
  }
}
Serializable.register(Person, globalThis);
```

## Design goals
- Ergonomics: This new mechanism should be clean to use in practice and correspond to ordinary object-oriented JavaScript. ("Functional" JS based on object literals already has sufficient support for structured clone.)
- Performance:
  - Hard requirement: Do not add overhead to the serialization algorithm applied to existing objects (e.g., through additional property accesses).
  - Ideally: Performance of user-defined structured clone should be similar to the mechanism that would be written by hand. For example, the interface should not require additional allocations which would otherwise be unnecessary.
- Expressivity:
  - Both serialize and transfer operations should be supported.
  - Circularities in serialized user-defined types should be handled properly. 
- Matching the same class:
  - Baseline: In practice, a class shouldn't be serialized as one class and then deserialized as another class, which would result in bugs.
  - Security: It should be possible to guarantee (for some level of guarantee) that we're talking about "the same class" (whatever that means exactly), to avoid misinterpretation which could lead to the [confused deputy problem](https://en.wikipedia.org/wiki/Confused_deputy_problem).

## Mechanism

### Serializable class/instance lifecycle

### Sub-serialization and transfer support

### Class identifier tuple

#### Included

##### Origin

##### The class's original name

#### Infeasible components

##### The module specifier where the class is defined

#### Possible components

##### The class's source code

##### The script's source code

##### A developer-provided GUID

## Q/A

#### Why require both extending and registering?

#### How sure are we that it's the same class on both sides?

#### Why require methods and not automatically serialize/deserialize?

#### What is the performance impact?

#### How does this proposal interact with subclassing?

## Alternatives considered

### postMessaging the class

## Related work

- [Napa.js Transportable](https://github.com/Microsoft/napajs/blob/master/docs/api/transport.md#transportable) -- a broadly similar interface

## Standardization plan

Structured clone is defined in WHATWG's HTML specification, which provides hooks such that any web stanard can define serialization semantics. Therefore, a web standards body would be the right final destination if this becomes a standard. If this gathers interest, the next step would be to form a WICG for further development, which would collaborate on the precise specification and tests, and identify the appropriate final standards document. The identification of "the same class" may be useful beyond this proposal, across JavaScript. Therefore, it may be worth defining (at least partly) in TC39.
