# JSON.parse source text access

![Stage 2](https://badges.aleen42.com/src/tc39_3.svg)

A proposal for extending `JSON.parse` behavior to grant reviver functions access to the input source text.

[2018 September slides](https://docs.google.com/presentation/d/1PB0HCOxWZikFmTAqR5U2ZZjEiDV7NjhPN_-SK5NNG0w/edit?usp=sharing)

## Status
This proposal is at stage 2 of [the TC39 Process](https://tc39.github.io/process-document/).

## Champions
* Richard Gibson
* Mathias Bynens

## Motivation
Transformation between ECMAScript values and JSON text is lossy.
This is most obvious in the case of deserializing numbers (e.g., `"999999999999999999"`, `"999999999999999999.0"`, and `"1000000000000000000"` all parse to `1000000000000000000`), but also comes up when attempting to round-tripping non-primitive values such as Date objects (e.g., `JSON.parse(JSON.stringify(new Date("2018-09-25T14:00:00Z")))` yields a string `"2018-09-25T14:00:00.000Z"`).

Neither of these examples is hypothetical—serializing a [BigInt](https://github.com/tc39/proposal-bigint) as JSON is specified to throw an exception because there is no output that would round-trip through `JSON.parse`, and a similar concept has been raised regarding the [Temporal proposal](https://github.com/tc39/proposal-temporal).

`JSON.parse` accepts a reviver function capable of processing inbound values, but it is invoked bottom-up and receives so little context (a _key_, an already-lossy _value_, and a receiver upon which _key_ is an own property with value _value_) that it is practically useless.
The authors intend to remedy that.

## Proposed Solution
Update `JSON.parse` to provide reviver functions with more arguments, primarily conveying the source text from which a value was derived (inclusive of punctuation but exclusive of leading/trailing insignificant whitespace).

## Illustrative examples
```js
const tooBigForNumber = BigInt(Number.MAX_SAFE_INTEGER) + 2n;
const intToBigInt = (key, val, {source}) => typeof val === "number" && val % 1 === 0 ? BigInt(source) : val;
const roundTripped = JSON.parse(String(tooBigForNumber), intToBigInt);
tooBigForNumber === roundTripped;
// → true
```

### Potential enhancements
#### Expose position and input information
`String.prototype.replace` passes position and input arguments to replacer functions and the return value from `RegExp.prototype.exec` has "index" and "input" properties; `JSON.parse` could behave similarly.
```js
const input = '\n\t"use\\u0020strict"';
let spied;
const parsed = JSON.parse(input, (key, val, context) => (spied = context, val));
parsed === 'use strict';
// → true
spied.source === '"use\\u0020strict"';
// → true
spied.index === 2;
// → true
spied.input === input;
// → true

```

#### Supply an array of keys for understanding value context
A reviver function sees values bottom-up, but the data structure hierarchy is already known and can be supplied to it, with or without the phantom leading empty string.
```js
const input = '{ "foo": [{ "bar": "baz" }] }';
const expectedKeys = ['foo', 0, 'bar'];
let spiedKeys;
JSON.parse(input, (key, val, {keys}) => (spiedKeys = spiedKeys || keys, val));
expectedKeys.length === spiedKeys.length;
// → true
expectedKeys.every((key, i) => spiedKeys[i] === key);
// → true
```

## Discussion
### Backwards Compatibility
Conforming ECMAScript implementations are not permitted to extend the grammar accepted by `JSON.parse`.
This proposal does not make any such attempt, and adding new function parameters is among the safest changes that can be made to the language.
All input to `JSON.parse` that is currently rejected will continue to be, all input that is currently accepted will continue to be, and the only changes to its output will be directly controlled by user code.

### Modified values
Reviver functions are intended to modify or remove values in the output, but those changes should have no effect on the source-derived arguments passed to them.
Because reviver functions are invoked bottom-up, this means that values may not correlate with source text.
The authors consider this to be acceptable.

## Specification
Not defined yet, and to be honest doing so may be difficult.
Help will be graciously accepted.
