# Coercion and Type Conversion

## The Idea

**In plain English:** Coercion and type conversion is when JavaScript automatically (or manually) changes a value from one kind of data into another kind — for example, turning the text "5" into the number 5, or turning the number 0 into a "false" answer. "Type" just means what category a value belongs to, like a number, a piece of text (called a string), or a true/false value (called a boolean).

**Real-world analogy:** Imagine you're at a currency exchange booth at an airport. You hand the cashier 100 US dollars, and they automatically convert it into 90 euros so you can use it in Europe — you didn't ask for the math, it just happened based on where you are.

- The currency exchange booth = JavaScript's engine doing automatic coercion
- The US dollars = the original value and type (e.g., the string "100")
- The euros = the converted value in the new type (e.g., the number 100)

---

## Table of Contents
- [Overview](#overview)
- [Abstract Operations](#abstract-operations)
- [ToNumber](#tonumber)
- [ToString](#tostring)
- [ToBoolean](#toboolean)
- [ToPrimitive](#toprimitive)
- [Symbol.toPrimitive](#symboltoprimitive)
- [Equality Coercion](#equality-coercion)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

Type coercion is the automatic conversion of values from one type to another in JavaScript. Unlike explicit conversion (casting), coercion happens implicitly during operations. Understanding coercion rules is essential for avoiding bugs and writing predictable JavaScript code.

JavaScript has two types of conversion:
- **Implicit coercion**: Automatic conversion by JavaScript engine
- **Explicit conversion**: Manual conversion using functions like `Number()`, `String()`, etc.

## Abstract Operations

Abstract operations are internal algorithms defined in the ECMAScript specification that perform type conversions.

### Overview of Abstract Operations

```javascript
// Abstract operations are internal, not directly callable
// They define how JavaScript converts types

// Key abstract operations:
// - ToPrimitive: Convert object to primitive
// - ToNumber: Convert value to number
// - ToString: Convert value to string
// - ToBoolean: Convert value to boolean

// These underlie all coercion in JavaScript
```

### Type Conversion Functions

```javascript
// Explicit conversion using wrapper functions
Number('123');    // 123 (string → number)
String(123);      // '123' (number → string)
Boolean(0);       // false (number → boolean)

// These functions call abstract operations internally:
// Number() → ToNumber
// String() → ToString  
// Boolean() → ToBoolean
```

## ToNumber

The ToNumber abstract operation converts values to numbers.

### ToNumber Conversion Rules

```javascript
// undefined → NaN
Number(undefined);  // NaN

// null → 0
Number(null);       // 0

// boolean → 0 or 1
Number(false);      // 0
Number(true);       // 1

// string → parsed number
Number('123');      // 123
Number('12.5');     // 12.5
Number('');         // 0 (empty string)
Number('  42  ');   // 42 (trimmed)
Number('12px');     // NaN (invalid)
Number('0x10');     // 16 (hexadecimal)
Number('0b1010');   // 10 (binary)
Number('0o12');     // 10 (octal)

// Special string values
Number('Infinity'); // Infinity
Number('-Infinity');// -Infinity
Number('NaN');      // NaN (string 'NaN', not the value)
```

### ToNumber with Objects

```javascript
// Objects → ToPrimitive(hint: 'number') → ToNumber

// Plain object
Number({});         // NaN
// Why? {} → '[object Object]' (ToString) → NaN (ToNumber)

// Array
Number([]);         // 0
Number([5]);        // 5
Number([1, 2]);     // NaN
// Why? [] → '' → 0, [5] → '5' → 5, [1,2] → '1,2' → NaN

// Date
Number(new Date()); // Timestamp (milliseconds since epoch)
// Date objects have special ToPrimitive behavior
```

### Implicit ToNumber

```javascript
// Unary plus operator
+'42';              // 42
+'3.14';            // 3.14
+[];                // 0
+{};                // NaN

// Arithmetic operators (except +)
'5' - 2;            // 3
'5' * '2';          // 10
'10' / '2';         // 5
'5' % 2;            // 1

// Comparison operators
'42' > 10;          // true (42 > 10)
'5' < 10;           // true (5 < 10)

// Bitwise operators
'5' | 0;            // 5 (common trick for ToNumber)
'3.7' | 0;          // 3 (also truncates)
```

### Edge Cases

```javascript
// Special values
Number(NaN);        // NaN
Number(Infinity);   // Infinity

// Weird coercions
Number(' ');        // 0 (whitespace)
Number('\n\t');     // 0 (whitespace)
Number('0');        // 0
Number('000');      // 0

// Objects with valueOf
const obj = {
  valueOf() {
    return 42;
  }
};
Number(obj);        // 42 (calls valueOf)

// Objects with toString (no valueOf)
const obj2 = {
  toString() {
    return '99';
  }
};
Number(obj2);       // 99 (calls toString, then ToNumber)
```

## ToString

The ToString abstract operation converts values to strings.

### ToString Conversion Rules

```javascript
// undefined → 'undefined'
String(undefined);  // 'undefined'

// null → 'null'
String(null);       // 'null'

// boolean → 'true' or 'false'
String(true);       // 'true'
String(false);      // 'false'

// number → string representation
String(123);        // '123'
String(3.14);       // '3.14'
String(NaN);        // 'NaN'
String(Infinity);   // 'Infinity'
String(0);          // '0'
String(-0);         // '0' (loses sign!)

// Scientific notation for very large/small numbers
String(1e21);       // '1e+21'
String(1e-7);       // '1e-7'
```

### ToString with Objects

```javascript
// Objects → ToPrimitive(hint: 'string') → ToString

// Plain object
String({});         // '[object Object]'
String({a: 1});     // '[object Object]'

// Array → joins elements with comma
String([]);         // ''
String([1, 2, 3]);  // '1,2,3'
String([[1, 2]]);   // '1,2'

// Function
String(function() {}); // 'function() {}'

// Date
String(new Date(0)); // 'Thu Jan 01 1970...' (locale-dependent)
```

### Implicit ToString

```javascript
// String concatenation with +
'Hello ' + 'World'; // 'Hello World'
'Value: ' + 42;     // 'Value: 42'
'Sum: ' + (1 + 2);  // 'Sum: 3'

// Template literals
`Value: ${42}`;     // 'Value: 42'
`Array: ${[1,2,3]}`; // 'Array: 1,2,3'

// alert, console.log (implicitly call ToString)
console.log({a: 1}); // Logs: { a: 1 } (console.log special handling)
alert({a: 1});       // Shows: [object Object]
```

### Custom toString

```javascript
// Object with custom toString
const obj = {
  toString() {
    return 'Custom String';
  }
};

String(obj);        // 'Custom String'
'' + obj;           // 'Custom String'
`Value: ${obj}`;    // 'Value: Custom String'

// Array with custom toString
Array.prototype.toString = function() {
  return `[${this.join(', ')}]`;
};
// DON'T actually do this! Just for illustration
```

### JSON.stringify vs ToString

```javascript
// JSON.stringify is NOT ToString
const obj = { a: 1, b: undefined, c: () => {} };

String(obj);        // '[object Object]'
JSON.stringify(obj); // '{"a":1}' (omits undefined, functions)

// JSON.stringify handles special values
JSON.stringify(undefined);  // undefined (not 'undefined')
JSON.stringify(NaN);        // 'null'
JSON.stringify(Infinity);   // 'null'
JSON.stringify(null);       // 'null'
```

## ToBoolean

The ToBoolean abstract operation converts values to booleans. This is the simplest conversion.

### Falsy Values

```javascript
// Only 7 falsy values in JavaScript:
Boolean(false);     // false
Boolean(0);         // false
Boolean(-0);        // false
Boolean(0n);        // false (BigInt zero)
Boolean('');        // false (empty string)
Boolean(null);      // false
Boolean(undefined); // false
Boolean(NaN);       // false

// Everything else is truthy!
```

### Truthy Values

```javascript
// All other values are truthy
Boolean(true);      // true
Boolean(1);         // true
Boolean(-1);        // true
Boolean('0');       // true (non-empty string)
Boolean('false');   // true (non-empty string)
Boolean([]);        // true (empty array)
Boolean({});        // true (empty object)
Boolean(() => {});  // true (function)

// Common surprises
Boolean(new Boolean(false)); // true (object wrapper)
Boolean(new Number(0));      // true (object wrapper)
Boolean(new String(''));     // true (object wrapper)
```

### Implicit ToBoolean

```javascript
// Conditional statements
if ('hello') {      // Truthy
  console.log('Executed');
}

// Logical operators
const result = value || 'default'; // Falsy check
const isValid = !!value;           // Double negation

// Ternary operator
const output = value ? 'yes' : 'no';

// while loops
while (condition) { // ToBoolean(condition)
  // ...
}
```

### Logical Operators

```javascript
// && returns first falsy value or last value
true && false;      // false
5 && 0;             // 0 (first falsy)
1 && 2 && 3;        // 3 (all truthy, return last)

// || returns first truthy value or last value
false || true;      // true
0 || 5;             // 5 (first truthy)
'' || 0 || null;    // null (all falsy, return last)

// ?? (nullish coalescing) only checks null/undefined
0 ?? 5;             // 0 (0 is not null/undefined)
null ?? 5;          // 5
undefined ?? 5;     // 5
```

## ToPrimitive

The ToPrimitive abstract operation converts objects to primitive values.

### ToPrimitive Algorithm

```javascript
// ToPrimitive(input, hint)
// hint: 'number', 'string', or 'default'

// Algorithm:
// 1. If input is primitive, return it
// 2. If hint is 'string':
//    Try toString(), then valueOf()
// 3. If hint is 'number' or 'default':
//    Try valueOf(), then toString()
// 4. If no primitive returned, throw TypeError
```

### ToPrimitive with Hint 'number'

```javascript
const obj = {
  valueOf() {
    console.log('valueOf called');
    return 42;
  },
  toString() {
    console.log('toString called');
    return 'Object';
  }
};

// Hint 'number': valueOf first
Number(obj);        // valueOf called → 42
+obj;               // valueOf called → 42
obj - 0;            // valueOf called → 42

// If valueOf returns non-primitive, try toString
const obj2 = {
  valueOf() {
    return {}; // Non-primitive
  },
  toString() {
    return '99';
  }
};

Number(obj2);       // 99 (valueOf returned object, toString used)
```

### ToPrimitive with Hint 'string'

```javascript
const obj = {
  valueOf() {
    console.log('valueOf called');
    return 42;
  },
  toString() {
    console.log('toString called');
    return 'Object';
  }
};

// Hint 'string': toString first
String(obj);        // toString called → 'Object'
`${obj}`;           // toString called → 'Object'
alert(obj);         // toString called → shows 'Object'

// If toString returns non-primitive, try valueOf
const obj2 = {
  toString() {
    return {}; // Non-primitive
  },
  valueOf() {
    return 42;
  }
};

String(obj2);       // '42' (toString returned object, valueOf used)
```

### ToPrimitive with Hint 'default'

```javascript
// Hint 'default': behaves like 'number' for most objects
// Exception: Date objects treat 'default' as 'string'

const obj = {
  valueOf() {
    return 42;
  },
  toString() {
    return 'Object';
  }
};

// + operator uses hint 'default'
obj + 1;            // 43 (valueOf called)
obj + '!';          // '42!' (valueOf called, then concatenation)

// Date special case
const date = new Date(0);
date + 1;           // 'Thu Jan 01 1970...1' (toString, then concat)
date - 0;           // 0 (valueOf, returns timestamp)
```

## Symbol.toPrimitive

`Symbol.toPrimitive` is a well-known symbol that lets objects customize their conversion to primitives.

### Basic Usage

```javascript
const obj = {
  [Symbol.toPrimitive](hint) {
    console.log(`Hint: ${hint}`);
    
    if (hint === 'number') {
      return 42;
    }
    if (hint === 'string') {
      return 'Hello';
    }
    return 'default';
  }
};

Number(obj);        // Hint: number → 42
String(obj);        // Hint: string → 'Hello'
obj + 1;            // Hint: default → 'default1'
```

### Symbol.toPrimitive Overrides valueOf/toString

```javascript
const obj = {
  valueOf() {
    return 99; // Ignored if [Symbol.toPrimitive] exists
  },
  toString() {
    return 'Ignored'; // Also ignored
  },
  [Symbol.toPrimitive](hint) {
    return hint === 'number' ? 42 : 'Custom';
  }
};

Number(obj);        // 42 (toPrimitive, not valueOf)
String(obj);        // 'Custom' (toPrimitive, not toString)
```

### Practical Example

```javascript
class Money {
  constructor(amount, currency) {
    this.amount = amount;
    this.currency = currency;
  }
  
  [Symbol.toPrimitive](hint) {
    if (hint === 'number') {
      return this.amount;
    }
    if (hint === 'string') {
      return `${this.currency} ${this.amount}`;
    }
    return this.amount; // default
  }
}

const price = new Money(100, 'USD');

console.log(+price);           // 100 (number hint)
console.log(`Price: ${price}`); // 'Price: USD 100' (string hint)
console.log(price + 50);       // 150 (default hint → number)
```

### Error Handling

```javascript
const obj = {
  [Symbol.toPrimitive](hint) {
    return {}; // Must return primitive!
  }
};

// Number(obj); // TypeError: Cannot convert object to primitive value

// Must return primitive value
const validObj = {
  [Symbol.toPrimitive](hint) {
    return null; // Primitives: string, number, bigint, boolean, undefined, null
  }
};

Number(validObj); // 0 (null → 0)
```

## Equality Coercion

The `==` operator performs type coercion, while `===` doesn't.

### Abstract Equality (==)

```javascript
// Same type: no coercion
5 == 5;             // true
'hello' == 'hello'; // true

// null and undefined are equal to each other
null == undefined;  // true
null == null;       // true
undefined == undefined; // true

// Number comparison: coerce to number
5 == '5';           // true (ToNumber('5') = 5)
0 == false;         // true (ToNumber(false) = 0)
1 == true;          // true (ToNumber(true) = 1)
'' == 0;            // true (ToNumber('') = 0)

// Object comparison: ToPrimitive, then compare
[1] == 1;           // true ([1] → '1' → 1)
[1,2] == '1,2';     // true ([1,2] → '1,2')
```

### Equality Algorithm

```javascript
// Simplified == algorithm:
// 1. Same type? Use === (strict equality)
// 2. null == undefined? true
// 3. Number vs String? ToNumber(string)
// 4. Boolean? ToNumber(boolean)
// 5. Object vs Primitive? ToPrimitive(object)
// 6. Otherwise false

// Examples
'0' == false;       // true
// false → 0 (ToNumber)
// '0' → 0 (ToNumber)
// 0 === 0 → true

[1] == true;        // true
// [1] → '1' (ToPrimitive)
// true → 1 (ToNumber)
// '1' → 1 (ToNumber)
// 1 === 1 → true
```

### Strict Equality (===)

```javascript
// No coercion, must be same type and value
5 === 5;            // true
5 === '5';          // false (different types)
null === undefined; // false (different types)
0 === false;        // false (different types)
[] === [];          // false (different objects)

// Special cases
NaN === NaN;        // false (NaN is never equal to itself)
+0 === -0;          // true (signed zeros are equal)

// Use Object.is() for true equality
Object.is(NaN, NaN); // true
Object.is(+0, -0);   // false
```

### Common Coercion Gotchas

```javascript
// Tricky coercions
[] == ![];          // true
// ![] → false (ToBoolean([]) = true, negated = false)
// [] == false → '' == false → 0 == 0 → true

'' == 0;            // true
'  ' == 0;          // true (whitespace trimmed)
'0' == 0;           // true

// Watch out for these
if (obj == true) {} // Probably not what you want
if (obj) {}         // Better: ToBoolean check

// Avoid == with these values
// 0, '', '   ', false, null, undefined, []
// Use === instead
```

## Common Misconceptions

### Misconception 1: "Coercion Is Always Bad"

**Reality**: Coercion is a language feature, not a bug. Understanding it enables writing concise code.

```javascript
// VERBOSE: Explicit checks
if (value !== null && value !== undefined && value !== '') {
  // ...
}

// CONCISE: Leveraging truthiness
if (value) {
  // ...
}

// GOOD use of coercion
const count = items.length || 0; // Default to 0 if undefined
const name = user.name ?? 'Anonymous'; // Nullish coalescing
```

### Misconception 2: "== Always Does ToNumber"

**Reality**: `==` has multiple coercion rules, not just ToNumber.

```javascript
// Different coercion paths
'5' == 5;           // String → Number
false == 0;         // Boolean → Number
[1] == '1';         // Object → Primitive
null == undefined;  // Special case (no coercion)
```

### Misconception 3: "+ Always Concatenates with Strings"

**Reality**: `+` is overloaded. If one operand is string, concatenates; otherwise, adds numbers.

```javascript
1 + 2;              // 3 (addition)
'1' + 2;            // '12' (concatenation)
1 + '2';            // '12' (concatenation)
1 + 2 + '3';        // '33' (left-to-right: 3 + '3')
'1' + 2 + 3;        // '123' ('1' + 2 → '12', then + 3 → '123')
```

### Misconception 4: "Objects Are Always Truthy"

**Reality**: All objects are truthy, but their coerced values might not be.

```javascript
if (new Boolean(false)) {
  console.log('Runs!'); // Object is truthy
}

if (new Boolean(false) == true) {
  console.log('Doesn\'t run'); // Coerced value is false
}

// Object wrapper is truthy, coerced value follows object's value
```

## Performance Implications

### Explicit vs Implicit Conversion

```javascript
// Explicit conversions (clearer, similar performance)
const num1 = Number(str);
const str1 = String(num);
const bool1 = Boolean(val);

// Implicit conversions (can be optimized by engine)
const num2 = +str;
const str2 = num + '';
const bool2 = !!val;

// Modern engines optimize both similarly
// Choose based on code clarity, not performance
```

### Type Stability

```javascript
// BAD: Polymorphic (different types)
function add(a, b) {
  return a + b; // Sometimes number, sometimes string
}

add(1, 2);      // 3
add('1', '2');  // '12'

// GOOD: Monomorphic (consistent types)
function addNumbers(a, b) {
  return Number(a) + Number(b); // Always number
}

function concatStrings(a, b) {
  return String(a) + String(b); // Always string
}

// Monomorphic functions optimize better
```

### Avoiding Coercion Overhead

```javascript
// BAD: Repeated coercion in loop
for (let i = 0; i < arr.length; i++) {
  if (arr[i] == targetString) { // Coercion each iteration
    // ...
  }
}

// GOOD: Coerce once
const target = Number(targetString);
for (let i = 0; i < arr.length; i++) {
  if (arr[i] === target) { // No coercion
    // ...
  }
}
```

## Interview Questions

### Question 1: Explain the difference between == and ===

**Answer**: `==` (abstract equality) performs type coercion if operands are different types, then compares. `===` (strict equality) checks both type and value without coercion. `==` has complex coercion rules (null == undefined, number vs string, etc.), while `===` is straightforward. Generally prefer `===` except when specifically wanting null/undefined equivalence.

### Question 2: What are the falsy values in JavaScript?

**Answer**: There are exactly 7 falsy values: `false`, `0`, `-0`, `0n` (BigInt zero), `''` (empty string), `null`, `undefined`, and `NaN`. Everything else is truthy, including empty arrays `[]`, empty objects `{}`, and wrapper objects like `new Boolean(false)`.

### Question 3: How does the ToPrimitive abstract operation work?

**Answer**: ToPrimitive converts objects to primitives based on a hint ('number', 'string', or 'default'). With hint 'number', it tries `valueOf()` then `toString()`. With hint 'string', it tries `toString()` then `valueOf()`. Objects can customize this with `Symbol.toPrimitive`. If no primitive is returned, it throws a TypeError.

### Question 4: Why does [] == ![] evaluate to true?

**Answer**: 
1. `![]` → `false` (empty array is truthy, negated becomes false)
2. `[] == false` → ToPrimitive([]) and ToNumber(false)
3. `[] → ''` (array to string is empty)
4. `'' == false` → ToNumber('') and ToNumber(false)
5. `0 == 0` → `true`

This demonstrates why understanding coercion rules is important and why `===` is often safer.

### Question 5: What's the difference between + operator with numbers and strings?

**Answer**: The `+` operator is overloaded. If either operand is a string, it performs string concatenation (both operands converted to strings). If both are numbers, it performs addition. This is why `1 + 2` is `3` but `1 + '2'` is `'12'`. The evaluation is left-to-right, so `1 + 2 + '3'` becomes `'33'` (3 + '3'), while `'1' + 2 + 3` becomes `'123'` ('1' + 2 → '12', then + 3 → '123').

## Key Takeaways

1. **Coercion is built-in**: JavaScript automatically converts types in many operations

2. **Abstract operations define coercion**: ToNumber, ToString, ToBoolean, ToPrimitive

3. **Falsy values**: Only 7 falsy values; everything else is truthy

4. **ToPrimitive uses valueOf/toString**: Objects converted to primitives via these methods

5. **Symbol.toPrimitive**: Custom primitive conversion, overrides valueOf/toString

6. **== performs coercion**: Complex rules, type-dependent behavior

7. **=== is stricter**: No coercion, checks type and value

8. **+ operator is special**: Concatenates with strings, adds with numbers

9. **Understanding enables concise code**: Leveraging truthiness, default values

10. **Type stability aids performance**: Consistent types optimize better than polymorphic code

## Resources

### Official Documentation
- [MDN: Type coercion](https://developer.mozilla.org/en-US/docs/Glossary/Type_coercion)
- [MDN: Equality comparisons](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Equality_comparisons_and_sameness)
- [ECMAScript Specification: Abstract Operations](https://tc39.es/ecma262/#sec-abstract-operations)

### Articles
- "JavaScript type coercion explained" by Alexey Samoshkin
- "Understanding JS: Coercion" by Preethi Kasireddy
- "Coercion in JavaScript" by Axel Rauschmayer

### Books
- "You Don't Know JS: Types & Grammar" by Kyle Simpson
- "JavaScript: The Definitive Guide" by David Flanagan
- "Eloquent JavaScript" by Marijn Haverbeke

### Tools
- [JavaScript Equality Table](https://dorey.github.io/JavaScript-Equality-Table/)
- [Type coercion visualizer](https://bit.ly/2Gnqlr8)
