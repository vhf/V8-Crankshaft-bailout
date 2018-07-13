# V8 bailout/deopts reasons

A list of V8 bailout/deopts reasons with examples, explanations and advices. Includes both the former optimizing compiler Crankshaft and the current compilation infrastructure Ignition + [TurboFan](https://github.com/v8/v8/wiki/TurboFan).


Unless otherwise specified, the following are Crankshaft bailouts.

### Warning!

Starting from Chrome 59 and [Node.js 8.3.0](https://medium.com/the-node-js-collection/node-js-8-3-0-is-now-available-shipping-with-the-ignition-turbofan-execution-pipeline-aa5875ad3367), **Crankshaft is not used anymore**. It uses [TurboFan](https://github.com/v8/v8/wiki/TurboFan) instead.

**The content of this repository only applies to Crankshaft**. If the JavaScript engine you are targeting does not use Crankshaft as its optimizing compiler, you should not care about this repository and the advices present in there. This repository will stay here for historical reasons, it still has documentation value.

### What this is about

In order to keep this section short and allow people to get to the primary content of this repo faster, here is what it's all about and why you (probably) should care if you are using Crankshaft: [Chromium, Chrome, Node.js, V8, Crankshaft and bailout reasons](https://draft.li/blog/2016/01/22/chromium-chrome-v8-crankshaft-bailout-reasons/).

## Index
### [Bailout reasons](#bailout-reasons-1)

* [Assignment to parameter in arguments object](#assignment-to-parameter-in-arguments-object)
* [Bad value context for arguments value](#bad-value-context-for-arguments-value)
* [ForInStatement with non-local each variable](#forinstatement-with-non-local-each-variable)
* [Inlining bailed out](#inlining-bailed-out)
* [Object literal with complex property](#object-literal-with-complex-property)
* [Optimized too many times](#optimized-too-many-times)
* [Reference to a variable which requires dynamic lookup](#reference-to-a-variable-which-requires-dynamic-lookup)
* [Rest parameters](#rest-parameters)
* [Smi addition overflow](#smi-addition-overflow)
* [Smi subtraction overflow](#smi-subtraction-overflow)
* [Too many parameters](#too-many-parameters)
* [TryCatchStatement](#trycatchstatement)
* [TryFinallyStatement](#tryfinallystatement)
* [Unsupported phi use of arguments](#unsupported-phi-use-of-arguments)
* [Unsupported phi use of const or let variable](#unsupported-phi-use-of-const-or-let-variable)
* [Yield](#yield)

### [References](#references-1)

* [Resources](#resources)
* [All bailout reasons](#all-bailout-reasons)

## Deoptimization reasons

### WrongMap
One of the most common deopt reasons. It happens when the shape of passed object doesn't match.
More about that [here](http://ripsawridge.github.io/articles/stack-changes/)

* Simple reproduction(s)

```js
function getFoo(obj) {
  return obj.foo;
}

getFoo({ foo: 'bar' });
getFoo({ foo: 'bar' });
%OptimizeFunctionOnNextCall(getFoo);
getFoo({ foo: 'bar', test: 'yaay' });
```

```js
function getFirstItem(arr) {
  return arr[0];
}

getFirstItem([0, 1]);
getFirstItem([0, 1, 2, 3, 4]);
%OptimizeFunctionOnNextCall(getFirstItem);
getFirstItem(['boo']);
```

* Why
  * Collected feedback is invalidated as the object met is simply unknown and the past assumptions are likely to be obsolete.
  * More about that [here](http://ripsawridge.github.io/articles/stack-changes/)

* Advices
  * Try to sort properties in objects in the alphabetical order (yes, order of properties does matter)
  * Kinds can be found [here](https://github.com/v8/v8/blob/46a5d96bf74d648e84d92f2e1cfff788e522503b/src/elements-kind.h)


### Smi
Only happens when your function is optimized and you don't expect SMI to be passed.

* Simple reproduction(s)

```
function concat(str1, str2) {
 return str1 + str2;
}

concat('2', '3');
concat('10', '3');
%OptimizeFunctionOnNextCall(concat);
concat('2', 3);
```

* Why

* Advices
  * Keep your function statically typed and don't pass Smi if you don't have to. You can pass String(3) instead.

### Not a Smi

* Simple reproduction(s)

```
function add(a, b) {
 return a + b;
}

add(1, 2);
add(4, 5);
%OptimizeFunctionOnNextCall(add);
add('2', 3);
```
* Why

* Advices
  * Keep your function statically typed (there is info about Smi within this README). You can pass Number(3) instead.


### Not a heap number/undefined

* Simple reproduction(s)
```
function add(a) {
  return a + 1;
}

add(2 ** 32);
add(2 ** 32);
%OptimizeFunctionOnNextCall(add);
add('d');
```
* Why

* Advices

## Bailout reasons
### Assignment to parameter in arguments object

Only happens if you reassign to a parameter while also mentioning `arguments` in the function. [More info](https://github.com/petkaantonov/bluebird/wiki/Optimization-killers#31-reassigning-a-defined-parameter-while-also-mentioning-arguments-in-the-body-in-sloppy-mode-only-typical-example).

* Simple reproduction(s)

```js
// sloppy mode only
function test(a) {
  if (arguments.length < 2) {
    a = 0;
  }
}
```

* Why
  * In sloppy mode V8 needs to preserve bindings between `arguments[0]` and `a` so that when any `a` was passed, and you reassign it to `10`, and later try to read `arguments[0]`, it has to return `10`, too. This is non-trivial for the engine, so it chooses to bail out. Strict mode removes this requirement, and `arguments` and `a` behave as regular independent JavaScript variables, so deoptimization does not occur.

* Advices
  * In the above example, you could assign `a` to a new variable.
  * You should use strict mode anyway.
  * It seems this will be optimized by TurboFan [#1][1].

* External examples


### Bad value context for arguments value

* Simple reproduction(s)

```js
// strict & sloppy modes
function test1() {
  arguments[0] = 0;
}

// strict & sloppy modes
function test2() {
  arguments.length = 0;
}

// strict & sloppy modes
function test3() {
  return arguments;
}

// strict & sloppy modes
function test4() {
  var args = [].slice.call(arguments);
}

// strict & sloppy modes
function test5() {
  var a = arguments;
  return function() {
    return a;
  };
}
```

* Why
  * It requires rematerialization of the `arguments` array.

* Advices
  * Read this: https://github.com/petkaantonov/bluebird/wiki/Optimization-killers#3-managing-arguments
  * You could loop over `arguments` to build a new array, but it's not recommended. See [Unsupported phi use of arguments](#unsupported-phi-use-of-arguments)
  * Usages of `arguments` as shown above are very rarely legitimate.
  * [More about this bailout reason][7]
  * It seems this will be optimized by TurboFan [#1][1].

* External examples
  * https://github.com/bevry/taskgroup/issues/12
  * https://github.com/babel/babel/pull/3249


### ForInStatement with non-local each variable

* Simple reproduction(s)

```js
// strict & sloppy modes
function test1() {
  var obj = {};
  for(key in obj);
}

// strict & sloppy modes
function key() {
  return 'a';
}
function test2() {
  var obj = {};
  for(key in obj);
}
```

* Why

* Advices
  * Only use pure (i.e. non-computed) local variable in a for...in.
  * https://github.com/petkaantonov/bluebird/wiki/Optimization-killers#5-for-in

* External examples
  * https://github.com/mbostock/d3/pull/2686


### Inlining bailed out

* Simple reproduction(s)

[Courtesy of @kangax](https://github.com/GoogleChrome/devtools-docs/issues/53#issuecomment-32784608)

```js
// strict & sloppy modes
var obj = { prop1: ... };

function test(param) {
  param.prop2 = ...; // Inlining bailed out
}
test(obj);

// strict & sloppy modes
var obj = { prop1: ... };

function someMethodThatAssignsSomeOtherProp(param) {
  param.someOtherProp = ...; // Inlining bailed out
}

function test(param) {
  someMethodThatAssignsSomeOtherProp(param);
}

f(obj);
```

* Why
  * Crankshaft predicts that `param` will have the same hidden class as `obj`, allowing optimization by doing inline caching. At the annotated lines above, Crankshaft notices that `obj`'s hidden class is not suitable for `param` and bails out (it cannot patch the inline cache code with the hidden class information). [More about inline caching and hidden classes][8]

* Advices
  * When creating an object initialize all the properties you are going to use, instead of adding new properties to an existing object.

* External examples


### Object literal with complex property

* Simple reproduction(s)

```js
// strict & sloppy modes
function test() {
  return {
    __proto__: 3
  };
}
```

* Why

* Advices

* External examples


### Optimized too many times

* Simple reproduction(s)

```js
// strict & sloppy modes
// No known canonical reproduction
```

* Why
  * Optimization failed so many times that Crankshaft gave up.
  * "In reality this very often actually means a bug in V8 - there is some optimization which is too optimistic so the generated code deopts all the time." - @mraleph [#5][5]
  * [More about this bailout reason][6]

* Advices
  * "Just use IRHydra and look at the deoptimization reasons - the picture should become clear immediately." - @mraleph [#5][5]

* External examples


### Reference to a variable which requires dynamic lookup

* Simple reproduction(s)

```js
// sloppy mode only
function test() {
  with ({x:1}) {
    return x;
  }
}
```

* Why
  * "Variable lookup fails at compile time, Crankshaft needs to resort to dynamic lookup at runtime." - Yang Guo [#3][3]

* Advices
  * "Refactor to remove the dependency on runtime-information to resolve the lookup." - Paul Irish [#4][4]
  * **No bailout with TurboFan.**

* External examples


### Rest parameters

* Simple reproduction(s)

```js
// strict & sloppy modes
function test(...rest) {
  return rest[0];
}
```

* Why
  * Probably because it requires materializing the `arguments` array.

* Advices
  * Avoid rest parameters or use Babel's [transform-es2015-parameters](http://babeljs.io/docs/plugins/transform-es2015-parameters/) until TurboFan is able to optimize them [#1][1], [#2][2].

* External examples

### Smi addition overflow
* Simple reproduction(s)
Smi - A Smi is a 32-bit signed int on 64-bit architectures and a 31-bit signed int on 32-bit architectures.

Smi addition overflow
```js
function add(a, b) {
  return a + b;
}

add(1, 2);
add(1, 2);
%OptimizeFunctionOnNextCall(add);
add(2 ** 31 - 2, 20);
```

* Why
    * Once addition is performed, the return value cannot be represented as a Smi and thus is casted to [HeapNumber](https://github.com/v8/v8/blob/master/src/objects.h#L1838)
* Advices
    * You could have 2 separate functions - one for Smis and one for HeapNumbers.
* External examples

### Smi subtraction overflow
* Simple reproduction(s)
Same case as with Smi addition overflow
Smi addition overflow
```js
function subtract(a, b) {
  return a - b;
}

subtract(1, 2);
subtract(1, 2);
%OptimizeFunctionOnNextCall(subtract);
subtract(-3, 2 ** 31 - 1);
```

* Why
    * Once subtraction is performed, the return value cannot be represented as a Smi and thus is casted to [HeapNumber](https://github.com/v8/v8/blob/master/src/objects.h#L1838)
* Advices
    * You could have 2 separate functions - one for Smis and one for HeapNumbers.
* External examples

### Too many parameters

* Simple reproduction(s)

```js
// strict & sloppy modes
function test(p1, p2, p3, ..., p512) {
}
```

* Why
  * Setting limits.

* Advices
  * If you write functions with 512 parameters or more, you probably don't worry about optimizing your code for V8 anyway.

* External examples
  * Obviously nobody ever did that. Hopefully nobody will ever do that. Zero google result on this bailout reason.
  * [V8 code source](https://chromium.googlesource.com/v8/v8/+/fe0fe20e8f094d5688256583abc5695243c6759d%5E%21/#F2)


### TryCatchStatement

* Simple reproduction(s)

```js
// strict & sloppy modes
function test() {
  return 3;
  try {} catch(e) {}
}
```

* Why
  * Try/catch makes the control flow jump virtually anywhere. It's hardly optimizable because the caught exception is potentially only known at runtime.

* Advices
  * Don't put try/catch inside computationally intensive functions.
  * You could `try { test() } catch`

* External examples


### TryFinallyStatement

* Simple reproduction(s)

```js
// strict & sloppy modes
function test() {
  return 3;
  try {} finally {}
}
```

* Why
  * See [TryCatchStatement](#trycatchstatement)

* Advices
  * See [TryCatchStatement](#trycatchstatement)

* External example


### Unsupported phi use of arguments

* Simple reproduction(s)

```js
// strict & sloppy modes
function test1() {
  var _arguments = arguments;
  if (0 === 0) { // anything evaluating to true, except a number or `true`
    _arguments = [0]; // Unsupported phi use of arguments
  }
}

// strict & sloppy modes
function test2() {
  var _arguments = arguments;
  for (var i = 0; i < 1; i++) {
    _arguments = [0]; // Unsupported phi use of arguments
  }
}

// strict & sloppy modes
function test3() {
  var _arguments = arguments;
  var again = true;
  while (again) {
    _arguments = [0]; // Unsupported phi use of arguments
    again = false;
  }
}
```

* Why
  * Crankshaft is unable to guess whether `_arguments` should be an object or an array. It cannot dematerialize `_arguments` and gives up.
  * [In-depth explaination](http://mrale.ph/blog/2015/11/02/crankshaft-vs-arguments-object.html)

* Advices
  * There is no good workaround except splitting your function into smaller ones that don't manipulate a copy of `arguments`.
  * Don't try to fool V8 by looping over `arguments` to create a new array out of it: "Allocating array (and hope it will get handled by some optimization pass in the V8) is a bad idea." - [@mraleph](https://github.com/mraleph) ([source](https://draft.li/blog/2015/11/02/javascript-performance-with-babel-and-node-js/))
  * It seems this will be optimized by TurboFan [#1][1].

* External examples


### Unsupported phi use of const or let variable

* Simple reproduction(s)

```js
function test() {
  for (let i = 0; i < 0; i++) {
    const x = __lookupGetter__; // `__lookupGetter__` and
  }
  const self = this; // `this` should both be present for this to happen
}
```

* Why
  * Crankshaft sees a hole (marker for Temporary Dead Zone of `let`/`const`) and aborts compilation.

* Advices

* External examples


### Yield

* Simple reproduction(s)

```js
// strict & sloppy modes
function* test() {
  yield 0;
}
```

* Why

* Advices

* External examples

---

[1]: https://chromium.googlesource.com/v8/v8/+/d3f074b23195a2426d14298dca30c4cf9183f203%5E%21/src/bailout-reason.h
[2]: https://codereview.chromium.org/1272673003
[3]: https://groups.google.com/forum/#!msg/google-chrome-developer-tools/Y0J2XQ9iiqU/H60qqZNlQa8J
[4]: https://github.com/GoogleChrome/devtools-docs/issues/53#issuecomment-37269998
[5]: https://github.com/GoogleChrome/devtools-docs/issues/53#issuecomment-140030617
[6]: https://github.com/GoogleChrome/devtools-docs/issues/53#issuecomment-145192013
[7]: https://github.com/GoogleChrome/devtools-docs/issues/53#issuecomment-147569505
[8]: https://developers.google.com/v8/design


## References
### Resources

* [All bailout reasons in Chromium codebase](https://code.google.com/p/chromium/codesearch#chromium/src/v8/src/bailout-reason.h)
* [Bad value context for arguments value](https://gist.github.com/Hypercubed/89808f3051101a1a97f3)
* [I-want-to-optimize-my-JS-application-on-V8 checklist](http://mrale.ph/blog/2011/12/18/v8-optimization-checklist.html)
* [JavaScript: Performance loss on incorrect arguments using](http://techblog.dorogin.com/2015/05/performance-loss-on-incorrect-arguments-using.html)
* [Optimization killers](https://github.com/petkaantonov/bluebird/wiki/Optimization-killers)
* [OptimizationKillers](https://github.com/zhangchiqing/OptimizationKillers)
* [Performance Tips for JavaScript in V8](http://www.html5rocks.com/en/tutorials/speed/v8/)
* [thlorenz/v8-perf](https://github.com/thlorenz/v8-perf/blob/master/compiler.md)
* [A high-level tutorial about tracing deopts points](https://www.netguru.co/blog/tracing-patterns-hinder-performance)

### All bailout reasons

* 32 bit value in register is not zero-extended
* Alignment marker expected
* Allocation is not double aligned
* API call returned invalid object
* Arguments object value in a test context
* Array boilerplate creation failed
* Array index constant value too big
* Assignment to arguments
* Assignment to let variable before initialization
* Assignment to LOOKUP variable
* ~~Assignment to parameter in arguments object~~
* Assignment to parameter, function uses arguments object
* Bad value context for arguments object value
* ~~Bad value context for arguments value~~
* Bailed out due to dependency change
* Bailout was not prepared
* Both registers were smis in SelectNonSmi
* Call to a JavaScript runtime function
* Class literal
* Code generation failed
* Code object not properly patched
* Compound assignment to lookup slot
* Computed property name
* Context-allocated arguments
* Copy buffers overlap
* Could not generate +0.0
* Could not generate -0.0
* DebuggerStatement
* Declaration in catch context
* Declaration in with context
* Default NaN mode not set
* Delete with global variable
* Delete with non-global variable
* Destination of copy not aligned
* Do expression encountered
* DontDelete cells can't contain the hole
* DoPushArgument not implemented for double type
* Eliminated bounds check failed
* EmitLoadRegister: Unsupported double immediate
* eval
* Expected +0.0
* Expected alignment marker
* Expected allocation site
* Expected function object in register
* Expected HeapNumber
* Expected native context
* Expected new space object
* Expected non-identical objects
* Expected non-null context
* Expected undefined or cell in register
* Expecting alignment for CopyBytes
* Export declaration
* External string expected, but not found
* ForInStatement optimization is disabled
* ~~ForInStatement with non-local each variable~~
* ForOfStatement
* Frame is expected to be aligned
* Function calls eval
* Function is being debugged
* Function with illegal redeclaration
* Generated code is too large
* Generator
* Generator failed to resume
* Global functions must have initial map
* HeapNumberMap register clobbered
* Import declaration
* Index is negative
* Index is too large
* Inlined runtime function: FastOneByteArrayJoin
* ~~Inlining bailed out~~
* Input GPR is expected to have upper32 cleared
* Input string too long
* Integer32ToSmiField writing to non-smi location
* Invalid capture referenced
* Invalid ElementsKind for InternalArray or InternalPackedArray
* invalid full-codegen state
* Invalid HandleScope level
* Invalid left-hand side in assignment
* Invalid lhs in compound assignment
* Invalid lhs in count operation
* Invalid min_length
* JSGlobalObject::native_context should be a native context
* JSGlobalProxy::context() should not be null
* JSObject with fast elements map has slow elements
* Let binding re-initialization
* Live Bytes Count overflow chunk size
* LiveEdit
* Lookup variable in count operation
* Map became deprecated
* Map became unstable
* Native function literal
* Need a Smi literal here
* No cases left
* No empty arrays here in EmitFastOneByteArrayJoin
* Non-initializer assignment to const
* Non-object value
* Non-smi index
* Non-smi key in array literal
* Non-smi value
* Not enough spill slots for OSR
* Not enough virtual registers (regalloc)
* Not enough virtual registers for values
* Object found in smi-only array
* ~~Object literal with complex property~~
* Offset out of range
* Operand is a smi
* Operand is a smi and not a bound function
* Operand is a smi and not a function
* Operand is a smi and not a name
* Operand is a smi and not a string
* Operand is not a bound function
* Operand is not a date
* Operand is not a function
* Operand is not a name
* Operand is not a number
* Operand is not a smi
* Operand is not a string
* Operand is not smi
* Operand not a number
* Optimization disabled by filter
* Optimization is disabled
* ~~Optimized too many times~~
* Out of virtual registers while trying to allocate temp register
* Parse/scope error
* Possible direct call to eval
* Received invalid return address
* ~~Reference to a variable which requires dynamic lookup~~
* Reference to global lexical variable
* Reference to uninitialized variable
* Register did not match expected root
* Register was clobbered
* Remembered set pointer is in new space
* ~~Rest parameters~~
* Return address not found in frame
* Should not directly enter OSR-compiled function
* Sloppy function expects JSReceiver as receiver.
* ~~Smi addition overflow~~
* ~~Smi subtraction overflow~~
* Spread in array literal
* Stack access below stack pointer
* Stack frame types must match
* Super reference
* The current stack pointer is below csp
* The function_data field should be a BytecodeArray on interpreter entry
* The object is not tagged
* The object is tagged
* The source and destination are the same
* The stack pointer is not the expected value
* The stack was corrupted by MacroAssembler::Call()
* ~~Too many parameters~~
* Too many parameters/locals
* Too many spill slots needed for OSR
* ToOperand IsDoubleRegister unimplemented
* ToOperand Unsupported double immediate
* ToOperand32 unsupported immediate.
* ~~TryCatchStatement~~
* ~~TryFinallyStatement~~
* Unaligned allocation in new space
* Unaligned cell in write barrier
* Unexpected allocation top
* Unexpected color bit pattern found
* Unexpected ElementsKind in array constructor
* Unexpected fall-through from string comparison
* Unexpected fallthrough from CharCodeAt slow case
* Unexpected fallthrough from CharFromCode slow case
* Unexpected fallthrough to CharCodeAt slow case
* Unexpected fallthrough to CharFromCode slow case
* Unexpected FPCR mode.
* Unexpected FPU stack depth after instruction
* Unexpected initial map for Array function
* Unexpected initial map for Array function (1)
* Unexpected initial map for Array function (2)
* Unexpected initial map for InternalArray function
* Unexpected level after return from api call
* Unexpected negative value
* Unexpected number of pre-allocated property fields
* Unexpected smi value
* Unexpected string type
* Unexpected type for RegExp data, FixedArray expected
* Unexpected value
* Unexpectedly returned from a throw
* ~~Unsupported const compound assignment~~
* Unsupported count operation with const
* Unsupported double immediate
* Unsupported let compound assignment
* Unsupported lookup slot in declaration
* Unsupported non-primitive compare
* ~~Unsupported phi use of arguments~~
* ~~Unsupported phi use of const or let variable~~
* Unsupported switch statement
* Unsupported tagged immediate
* Variable resolved to with context
* We should not have an empty lexical context
* WithStatement
* Wrong address or value passed to RecordWrite
* Wrong context passed to function
* ~~Yield~~



### All deoptimize reasons
*NOTE*: For the latest deopts list, look at `src/deoptimize-reason.h` in the V8 src.

* V(AccessCheck, "Access check needed")
* V(NoReason, "no reason")
* V(ConstantGlobalVariableAssignment, "Constant global variable assignment")
* V(ConversionOverflow, "conversion overflow")
* V(DivisionByZero, "division by zero")
* V(ExpectedHeapNumber, "Expected heap number")
* V(ExpectedSmi, "Expected smi")
* V(ForcedDeoptToRuntime, "Forced deopt to runtime")
* V(Hole, "hole")
* V(InstanceMigrationFailed, "instance migration failed")
* V(InsufficientTypeFeedbackForCall, "Insufficient type feedback for call")
* V(InsufficientTypeFeedbackForCallWithArguments, "Insufficient type feedback for call with arguments")
* V(InsufficientTypeFeedbackForConstruct, "Insufficient type feedback for construct")
* V(FastPathFailed, "Falling off the fast path")
* V(InsufficientTypeFeedbackForCombinedTypeOfBinaryOperation, "Insufficient type feedback for combined type of binary operation")
* V(InsufficientTypeFeedbackForGenericNamedAccess, "Insufficient type feedback for generic named access")
* V(InsufficientTypeFeedbackForGenericKeyedAccess, "Insufficient type feedback for generic keyed access")
* V(InsufficientTypeFeedbackForLHSOfBinaryOperation, "Insufficient type feedback for LHS of binary operation")
* V(InsufficientTypeFeedbackForRHSOfBinaryOperation, "Insufficient type feedback for RHS of binary operation")
* V(KeyIsNegative, "key is negative")
* V(LostPrecision, "lost precision")
* V(LostPrecisionOrNaN, "lost precision or NaN")
* V(MementoFound, "memento found")
* V(MinusZero, "minus zero")
* V(NaN, "NaN")
* V(NegativeKeyEncountered, "Negative key encountered")
* V(NegativeValue, "negative value")
* V(NoCache, "no cache")
* V(NotAHeapNumber, "not a heap number")
* V(NotAHeapNumberUndefined, "not a heap number/undefined")
* V(NotAJavaScriptObject, "not a JavaScript object")
* V(NotANumberOrOddball, "not a Number or Oddball")
* V(NotASmi, "not a Smi")
* V(NotASymbol, "not a Symbol")
* V(OutOfBounds, "out of bounds")
* V(OutsideOfRange, "Outside of range")
* V(Overflow, "overflow")
* V(Proxy, "proxy")
* V(ReceiverWasAGlobalObject, "receiver was a global object")
* V(Smi, "Smi")
* V(TooManyArguments, "too many arguments")
* V(TracingElementsTransitions, "Tracing elements transitions")
* V(TypeMismatchBetweenFeedbackAndConstant, "Type mismatch between feedback and constant")
* V(UnexpectedCellContentsInConstantGlobalStore, "Unexpected cell contents in constant global store")
* V(UnexpectedCellContentsInGlobalStore, "Unexpected cell contents in global store")
* V(UnexpectedObject, "unexpected object")
* V(UnexpectedRHSOfBinaryOperation, "Unexpected RHS of binary operation")
* V(UnknownMapInPolymorphicAccess, "Unknown map in polymorphic access")
* V(UnknownMapInPolymorphicCall, "Unknown map in polymorphic call")
* V(UnknownMapInPolymorphicElementAccess, "Unknown map in polymorphic element access")
* V(UnknownMap, "Unknown map")
* V(ValueMismatch, "value mismatch")
* V(WrongInstanceType, "wrong instance type")
* V(WrongMap, "wrong map")
* V(UndefinedOrNullInForIn, "null or undefined in for-in")
* V(UndefinedOrNullInToObject, "null or undefined in ToObject")

