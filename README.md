# ECMAScript Proposal: `Object.propertyCount`

## Status

Champion: Jordan Harband

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

The implementation in V8 seems quite straight forward, due to the already existing [GetPropertyNames](https://github.com/v8/v8/blob/2b13b925298112a1366f721c8d30c96b8b61aeae/include/v8-object.h#L402-L405) API.

### Concrete usage examples

I only searched for `Object.keys().length`, since that is the most common one.

#### Angular

- https://github.com/angular/angular/blob/7499b74d7d2d6db132d1b19a73e13cf6e306e41e/packages/router/src/url_tree.ts#L119
- https://github.com/angular/angular/blob/7499b74d7d2d6db132d1b19a73e13cf6e306e41e/packages/core/src/transfer_state.ts#L120
- https://github.com/angular/angular/blob/7499b74d7d2d6db132d1b19a73e13cf6e306e41e/packages/compiler/src/render3/view/i18n/util.ts#L57
- https://github.com/angular/angular/blob/7499b74d7d2d6db132d1b19a73e13cf6e306e41e/packages/core/src/util/ng_dev_mode.ts#L97

And multiple more.

#### React

- https://github.com/facebook/react/blob/254dc4d9f37eb512d4ee8bad6a0fae7ae491caef/packages/shared/shallowEqual.js#L33C9-L35
- https://github.com/facebook/react/blob/254dc4d9f37eb512d4ee8bad6a0fae7ae491caef/packages/react-devtools-shared/src/hydration.js#L97
- https://github.com/facebook/react/blob/254dc4d9f37eb512d4ee8bad6a0fae7ae491caef/packages/react-reconciler/src/ReactFiberHydrationDiffs.js#L416-L417
- https://github.com/facebook/react/blob/254dc4d9f37eb512d4ee8bad6a0fae7ae491caef/packages/react-reconciler/src/ReactFiber.js#L677
- https://github.com/facebook/react/blob/254dc4d9f37eb512d4ee8bad6a0fae7ae491caef/packages/react-server/src/ReactFizzServer.js#L2384
- https://github.com/facebook/react/blob/254dc4d9f37eb512d4ee8bad6a0fae7ae491caef/packages/react-dom-bindings/src/client/ReactDOMComponent.js#L3138

#### Node.js

- https://github.com/nodejs/node/blob/c3b6f949748b49ef25b0239bd4582d29976fdbad/lib/internal/util/comparisons.js#L385
- https://github.com/nodejs/node/blob/c3b6f949748b49ef25b0239bd4582d29976fdbad/lib/internal/util/comparisons.js#L754-L763
- https://github.com/nodejs/node/blob/c3b6f949748b49ef25b0239bd4582d29976fdbad/lib/internal/util/comparisons.js#L713-L720 (could be rewritten in a more performant way with the new API)
- https://github.com/nodejs/node/blob/c3b6f949748b49ef25b0239bd4582d29976fdbad/lib/internal/debugger/inspect_client.js#L248
- https://github.com/nodejs/node/blob/c3b6f949748b49ef25b0239bd4582d29976fdbad/lib/internal/cluster/primary.js#L146
- https://github.com/nodejs/node/blob/c3b6f949748b49ef25b0239bd4582d29976fdbad/lib/internal/console/constructor.js#L537

#### Minimatch

https://github.com/isaacs/minimatch/blob/0569cd3373408f9d701d3aab187b3f43a24a0db7/src/index.ts#L158

#### Vue

- https://github.com/vuejs/core/blob/d65b25cdda4c0e7fe8b51e000ecc3696baad0492/packages/shared/src/looseEqual.ts#L36C11-L37
- https://github.com/vuejs/core/blob/d65b25cdda4c0e7fe8b51e000ecc3696baad0492/rollup.config.js#L251
- https://github.com/vuejs/core/blob/d65b25cdda4c0e7fe8b51e000ecc3696baad0492/packages/compiler-core/src/utils.ts#L505
- https://github.com/vuejs/core/blob/d65b25cdda4c0e7fe8b51e000ecc3696baad0492/packages/runtime-core/src/customFormatter.ts#L123
- https://github.com/vuejs/core/blob/d65b25cdda4c0e7fe8b51e000ecc3696baad0492/packages/compiler-sfc/src/style/pluginScoped.ts#L33
- https://github.com/vuejs/core/blob/d65b25cdda4c0e7fe8b51e000ecc3696baad0492/packages/compiler-core/src/transforms/transformElement.ts#L905

#### Lodash

Lodash uses an own implementation that behaves as Object.keys()

- https://github.com/lodash/lodash/blob/8a26eb42adb303f4adc7ef56e300f14c5992aa68/dist/lodash.js#L9921
- https://github.com/lodash/lodash/blob/8a26eb42adb303f4adc7ef56e300f14c5992aa68/dist/lodash.js#L11561

#### Other popular ones

Almost all popular JS/TS modules make use of this pattern.

- https://github.com/trekhleb/javascript-algorithms/blob/e40a67b5d1aaf006622a90e2bda60043f4f66679/src/algorithms/graph/detect-cycle/detectDirectedCycle.js#L83
- https://github.com/storybookjs/storybook/blob/b91e25a25c8c1cc77ea6b316d03b4cce183d815c/code/core/src/theming/ensure.ts#L15
- https://github.com/storybookjs/storybook/blob/b91e25a25c8c1cc77ea6b316d03b4cce183d815c/code/core/src/theming/ensure.ts#L15
- https://github.com/storybookjs/storybook/blob/b91e25a25c8c1cc77ea6b316d03b4cce183d815c/code/lib/blocks/src/blocks/Controls.tsx#L60-L63
- https://github.com/storybookjs/storybook/blob/b91e25a25c8c1cc77ea6b316d03b4cce183d815c/scripts/sandbox/templates/root.ejs#L7
- https://github.com/storybookjs/storybook/blob/b91e25a25c8c1cc77ea6b316d03b4cce183d815c/code/lib/blocks/src/blocks/DocsPage.tsx#L15
- https://github.com/storybookjs/storybook/blob/b91e25a25c8c1cc77ea6b316d03b4cce183d815c/code/core/assets/server/template.ejs#L67
- https://github.com/tailwindlabs/tailwindcss/blob/e8715d081eac683d002892b8b3e13550f0276b45/packages/tailwindcss/src/compat/theme-variants.ts#L9
- https://github.com/tailwindlabs/tailwindcss/blob/e8715d081eac683d002892b8b3e13550f0276b45/packages/%40tailwindcss-upgrade/src/migrate-postcss.ts#L346
- https://github.com/tailwindlabs/tailwindcss/blob/e8715d081eac683d002892b8b3e13550f0276b45/packages/tailwindcss/src/compat/apply-compat-hooks.ts#L100
- https://github.com/puppeteer/puppeteer/blob/ff74c58464f985253b0a986f5fbbe4edc1658a42/packages/puppeteer-core/src/bidi/HTTPRequest.ts#L149
- https://github.com/puppeteer/puppeteer/blob/ff74c58464f985253b0a986f5fbbe4edc1658a42/packages/puppeteer-core/src/bidi/Page.ts#L623
- https://github.com/excalidraw/excalidraw/blob/e1bb59fb8f115cd8e75fcaaeefa03a81b0fdc697/packages/excalidraw/actions/actionSelectAll.ts#L50
- https://github.com/excalidraw/excalidraw/blob/e1bb59fb8f115cd8e75fcaaeefa03a81b0fdc697/packages/excalidraw/change.ts#L133
- https://github.com/excalidraw/excalidraw/blob/e1bb59fb8f115cd8e75fcaaeefa03a81b0fdc697/packages/excalidraw/groups.ts#L38
- https://github.com/excalidraw/excalidraw/blob/e1bb59fb8f115cd8e75fcaaeefa03a81b0fdc697/packages/excalidraw/components/App.tsx#L2760
- https://github.com/excalidraw/excalidraw/blob/e1bb59fb8f115cd8e75fcaaeefa03a81b0fdc697/packages/excalidraw/actions/actionProperties.tsx#L1038
- https://github.com/excalidraw/excalidraw/blob/e1bb59fb8f115cd8e75fcaaeefa03a81b0fdc697/packages/excalidraw/clipboard.ts#L317
- https://github.com/excalidraw/excalidraw/blob/e1bb59fb8f115cd8e75fcaaeefa03a81b0fdc697/packages/excalidraw/element/mutateElement.ts#L44
- https://github.com/microsoft/vscode/blob/2e6728cc3b6ab7f2bc5223dd52abb5f3b595b827/src/vs/base/common/equals.ts#L87-L91
- https://github.com/microsoft/vscode/blob/2e6728cc3b6ab7f2bc5223dd52abb5f3b595b827/src/vs/platform/policy/common/policy.ts#L37
- https://github.com/microsoft/vscode/blob/2e6728cc3b6ab7f2bc5223dd52abb5f3b595b827/build/lib/util.ts#L37
- https://github.com/microsoft/vscode/blob/2e6728cc3b6ab7f2bc5223dd52abb5f3b595b827/src/vs/platform/policy/node/nativePolicyService.ts#L26
- https://github.com/microsoft/vscode/blob/2e6728cc3b6ab7f2bc5223dd52abb5f3b595b827/extensions/terminal-suggest/src/fig/shared/utils.ts#L165
- https://github.com/microsoft/vscode/blob/2e6728cc3b6ab7f2bc5223dd52abb5f3b595b827/build/lib/i18n.ts#L90
- https://github.com/microsoft/vscode/blob/2e6728cc3b6ab7f2bc5223dd52abb5f3b595b827/src/vs/platform/product/common/product.ts#L61
- https://github.com/sveltejs/svelte/blob/f498a21063894e6e515e62d753396410624b2e0f/packages/svelte/src/compiler/phases/3-transform/client/visitors/shared/component.js#L259
- https://github.com/mrdoob/three.js/blob/b0805c2a0fd46c137d605ba098bc2b17d507b46f/src/materials/ShaderMaterial.js#L359
- https://github.com/mrdoob/three.js/blob/b0805c2a0fd46c137d605ba098bc2b17d507b46f/editor/js/Sidebar.Geometry.BufferGeometry.js#L60
- https://github.com/mrdoob/three.js/blob/b0805c2a0fd46c137d605ba098bc2b17d507b46f/examples/jsm/loaders/3MFLoader.js#L239
- https://github.com/vercel/next.js/blob/5c5875105af06e17ccad4080a6ace137f14cdabb/packages/next/src/build/babel/loader/get-config.ts#L177
- https://github.com/vercel/next.js/blob/5c5875105af06e17ccad4080a6ace137f14cdabb/turbopack/packages/devlow-bench/src/cli.ts#L34
- https://github.com/vercel/next.js/blob/5c5875105af06e17ccad4080a6ace137f14cdabb/packages/next/check-error-codes.js#L31
- https://github.com/vercel/next.js/blob/5c5875105af06e17ccad4080a6ace137f14cdabb/scripts/trace-dd.mjs#L67
- https://github.com/vercel/next.js/blob/5c5875105af06e17ccad4080a6ace137f14cdabb/packages/next/src/server/lib/utils.ts#L243
- https://github.com/vercel/next.js/blob/5c5875105af06e17ccad4080a6ace137f14cdabb/scripts/trace-to-tree.mjs#L147

## Problem Statement

Currently, accurately counting object properties involves verbose and inefficient workarounds:

```javascript
const count = [
  ...Object.getOwnPropertyNames(obj),
  ...Object.getOwnPropertySymbols(obj)
].length;

const reflectCount = Reflect.ownKeys(obj);

assert.strictEqual(count, reflectCount);
```

This creates intermediate arrays, causing unnecessary memory usage and garbage collection, impacting application performance — especially at scale and in performance-critical code paths.

On top of that, it is also not possible to identify an array that is sparse without calling `Object.keys()` (or similar). This API would allow that by explicitly checking for own index properties.

## Proposed API

```javascript
Object.propertyCount(target[, options])
```

### Parameters

- **`target`**: The object whose properties will be counted.
  - Throws `TypeError` if target is not an object.
- **`options`** *(optional)*: An object specifying filtering criteria:
  - `keyTypes`: Array specifying property types to include:
    - Possible values: `'index'`, `'nonIndexString'`, `'symbol'`.
    - Defaults to `['index', 'nonIndexString']` (aligning closely with `Object.keys`).
    - Throws `TypeError` if provided invalid values.
  - `enumerable`: Indicates property enumerability:
    - `true` to count only enumerable properties (default).
    - `false` to count only non-enumerable properties.
    - `'all'` to count both enumerable and non-enumerable properties.
    - Throws `TypeError` if provided invalid values.

Defaults align closely with `Object.keys` for ease of adoption, ensuring intuitive behavior without needing explicit configuration in common cases.

The naming of keyTypes and if it's an array or an object or the like is open for discussion.
Important is just, that it's possible to differentiate index from non index strings somehow, as well as symbol properties.

Similar applies to the enumerable option: true, false, and "all" seems cleanest, but it's not important how they are named.

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

```javascript
Object.propertyCount({ __proto__: null }); // returns 0
```

- **Array index keys**:

See https://tc39.es/ecma262/#array-index

```javascript
let obj = { "01": "string key", 1: "index", 2: "index" };
Object.propertyCount(obj, { keyTypes: ['index'] }); // returns 2

obj = { "0": "index", "-1": "string key", "01": "string key" };
Object.propertyCount(obj, { keyTypes: ['index'] }); // returns 1 (only "0")
```

- **String based keys**:

```javascript
const obj = { "01": "string key", 1: "index", 2: "index" };
Object.propertyCount(obj, { keyTypes: ['nonIndexString'] }); // returns 1
```

- **Symbol based keys**:

```javascript
const obj = { [Symbol()]: "symbol", 1: "index", 2: "index" };
Object.propertyCount(obj, { keyTypes: ['symbol'] }); // returns 1
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
     - Determine if the key is a numeric index, a regular non-index string, or a symbol.
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
4. If _keyTypes_ is undefined, set _keyTypes_ to the array `['index', 'nonIndexString']`.
5. Else, perform the following:
    1. If Type(_keyTypes_) is not Object, throw a TypeError exception.
    2. Set _keyTypes_ to an internal List whose elements are the String values of the elements of _keyTypes_.
    3. If _keyTypes_ contains any value other than "index", "nonIndexString", or "symbol", throw a TypeError exception.
1. Let _enumerable_ be ? Get(_options_, "enumerable").
2. If _enumerable_ is undefined, set _enumerable_ to true.
3. Else if _enumerable_ is not one of true, false, or "all", throw a TypeError exception.
4.  Let _count_ be 0.
5.  Let _ownKeys_ be the List of own property keys of _target_, in the order returned by OrdinaryOwnPropertyKeys(_target_). (Note: No intermediate array should be allocated.)
6.  For each element _key_ of _ownKeys_, perform the following steps:
    1. Let _desc_ be ? OrdinaryGetOwnProperty(_target_, _key_).
    2. If _enumerable_ is not 'all'
        i. If _enumerable_ is unequal to _desc_.[[Enumerable]], continue to the next _key_.
    3. If Type(_key_) is Symbol and "symbol" is present in _keyTypes_, increment _count_ by 1.
    4. Else if Type(_key_) is array index and "index" is present in _keyTypes_, increment _count_ by 1.
    5. Else if "nonIndexString" is present in _keyTypes_, increment _count_ by 1
7.  Return _count_.

## Alternatives Considered

- **Multiple separate methods**: Rejected due to increased cognitive load and API complexity.

## TC39 Stages and Champion

- Ready for **Stage 1** (proposal)

## Use Cases

- Improved readability and explicit intent
- Significant performance gains
- Reduced memory overhead
- Simpler code

## Precedent

Frequent patterns in widely-used JavaScript runtimes, frameworks, and libraries (Node.js, React, Angular, Lodash) demonstrate the common need for an optimized property counting mechanism.

## Polyfill

```javascript
const validTypes = new Set(['index', 'nonIndexString', 'symbol']);

Object.propertyCount = function(target, options = {}) {
  const { keyTypes = ['index', 'nonIndexString'], enumerable = true } = options;

  for (const type of keyTypes) {
    if (!validTypes.has(type)) {
      throw new TypeError(`Invalid property type (${type}) in 'keyTypes' option.`);
    }
  }

  if (typeof enumerable !== 'boolean' && enumerable !== 'all') {
    throw new TypeError(`Invalid input (${enumerable}) in 'enumerable' option.`);
  }

  let props = [];

  if (keyTypes.includes('index') || keyTypes.includes('nonIndexString')) {
    let stringProps = Object.getOwnPropertyNames(target);

    if (!keyTypes.includes('nonIndexString')) {
      stringProps = stringProps.filter(key => String(parseInt(key, 10)) === key && parseInt(key, 10) >= 0);
    }

    if (!keyTypes.includes('index')) {
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
