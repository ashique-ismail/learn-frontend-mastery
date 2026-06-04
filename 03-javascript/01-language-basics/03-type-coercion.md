# Type Coercion: Understanding JavaScript's Type Conversions

## Overview

Type coercion is the automatic or implicit conversion of values from one data type to another. JavaScript is a dynamically typed language with weak typing, meaning it performs type coercion automatically in many situations. While this flexibility can be convenient, it's also a major source of bugs and confusion. Understanding how and when coercion happens is essential for writing predictable JavaScript code.

## Explicit vs Implicit Coercion

### Explicit Coercion (Type Casting)

Explicit coercion is intentional and obvious in the code:

```javascript
// String conversion
const num = 42;
const str = String(num);        // "42"
const str2 = num.toString();    // "42"

// Number conversion
const str = "42";
const num = Number(str);        // 42
const num2 = parseInt(str, 10); // 42
const num3 = parseFloat(str);   // 42
const num4 = +str;              // 42 (unary plus)

// Boolean conversion
const value = "hello";
const bool = Boolean(value);    // true
const bool2 = !!value;          // true (double negation)
```

### Implicit Coercion

Implicit coercion happens automatically during operations:

```javascript
// String coercion
const result = "The answer is " + 42;  // "The answer is 42"
const template = `Value: ${true}`;     // "Value: true"

// Number coercion
const sum = "5" - 2;              // 3 (string "5" -> number 5)
const product = "3" * "4";        // 12
const comparison = "10" > 5;      // true

// Boolean coercion
if ("hello") {                    // "hello" -> true
  console.log("Truthy");
}

const value = null || "default";  // "default" (null is falsy)
```

## String Coercion

### ToPrimitive → ToString

Objects are converted to strings via the ToPrimitive algorithm:

```javascript
// Simple values
String(123);        // "123"
String(true);       // "true"
String(false);      // "false"
String(null);       // "null"
String(undefined);  // "undefined"

// Objects use toString() method
String({});                    // "[object Object]"
String([]);                    // ""
String([1, 2, 3]);            // "1,2,3"
String(function() {});        // "function() {}"

// Custom toString
const obj = {
  toString() {
    return "Custom string";
  }
};
String(obj); // "Custom string"

// Symbol throws TypeError
String(Symbol('test')); // TypeError: Cannot convert a Symbol value to a string
```

### The + Operator (String Concatenation)

When + is used with a string, it triggers string coercion:

```javascript
"hello" + "world";     // "helloworld"
"number: " + 42;       // "number: 42"
"5" + 3;               // "53" (NOT 8!)
3 + "5";               // "35"
"" + 123;              // "123"

// Multiple operands evaluated left-to-right
1 + 2 + "3";           // "33" (1+2=3, then "3"+"3")
"1" + 2 + 3;           // "123" (all concatenated)

// Template literals always coerce to string
`The value is ${null}`;        // "The value is null"
`Result: ${[1, 2, 3]}`;        // "Result: 1,2,3"
```

### String() vs toString()

```javascript
// toString() cannot be called on null/undefined
null.toString();      // TypeError
undefined.toString(); // TypeError

// String() handles null/undefined safely
String(null);         // "null"
String(undefined);    // "undefined"

// Custom toString() is respected
const obj = {
  toString() { return "custom"; }
};
obj.toString();  // "custom"
String(obj);     // "custom"
```

## Number Coercion

### ToNumber Algorithm

```javascript
// Primitives
Number("123");        // 123
Number("12.5");       // 12.5
Number("");           // 0 (empty string -> 0)
Number("  ");         // 0 (whitespace only -> 0)
Number("0x10");       // 16 (hex)
Number("123abc");     // NaN (invalid)
Number(true);         // 1
Number(false);        // 0
Number(null);         // 0 (!)
Number(undefined);    // NaN (!)

// Objects -> valueOf() or toString() -> Number()
Number({});              // NaN
Number([]);              // 0 ([] -> "" -> 0)
Number([1]);             // 1 ([1] -> "1" -> 1)
Number([1, 2]);          // NaN ([1,2] -> "1,2" -> NaN)

// Custom valueOf
const obj = {
  valueOf() { return 42; }
};
Number(obj); // 42
```

### Arithmetic Operators (Except +)

All arithmetic operators (except +) trigger number coercion:

