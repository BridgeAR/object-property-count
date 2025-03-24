# ECMAScript Proposal: `Object.propertyCount`

## Status

Champion(s): champion name(s)

Author: Ruben Bridgewater <ruben@bridgewater.de>

Stage: 0

## Overview

This proposal introduces `Object.propertyCount`, a built-in method to efficiently and intuitively obtain the count of an object's own properties, with support for distinguishing among indexed properties, string-keyed properties, symbol properties, enumerable and non-enumerable properties, without the performance overhead of intermediate array allocations of the object's keys.

## Motivation

Developers frequently rely on patterns like:

```javascript
const obj = { a: 1, b: 2 };
const count = Object.keys(obj).length;
```

However, this approach creates unnecessary memory overhead and garbage collection pressure, as an intermediate array is allocated solely for counting properties. Highly-used runtimes, frameworks, and libraries (e.g., Node.js, React, Lodash, Angular) frequently utilize `Object.keys(obj).length`, compounding performance issues across applications.

For instance, React often counts props or state keys:

```javascript
// React component example
const propCount = Object.keys(this.props).length;
```

Replacing these patterns with a native and optimized counting method significantly reduces memory overhead, garbage collection, and as such, runtime performance impacts.

## Problem Statement

Currently, accurately counting object properties involves verbose and inefficient workarounds:

```javascript
const count = [
  ...Object.getOwnPropertyNames(obj),
  ...Object.getOwnPropertySymbols(obj)
].length;
```

This creates intermediate arrays, causing unnecessary memory usage and garbage collection, impacting application performance — especially at scale and in performance-critical code paths.

## Proposed Syntax

```javascript
Object.propertyCount(target[, options])
```

### Parameters

- **`target`**: The object whose properties will be counted.
  - Throws `TypeError` if target is not provided.
- **`options`** *(optional)*: An object specifying filtering criteria:
  - `keyTypes`: Array specifying property types to include:
    - Possible values: `'indices'`, `'string'`, `'symbol'`.
    - Defaults to `['indices', 'string']` (aligning closely with `Object.keys`).
    - Throws `TypeError` if provided invalid values.
  - `enumerable`: Indicates property enumerability:
    - `true` to count only enumerable properties (default).
    - `false` to count only non-enumerable properties.
    - `'all'` to count both enumerable and non-enumerable properties.
    - Throws `TypeError` if provided invalid values.

Defaults align closely with `Object.keys` for ease of adoption, ensuring intuitive behavior without needing explicit configuration in common cases.

## Detailed Examples and Edge Cases

- **Empty object**:

```javascript
Object.propertyCount({}); // returns 0
```

- **Object without prototype**:

```javascript
const obj = Object.create(null);
obj.property = 1;
Object.propertyCount(obj); // returns 1
```

- **Numeric-like non-index (CanonicalNumericIndexString) keys**:

```javascript
let obj = { "01": "string key", 1: "index", 2: "index" };
Object.propertyCount(obj, { keyTypes: ['indices'] }); // returns 2

obj = { "0": "index", "-1": "string key", "01": "string key" };
Object.propertyCount(obj, { keyTypes: ['indices'] }); // returns 1 (only "0")
```

- **String based keys**:

```javascript
const obj = { "01": "string key", 1: "index", 2: "index" };
Object.propertyCount(obj, { keyTypes: ['string'] }); // returns 1
```

## Explicit Semantics

- Only own properties are considered.
- Enumerability explicitly defined by the `enumerable` parameter.
- Avoids intermediate array allocation entirely when implemented natively.

## Algorithmic Specification

The native implementation should strictly avoid creating intermediate arrays or unnecessary allocations:

1. Initialize a numeric property counter to `0`.
2. Iterate directly over the object's own property descriptors
   - Access the internal property keys directly via the object's internal slots.
   - For each own property:
     - Determine if the key is a numeric index, a regular string, or a symbol.
     - Check if the property type matches any specified in `keyTypes`.
     - If `enumerable` is not `'all'`, match the property's enumerability against the provided boolean value.
     - If the property meets all criteria, increment the counter.
3. Return the final count value.

### Object.propertyCount ( _target_ [, _options_ ] )

When the `propertyCount` method is called, the following steps are taken:

**This is work in progress! The Algorithm is not yet fully defined.**

