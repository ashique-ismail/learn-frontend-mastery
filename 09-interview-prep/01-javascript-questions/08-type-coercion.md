# Type Coercion - JavaScript Interview Questions

## The Idea

**In plain English:** Type coercion is when JavaScript automatically converts a value from one kind of data (like a number) into another kind (like text) so it can complete an operation. A "type" is just the category of a value — for example, whether something is a number, a word, or true/false.

**Real-world analogy:** Imagine a vending machine that only accepts dollar bills, but you insert a handful of coins. The machine quietly counts the coins and treats them as if they were dollars so the transaction can go through — you never asked it to do that, it just did it on its own.

- The coins = a value of one type (e.g., a text string like `"5"`)
- The dollar-bill slot = an operation that expects a different type (e.g., a number)
- The machine silently converting coins to dollars = JavaScript automatically converting the string to a number

---

## Table of Contents

- [Core Concepts](#core-concepts)
- [Common Interview Questions](#common-interview-questions)
- [Advanced Questions](#advanced-questions)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Red Flags to Avoid](#red-flags-to-avoid)
- [Key Takeaways](#key-takeaways)

## Core Concepts

### Type Coercion

Automatic or implicit conversion of values from one data type to another. JavaScript is loosely typed, so coercion happens frequently.

**Two Types:**
1. **Implicit Coercion**: Automatic conversion by JavaScript
2. **Explicit Coercion**: Manual conversion using functions (Number(), String(), Boolean())

## Common Interview Questions

### Question 1: Explain == vs ===

**Expected Answer:**

```javascript
// === (strict equality) - checks type AND value
console.log(5 === 5); // true
console.log(5 === '5'); // false (different types)
console.log(null === undefined); // false

// == (loose equality) - performs type coercion
console.log(5 == 5); // true
console.log(5 == '5'); // true (string converted to number)
console.log(null == undefined); // true (special case)
console.log(0 == false); // true (false converted to 0)
console.log('' == false); // true (both convert to 0)
console.log('0' == false); // true (both convert to 0)

// Best practice: always use ===
if (value === undefined || value === null) { }
// Modern: use nullish coalescing
const result = value ?? 'default';
```

### Question 2: Truthy and Falsy Values

**Expected Answer:**

```javascript
// Falsy values (8 total)
false        // boolean false
0            // number zero
-0           // negative zero
0n           // BigInt zero
''           // empty string
null         // null
undefined    // undefined
NaN          // Not a Number

// Everything else is truthy, including:
true
1, -1, 3.14   // any non-zero number
'0', 'false'  // any non-empty string
[]            // empty array
{}            // empty object
function(){}  // functions

// Usage in conditions
if (value) { } // checks truthiness

// Common mistakes
if (value == true) { } // DON'T - fails for truthy values
if (value === true) { } // DON'T - only true for boolean true
if (value) { } // DO - checks truthiness
if (Boolean(value)) { } // DO - explicit

// Checking for empty array
if (arr.length > 0) { } // DO
if (arr) { } // DON'T - always truthy, even if empty
```

### Question 3: String Coercion

**Expected Answer:**

```javascript
// Implicit string coercion with +
console.log('5' + 3); // '53' (number to string)
console.log('5' + true); // '5true'
console.log('5' + null); // '5null'
console.log('5' + undefined); // '5undefined'

// Explicit conversion
String(123); // '123'
(123).toString(); // '123'
'' + 123; // '123' (implicit)

// Template literals
`Value: ${123}`; // 'Value: 123'

// Objects and arrays
console.log('Result: ' + [1, 2, 3]); // 'Result: 1,2,3'
console.log('Result: ' + { a: 1 }); // 'Result: [object Object]'

// Custom toString
const obj = {
  toString() {
    return 'Custom string';
  }
};
console.log('Value: ' + obj); // 'Value: Custom string'
```

### Question 4: Number Coercion

**Expected Answer:**

```javascript
// Implicit number coercion with -, *, /, %
console.log('5' - 3); // 2
console.log('5' * 2); // 10
console.log('5' / 2); // 2.5
console.log('10' % 3); // 1

// Unary plus
console.log(+'5'); // 5
console.log(+true); // 1
console.log(+false); // 0
console.log(+null); // 0
console.log(+undefined); // NaN
console.log(+''); // 0
console.log(+'  '); // 0
console.log(+'5px'); // NaN

// Explicit conversion
Number('123'); // 123
parseInt('123px'); // 123 (stops at first non-digit)
parseFloat('3.14'); // 3.14

// parseInt with radix
parseInt('10', 2); // 2 (binary)
parseInt('10', 8); // 8 (octal)
parseInt('10', 16); // 16 (hex)

// Special cases
Number(null); // 0
Number(undefined); // NaN
Number(true); // 1
Number(false); // 0
Number(''); // 0
Number('  '); // 0
```

### Question 5: Boolean Coercion

**Expected Answer:**

```javascript
// Explicit conversion
Boolean(1); // true
Boolean(0); // false
Boolean(''); // false
Boolean('text'); // true

// Implicit with !!
!!1; // true
!!0; // false
!!'text'; // true
!!''; // false

// Logical operators
true && true; // true
true && false; // false
true || false; // true
false || false; // false

// Short-circuit evaluation
const value = someValue || 'default'; // default if falsy
const value = someValue && processValue(); // process if truthy

// Nullish coalescing (ES2020)
const value = null ?? 'default'; // 'default'
const value = 0 ?? 'default'; // 0 (not 'default')
const value = '' ?? 'default'; // '' (not 'default')
```

### Question 6: Object to Primitive Conversion

**Expected Answer:**

```javascript
// valueOf and toString methods
const obj = {
  valueOf() {
    return 42;
  },
  toString() {
    return 'Object';
  }
};

console.log(obj + 1); // 43 (uses valueOf)
console.log(String(obj)); // 'Object' (uses toString)

// Symbol.toPrimitive (ES6)
const obj2 = {
  [Symbol.toPrimitive](hint) {
    if (hint === 'number') return 42;
    if (hint === 'string') return 'forty-two';
    return true; // default
  }
};

console.log(+obj2); // 42
console.log(`${obj2}`); // 'forty-two'
console.log(obj2 + ''); // 'true'

// Date objects
const date = new Date('2023-01-01');
console.log(+date); // timestamp
console.log(date + ''); // date string
```

### Question 7: Array Coercion

**Expected Answer:**

```javascript
// Array to string
[1, 2, 3] + ''; // '1,2,3'
String([1, 2, 3]); // '1,2,3'
[1, 2, 3].toString(); // '1,2,3'

// Array to number
+[]; // 0
+[5]; // 5
+[1, 2]; // NaN

// Array in boolean context
if ([]) { } // true (arrays are truthy)
if ([].length) { } // false (length is 0)

// Array equality
[] == []; // false (different references)
[] == false; // true (coercion)
[] == 0; // true
[] == ''; // true

// Common mistakes
const arr = [];
if (arr) { } // Always true
if (arr.length > 0) { } // Correct way
```

### Question 8: NaN and Type Checking

**Expected Answer:**

```javascript
// NaN (Not a Number)
console.log(typeof NaN); // 'number' (ironically)
console.log(NaN === NaN); // false (NaN is not equal to itself)

// Checking for NaN
isNaN(NaN); // true
isNaN('hello'); // true (coerces to NaN)
isNaN(undefined); // true

// Better: Number.isNaN (ES6)
Number.isNaN(NaN); // true
Number.isNaN('hello'); // false (no coercion)

// typeof operator
typeof 42; // 'number'
typeof 'text'; // 'string'
typeof true; // 'boolean'
typeof undefined; // 'undefined'
typeof null; // 'object' (historical bug)
typeof {}; // 'object'
typeof []; // 'object' (arrays are objects)
typeof function(){}; // 'function'

// Better type checking
Array.isArray([]); // true
value instanceof Array; // true for arrays
Object.prototype.toString.call(value); // '[object Array]'
```

## Advanced Questions

### Question 9: Predict Output - Complex Coercion

**Interview Question:**

```javascript
console.log([] + []); // ?
console.log([] + {}); // ?
console.log({} + []); // ?
console.log({} + {}); // ?
console.log(true + false); // ?
console.log([1, 2] + [3, 4]); // ?
console.log('5' + - '2'); // ?
console.log(null + undefined); // ?
```

**Answers:**

```javascript
console.log([] + []); // '' (both to empty string)
console.log([] + {}); // '[object Object]'
console.log({} + []); // 0 or '[object Object]' (depends on context)
console.log({} + {}); // '[object Object][object Object]' or NaN
console.log(true + false); // 1 (true=1, false=0)
console.log([1, 2] + [3, 4]); // '1,23,4'
console.log('5' + - '2'); // '5-2' (unary minus then concat)
console.log(null + undefined); // NaN
```

### Question 10: Comparison Coercion

**Expected Answer:**

```javascript
// <, >, <=, >= convert to primitives
console.log('2' > '12'); // true (string comparison)
console.log('2' > 12); // false (converts to number)
console.log(2 > '12'); // false (converts to number)

// null and undefined special cases
null == undefined; // true
null === undefined; // false
null == 0; // false (special case, not converted)
undefined == 0; // false

// Object comparison
{} == {}; // false (different references)
[] == []; // false (different references)

// Comparison algorithm
'abc' < 'abd'; // true (lexicographic)
'2' < '12'; // false (lexicographic, '2' > '1')
2 < '12'; // true (converts to numbers)

// Common pitfalls
const a = [1, 2];
const b = [1, 2];
a == b; // false (different references)
a.toString() == b.toString(); // true (both '1,2')
```

## What Interviewers Look For

### 1. Core Understanding
- Difference between == and ===
- Truthy/falsy values
- Implicit vs explicit coercion
- Type conversion rules

### 2. Practical Knowledge
- Common coercion pitfalls
- When coercion happens
- How to prevent bugs
- Type checking methods

### 3. Best Practices
- Always use ===
- Explicit conversions
- Proper type checking
- Avoiding coercion bugs

### 4. Advanced Knowledge
- Object to primitive conversion
- Symbol.toPrimitive
- Comparison algorithm
- Edge cases and quirks

## Red Flags to Avoid

### 1. Using == Instead of ===
- Using loose equality
- Not understanding coercion
- Relying on implicit conversion

### 2. Not Knowing Falsy Values
- Thinking [] is falsy
- Not checking array length
- Confusing 0, '', false

### 3. Type Checking Errors
- Using typeof null === 'object'
- Not using Array.isArray
- Confusing NaN checks

### 4. Coercion Bugs
- String concatenation surprises
- Unintended number conversion
- Comparison pitfalls

## Key Takeaways

### Essential Rules
1. **Use === always**: Avoid coercion bugs
2. **Know falsy values**: 8 values only
3. **Explicit is better**: Number(), String(), Boolean()
4. **Arrays are truthy**: Even when empty
5. **typeof null is 'object'**: Historical bug

### Best Practices
1. **Strict equality**: Use ===, !==
2. **Explicit conversion**: Number(x), String(x)
3. **Proper type checks**: Array.isArray, instanceof
4. **Nullish coalescing**: Use ?? over ||
5. **ESLint rules**: Enforce strict equality

### Common Patterns
1. **Default values**: `value || 'default'` or `value ?? 'default'`
2. **Boolean conversion**: `!!value`
3. **Number conversion**: `+value` or `Number(value)`
4. **String conversion**: `${value}` or `String(value)`
5. **Array check**: `Array.isArray(value)`

### Interview Success Tips
1. **Explain === vs ==**: Show coercion understanding
2. **List falsy values**: All 8 of them
3. **Predict outputs**: Walk through coercion steps
4. **Discuss best practices**: Always ===
5. **Mention ESLint**: Tool enforcement
6. **Show real bugs**: Examples you've seen

Remember: Type coercion is a powerful but dangerous feature. Understanding it prevents bugs, but avoiding it (with strict equality and explicit conversion) prevents even more bugs. Modern JavaScript has better tools (===, ??, optional chaining) that reduce reliance on coercion.