```javascript
"5" - 2;       // 3
"10" * "2";    // 20
"20" / "4";    // 5
"7" % 2;       // 1
"10" ** "2";   // 100

// Even with non-numeric strings
"abc" - 1;     // NaN
"5px" * 2;     // NaN

// Unary + is a number coercion operator
+"42";         // 42
+true;         // 1
+false;        // 0
+null;         // 0
+undefined;    // NaN
+"";           // 0
```

### parseInt vs Number

Important differences:

```javascript
Number("123px");       // NaN
parseInt("123px");     // 123 (stops at first non-digit)

Number("");            // 0
parseInt("");          // NaN

Number("0x10");        // 16
parseInt("0x10");      // 16

// parseInt with radix (always specify!)
parseInt("10", 10);    // 10 (decimal)
parseInt("10", 2);     // 2 (binary)
parseInt("10", 16);    // 16 (hex)
parseInt("010");       // 10 (used to be 8 in old browsers!)
```

## Boolean Coercion

### ToBoolean

JavaScript has specific rules for what's truthy and falsy:

```javascript
// FALSY values (only 7):
Boolean(false);        // false
Boolean(0);            // false
Boolean(-0);           // false
Boolean(0n);           // false (BigInt zero)
Boolean("");           // false
Boolean(null);         // false
Boolean(undefined);    // false
Boolean(NaN);          // false

// EVERYTHING ELSE is truthy:
Boolean(true);         // true
Boolean(1);            // true
Boolean(-1);           // true
Boolean("0");          // true (!)
Boolean("false");      // true (!)
Boolean([]);           // true (!)
Boolean({});           // true (!)
Boolean(function(){}); // true
Boolean(Infinity);     // true
Boolean(new Date());   // true
```

### Logical Operators

Logical operators trigger boolean coercion but return original values:

```javascript
// && returns first falsy value or last value
0 && "hello";           // 0 (first falsy)
1 && 2 && 3;            // 3 (all truthy, return last)
"a" && 0 && "b";        // 0 (first falsy)

// || returns first truthy value or last value
0 || null || "hello";   // "hello" (first truthy)
"" || 0 || false;       // false (all falsy, return last)

// ?? (nullish coalescing) - only null/undefined are nullish
null ?? "default";      // "default"
undefined ?? "default"; // "default"
0 ?? "default";         // 0 (0 is not nullish!)
"" ?? "default";        // "" (empty string is not nullish!)

// Common patterns
const value = userInput || "default";  // Falls back on any falsy
const value2 = userInput ?? "default"; // Falls back only on null/undefined
```

### If Statements and Loops

Conditions are coerced to boolean:

```javascript
if ("hello") { /* executes - truthy */ }
if (0) { /* doesn't execute - falsy */ }
if ([]) { /* executes - arrays are truthy! */ }

while ([].length) { /* doesn't execute - 0 is falsy */ }

// Ternary operator
const result = "value" ? "truthy" : "falsy"; // "truthy"
const result2 = 0 ? "truthy" : "falsy";      // "falsy"
```

## Equality Operators

### Loose Equality (==)

Performs type coercion before comparison:

```javascript
// Same type - no coercion
5 == 5;              // true
"hello" == "hello";  // true

// Different types - coercion occurs
5 == "5";            // true ("5" -> 5)
1 == true;           // true (true -> 1)
0 == false;          // true (false -> 0)
null == undefined;   // true (special case)

// Common gotchas
"" == 0;             // true ("" -> 0)
"" == false;         // true ("" -> 0, false -> 0)
" " == 0;            // true (" " -> 0)
"\n" == 0;           // true ("\n" -> 0)

// Arrays and objects
[1] == 1;            // true ([1] -> "1" -> 1)
[1,2] == "1,2";      // true ([1,2] -> "1,2")
```

### Strict Equality (===)

No type coercion - types must match:

```javascript
5 === 5;             // true
"5" === "5";         // true

// Different types always false
5 === "5";           // false
1 === true;          // false
0 === false;         // false
null === undefined;  // false

// Special cases
NaN === NaN;         // false (use Number.isNaN() instead)
0 === -0;            // true (use Object.is() for distinction)

// Objects compared by reference
{} === {};           // false (different objects)
[] === [];           // false (different arrays)
```

### Abstract Equality Algorithm

The == algorithm (simplified):

