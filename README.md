# User-defined structured clone for JavaScript objects

This proposal defines a mechanism such that JavaScript developers can define their own behavior for objects when they undergo [HTML serialization/deserialization](https://html.spec.whatwg.org/#safe-passing-of-structured-data), e.g., in [postMessage](https://html.spec.whatwg.org/#window-post-message-steps) or [IndexedDB](https://w3c.github.io/IndexedDB/#object-store-storage-operation).

## Motivation

### Ergonomics in working with Workers

WebWorkers are based on a postMessage interface. This forces developers to 

### Explaining the Web Platform

### Building block for future JS/web features

## Example code

```js
class 
```

## Mechanism

### Serializable class/instance lifecycle

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
