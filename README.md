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
import { Serializable, register } from "@std/structured-clone";

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
register(Person);
```

## Design goals
- Ergonomics: This new mechanism should be clean to use in practice and correspond to ordinary object-oriented JavaScript. ("Functional" JS based on object literals already has sufficient support for structured clone.)
- Performance:
    - Minimum requirement: Do not add overhead to the serialization algorithm applied to existing objects (e.g., through additional property accesses).
    - Ideally: Performance of user-defined structured clone should be similar to the mechanism that would be written by hand. For example, the interface should not require additional allocations which would otherwise be unnecessary.
- Expressivity: Both serialize and transfer operations should be supported.

## Proposed mechanism

### Interface

The [built-in module](https://github.com/tc39/proposal-javascript-standard-library/) `@std/structured-clone` exports classes which are designed to be subclassed to define structured clone semantics.
- To support serialization:
  ```js
  import { Serializable, register } from "@std/structured-clone";
  class Klass extends Serializable {
    constructor() { super(); }
    serialize(forStorage) { /* return another object which could be serialized */ }
    static deserialize(obj) { /* return an instance of Klass */ }
  }
  register(Klass);
  ```
- To support transfer (e:
  ```js
  import { Transferable, register } from "@std/structured-clone";
  class Klass extends Transferable {
    constructor() { super(); }
    transfer() { /* return another object which could be transfered */ }
    static receiveTransfer(obj) { /* return an instance of Klass */ }
  }
  register(Klass);
  ```
- To support both, use the `SeriaizableTransferable` class and override all four methods.

### How it works at a high level

When a subclass of Serializable is serialized, e.g., by `postMessage`, the `serialize` method is called to serialize it the sending end. This is expected to return return a simpler object, which can then be serialized fully into the browser's internal data structures. Along with that serialization is an identifier for the class (described in more detail in the "Class identifier tuple" section). On the receiving end, the class identifier is looked up, in order to find the corresponding, separate class in that Realm to call for deserialization. Transfer semantics are analogous.

### Algorithm modifications

When a class is registered:
- An associated record is constructed for the class `klass`, containing:
    - A reference to `klass`
    - The class identifier tuple for `klass` (described below)
    - `klass.deserialize` and `klass.prototype.serialize` (pre-cached for performance and predictability)
- If the environment settings object already has a class registered with it with the same primary key of the class identifier tuple, throw an error.
- This record is added to a list of class identifier tuples associated with the environment settings object

In the constructor for Serializable instances:
- The environment settings object is searched for a record with the class reference matching `new.target`. If none is found, an error is thrown. (Note: Serializable subclasses need to be registered separately.)
- A reference to the record is held in an internal slot in the instance.

The serialization of of `Serializable` objects includes:
- identifier: The class identifier tuple referenced from the instance (both primary and secondary components)
- payload: The result of calling the `serialize` method on the object

The deserialization steps of `Serializable`:
- Look up in the current environment settings object if there is a registered, Serializable subclass which has a matching class identifier tuple. If not, throw an uncaught exception.
- Deserialize the payload.
- Call the deserialize method from the record, given the deserialized payload as an argument.

### Class identifier tuple

The "hard part" of this proposal is how to figure out when we're talking about "the same class". Some design considerations here:
- In general, structured clone cannot depend on anything in the JavaScript heap--it can work across processes, or in persistent storage. Everything in this tuple must be fully serializable.
- It should be efficient to serialize and deserialize the class tuple in practice.
- Classes should not be accidentally mis-identified with each other (serialized as one and deserialized as another), as this could be a source of bugs.
- Mis-identifying two classes as equal may have security implications, with respect to multiple perspectives on security:
    - The same-origin policy: In the context of `postMessage`, deserialization triggers code execution with a sender-supplied argument before the recipient has the chance to check the origin.
    - Object capability security: The existence of an object of a particular shape may be used to imply certain capabilities. A `deserialize` method which trusts the sender for these purposes could escalate privileges if it's "tricked" into creating an object which it "shouldn't", if classes are falsely identified with each other.

Some components of the class identifier tuple may be "primary" and others "secondary", in that no two classes can be registered an environment settings object which agree in only their primary key, but both the primary and secondary key are used for serialization and deserialization. For example, we may decide that no two classes of the same name can be registered, but then verify that the source code of classes matches during deserialization.

#### Possible components

Some ideas for what the class identifier tuple might contain (none of these are certainly included or excluded; more investigation is needed):
- An **origin**: Including an origin may be important for maintaining the origin model, as described above. We may want to use the origin of the environment settings object, though this would prevent any custom serialization use across different origins, which could be unfortunate. Another option could be the origin of the active script during the `register` call. (Multiple origins may be used, to double-key it and be even more specific.)
- A **GUID**: A developer-supplied string to make accidental confusion less likely. This would not bring any guarantees, as it can be trivially spoofed.
- The **class's name**: Similarly, avoids accidental overlap.
- A **module specifier**: We might want to check that the class comes from the same module by comparing the module specifier. However, it's not clear how this would work, since a module doesn't have a single module specifier, but rather might be addressed by multiple module specifiers.
- Some **source code** (may include multiple):
    - **The class**: Checking that the source of the class is the same seems like a pretty strong way to prevent mix-ups. However, it doesn't provide strong proof that the behavior is the same, as the scope surrounding the class may be different.
    - **A script/module**: For example, the entire source of the active script during the `register` call could be included. If the whole script is included, then maybe some code earlier in the script could verify that a frozen, unmodified environment is being used (unclear how practical this is). However, this may introduce brittleness, if the main thread and a worker use a class from a module in common, but are bundled with different things, causing a mismatch.
- A **URL**: This could be the URL of the active script when `register` is called, for example. Checking the same-ness of the URL could prove we're talking about the same thing. However, it runs into the same issue as comparing the script/module source, in terms of being fragile when combined with bundling. Maybe the baseURL of the environment settings object would be a little more flexible (but fairly ad-hoc and fragile).

Some of these components are rather large. To avoid copying overhead, a cryptographic hash of the tuple could be used instead (for implementations that find this technique useful). The hash could be calculated once during `register`.

## Q/A

#### Why require both extending and registering?

Extending is needed so that instances have the right platform object brand, as well as a reference to the class identifier tuple based on the `new.target`. Registering is needed in order to be able to receive objects through deserialization. Neither one subsumes the other; it sort of matches how custom elements work.

#### How sure are we that it's the same class on both sides?

This depends on what we put in the class identifier tuple. TBD!

#### Why require methods and not automatically serialize/deserialize?

Serialization may require application-dependent steps, and not just be a serialization of each field. For example, you may make a class which wraps a non-serializable platform object, and have a way to serialize it for the cases you use it in.

#### What is the performance impact?

We'll have to verify this when there's a real implementation, but the design is hoped to be efficient in practice due to the following:
- On existing objects: There should be no extra checks or steps, so no performance impact.
- On new Serializable objects: The overhead should be as much as, calling a function, then performing structured clone, then looking for a hash in a hashtable, then calling another function. Hopefully it won't be so bad either!

#### How does this proposal interact with subclassing?

In this proposal, subclasses need to be registered separately from superclasses. It may be possible to relax this somewhat, but in practice, a subclass is likely to require its own serialization/deserialization steps, so it doesn't seem like a very big burden.

There is no way to make a subclass of an existing class serializable (e.g., subclass Array in a way that's serializable). This could be provided in a follow-on proposal, based on a sort of Serializable mixin. However, WebIDL doesn't have support for defining JavaScript-level mixins, and implementations are also likely to be somewhat more complicated as well. Therefore, this proposal leaves the mixin idea for "v2".

#### Why not do XYZ alternative instead?

One alternative considered was somehow passing a class itself to a worker, a la bloecks, to solve the class identifier tuple problem and make serialization implicit. However, it's not clear how all that would work.

If you see any further issues, or if you have another idea for how this should work, please file an issue or a PR!

### How are circularities handled, and infinite loops in the serialization algorithms prevented?

In this proposal, the user-defined subclass would need their own handling of these situations. Naive implementations can easily lead to infinite loops. Ideas for how to resolve this issue are appreciated; please file an issue with any thoughts.

## Related work

- [Napa.js Transportable](https://github.com/Microsoft/napajs/blob/master/docs/api/transport.md#transportable) -- a broadly similar interface

## Standardization plan

Structured clone is defined in WHATWG's HTML specification, which provides hooks such that any web stanard can define serialization semantics. Therefore, a web standards body would be the right final destination if this becomes a standard. If this gathers interest, the next step would be to form a WICG for further development, which would collaborate on the precise specification and tests, and identify the appropriate final standards document. The identification of "the same class" may be useful beyond this proposal, across JavaScript. Therefore, it may be worth defining (at least partly) in TC39.