1. If types are the same, use ===
2. If null == undefined, return true
3. If one is number and other is string, convert string to number
4. If one is boolean, convert it to number
5. If one is object and other is primitive, convert object to primitive

```javascript
// Step-by-step example: [1] == 1
// 1. Types different (object vs number)
// 2. [1] is object, convert to primitive
// 3. [1].toString() -> "1"
// 4. "1" == 1
// 5. Types different (string vs number)
// 6. "1" -> 1
// 7. 1 == 1 -> true
```

### Which to Use?

```javascript
// Best practice: Always use ===
if (value === 5) { /* ... */ }

// Exception: Checking for null OR undefined
if (value == null) {  // true for both null and undefined
  // Equivalent to: if (value === null || value === undefined)
}

// Don't use == in other cases
if (value == 0) { /* Bad - catches false, "", [], etc. */ }
if (value === 0) { /* Good - only catches number 0 */ }
```

## Truthy and Falsy Values

### The 8 Falsy Values

```javascript
if (false) {}       // 1. false
if (0) {}           // 2. 0
if (-0) {}          // 3. -0
if (0n) {}          // 4. 0n (BigInt zero)
if ("") {}          // 5. Empty string
if (null) {}        // 6. null
if (undefined) {}   // 7. undefined
if (NaN) {}         // 8. NaN
```

### Common Truthy Gotchas

```javascript
// These are all TRUTHY:
if ("0") {}         // String "0" is truthy!
if ("false") {}     // String "false" is truthy!
if ([]) {}          // Empty array is truthy!
if ({}) {}          // Empty object is truthy!
if (function(){}) {}// Functions are truthy

// Check for empty array/object
if (arr.length > 0) {}           // Array has items
if (Object.keys(obj).length > 0) {}  // Object has properties
```

### Checking for Actual Boolean

```javascript
const value = "false";

// Wrong - any string is truthy
if (value) {
  console.log("This runs!");
}

// Right - check for boolean type
if (value === true) {
  console.log("This doesn't run");
}

// Or explicit conversion
if (Boolean(value) === true) {
  console.log("This runs - string is truthy");
}
```

## Object to Primitive Conversion

### ToPrimitive Algorithm

Objects are converted to primitives via:
1. Call `[Symbol.toPrimitive]` if exists
2. Otherwise, for "string" hint: try `toString()` then `valueOf()`
3. For "number" or "default" hint: try `valueOf()` then `toString()`

```javascript
const obj = {
  valueOf() {
    console.log("valueOf");
    return 42;
  },
  toString() {
    console.log("toString");
    return "hello";
  }
};

// Number context (hint: "number")
+obj;            // Logs "valueOf", returns 42
Number(obj);     // Logs "valueOf", returns 42

// String context (hint: "string")
String(obj);     // Logs "toString", returns "hello"
`${obj}`;        // Logs "toString", returns "hello"

// Default context (hint: "default") - usually number
obj + 1;         // Logs "valueOf", returns 43
obj == 42;       // Logs "valueOf", returns true
```

### Custom Symbol.toPrimitive

Takes precedence over valueOf/toString:

```javascript
const obj = {
  [Symbol.toPrimitive](hint) {
    console.log(`Hint: ${hint}`);
    
    if (hint === 'number') {
      return 42;
    }
    if (hint === 'string') {
      return 'hello';
    }
    return 'default';
  }
};

+obj;           // "Hint: number", returns 42
String(obj);    // "Hint: string", returns "hello"
obj + '';       // "Hint: default", returns "default"
```

### Date Special Case

Date objects use different defaults:

```javascript
const date = new Date('2024-05-26');

// Default hint is "string" for Date (unusual!)
date + 1;        // "Sun May 26 2024 00:00:00 GMT+0000 (UTC)1"
String(date);    // "Sun May 26 2024 00:00:00 GMT+0000 (UTC)"

// Number hint
+date;           // 1716768000000 (timestamp)
date - 0;        // 1716768000000
Number(date);    // 1716768000000
```

## Common Gotchas and Edge Cases

### Array Coercion

```javascript
// Empty array
[] == false;           // true ([] -> "" -> 0, false -> 0)
[] == 0;               // true
[] == "";              // true
[].toString();         // ""

// Single element
[1] == 1;              // true ([1] -> "1" -> 1)
[1].toString();        // "1"

// Multiple elements
[1, 2] == "1,2";       // true
[1, 2].toString();     // "1,2"
```