1. If Type(_target_) is not Object, throw a TypeError exception.
2. If _options_ is undefined, let _options_ be an empty Object.
3. Let _keyTypes_ be ? Get(_options_, "keyTypes").
4. If _keyTypes_ is undefined, set _keyTypes_ to the array `['indices', 'string']`.
5. Else, perform the following:
    a. If Type(_keyTypes_) is not Object, throw a TypeError exception.
    b. Set _keyTypes_ to an internal List whose elements are the String values of the elements of _keyTypes_.
    c. If _keyTypes_ contains any value other than "indices", "string", or "symbol", throw a TypeError exception.
6. Let _enumerable_ be ? Get(_options_, "enumerable").
7. If _enumerable_ is undefined, set _enumerable_ to true.
8. Else if _enumerable_ is not one of true, false, or "all", throw a TypeError exception.
9.  Let _count_ be 0.
10. Let _ownKeys_ be the List of own property keys of _target_, in the order returned by OrdinaryOwnPropertyKeys(_target_). (Note: No intermediate array should be allocated.)
11. For each element _key_ of _ownKeys_, perform the following steps:
    a. Let _desc_ be ? OrdinaryGetOwnProperty(_target_, _key_).
    b. If _enumerable_ is not 'all'
        i. If _enumerable_ is true and _desc_.[[Enumerable]] is false, continue to the next _key_.
        ii. If _enumerable_ is false and _desc_.[[Enumerable]] is true, continue to the next _key_.
    c. If Type(_key_) is Symbol and "symbol" is present in _keyTypes_, increment _count_ by 1.
    d. Else if Type(_key_) is String, perform:
        i. Let _numericIndex_ be ! CanonicalNumericIndexString(_key_).
        ii. If _numericIndex_ is not undefined and "indices" is present in _keyTypes_, increment _count_ by 1.
        iii. Else if _numericIndex_ is undefined and "string" is present in _keyTypes_, increment _count_ by 1.
12. Return _count_.

## Alternatives Considered

- **Multiple separate methods**: Rejected due to increased cognitive load and API complexity.

## TC39 Stages and Champion

- Ready for **Stage 1** (proposal)
- Champion(s) and community stakeholders pending

## Use Cases

- Improved readability and explicit intent
- Significant performance gains
- Reduced memory overhead
- Simpler code

## Precedent

Frequent patterns in widely-used JavaScript runtimes, frameworks, and libraries (Node.js, React, Angular, Lodash) demonstrate the common need for an optimized property counting mechanism.

## Polyfill

```javascript
const validTypes = new Set(['indices', 'string', 'symbol']);

Object.propertyCount = function(target, options = {}) {
  const { keyTypes = ['indices', 'string'], enumerable = true } = options;

  for (const type of keyTypes) {
    if (!validTypes.has(type)) {
      throw new TypeError(`Invalid property type (${type}) in 'keyTypes' option.`);
    }
  }

  if (typeof enumerable !== 'boolean' && enumerable !== 'all') {
    throw new TypeError(`Invalid input (${enumerable}) in 'enumerable' option.`);
  }

  let props = [];

  if (keyTypes.includes('indices') || keyTypes.includes('string')) {
    let stringProps = Object.getOwnPropertyNames(target);

    if (!keyTypes.includes('string')) {
      stringProps = stringProps.filter(key => String(parseInt(key, 10)) === key && parseInt(key, 10) >= 0);
    }

    if (!keyTypes.includes('indices')) {
      stringProps = stringProps.filter(key => String(parseInt(key, 10)) !== key || parseInt(key, 10) < 0);
    }

    props = stringProps;
  }

  if (keyTypes.includes('symbol')) {
    props = props.concat(Object.getOwnPropertySymbols(target));
  }

  if (enumerable !== 'all') {
    props = props.filter(key => Object.getOwnPropertyDescriptor(target, key).enumerable === enumerable);
  }

  return props.length;
};
```

## Considerations

- **Backwards compatibility**: Fully backward compatible.
- **Performance**: Native implementation will significantly outperform existing approaches by eliminating intermediate arrays.
- **Flexibility**: Enumerable properties counted by default; easy inclusion/exclusion.
- **Simplicity**: Improved code readability and clarity.

## Conclusion

`Object.propertyCount` offers substantial performance benefits by efficiently counting object properties without intermediate arrays, enhancing ECMAScript with clarity, performance, and reduced memory overhead.