### Addition Edge Cases

```javascript
// + with mixed types
[] + [];               // "" (both -> "")
[] + {};               // "[object Object]"
{} + [];               // 0 (!) or "[object Object]" (depends on context)
{} + {};               // "[object Object][object Object]" or NaN

// Why the {} + [] confusion?
{} + [];               // {} treated as empty block, +[] = 0
({}) + [];             // Object + Array, "[object Object]"
```

### Comparison Gotchas

```javascript
// Transitive property doesn't hold
"0" == 0;              // true
0 == [];               // true
"0" == [];             // false (!!)

// null and undefined
null == 0;             // false
null == undefined;     // true
undefined == 0;        // false

// NaN
NaN == NaN;            // false
NaN === NaN;           // false
Object.is(NaN, NaN);   // true
Number.isNaN(NaN);     // true
```

### Boolean Coercion Surprises

```javascript
// All truthy, but not equal to true
Boolean([]);           // true
[] == true;            // false ([] -> "" -> 0, true -> 1)

Boolean({});           // true
{} == true;            // false

// Don't use == with booleans!
if (value == true) {}  // Bad
if (value) {}          // Good
if (Boolean(value) === true) {}  // Explicit
```

## Best Practices

### 1. Always Use Strict Equality

```javascript
// Bad
if (value == "5") {}

// Good
if (value === "5") {}

// Exception: null check
if (value == null) {}  // OK - checks for null OR undefined
```

### 2. Explicit Coercion Over Implicit

```javascript
// Bad - implicit
const total = "5" + 3;     // "53"
const sum = "5" - 3;       // 2

// Good - explicit
const total = String(5) + String(3);  // "53"
const sum = Number("5") - 3;          // 2
```

### 3. Validate Input Types

```javascript
// Bad
function add(a, b) {
  return a + b;  // Could be addition or concatenation!
}

// Good
function add(a, b) {
  if (typeof a !== 'number' || typeof b !== 'number') {
    throw new TypeError('Both arguments must be numbers');
  }
  return a + b;
}
```

### 4. Use Proper Empty Checks

```javascript
// Bad - catches more than you want
if (!value) {}  // false for 0, "", false, null, undefined, NaN

// Good - be specific
if (value === null || value === undefined) {}
if (value === "") {}
if (value === 0) {}

// Or use nullish coalescing
const result = value ?? defaultValue;  // Only for null/undefined
```

### 5. Avoid Truthiness for Boolean Values

```javascript
const isActive = someFunction(); // Returns true or false

// Bad - what if it returns "false" string?
if (isActive) {}

// Good
if (isActive === true) {}

// Or ensure boolean type
if (Boolean(isActive) === true) {}
```

## Interview Questions

### Q1: What's the output?

```javascript
console.log([] + []);
console.log([] + {});
console.log({} + []);
console.log({} + {});
```

**Answer:**
- `[] + []` = `""` (both arrays convert to empty strings)
- `[] + {}` = `"[object Object]"` (array to "", object to "[object Object]")
- `{} + []` = `0` (if {} interpreted as block) or `"[object Object]"` (if object)
- `{} + {}` = `NaN` (if blocks) or `"[object Object][object Object]"` (if objects)

### Q2: Why does this happen?

```javascript
console.log(0.1 + 0.2 === 0.3); // false
```

**Answer:** Floating-point precision errors. `0.1 + 0.2 = 0.30000000000000004`. Use `Math.abs(a - b) < Number.EPSILON` for comparison.

### Q3: What's the difference?

```javascript
const a = value || "default";
const b = value ?? "default";
```

**Answer:** `||` returns "default" for any falsy value (including 0, "", false). `??` only returns "default" for null or undefined.

## Resources

### Documentation
- [MDN: Type Coercion](https://developer.mozilla.org/en-US/docs/Glossary/Type_coercion)
- [MDN: Equality Comparisons](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Equality_comparisons_and_sameness)
- [ECMAScript: Type Conversion](https://tc39.es/ecma262/#sec-type-conversion)

### Articles
- [Understanding JavaScript Coercion](https://javascript.info/type-conversions)
- [Equality Table](https://dorey.github.io/JavaScript-Equality-Table/)
- [WTFJS - JavaScript Gotchas](https://github.com/denysdovhan/wtfjs)

### Books
- "You Don't Know JS: Types & Grammar" by Kyle Simpson
