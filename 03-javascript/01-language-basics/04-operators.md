# JavaScript Operators: Logical, Nullish, Optional Chaining, and Spread/Rest

## The Idea

**In plain English:** Operators are special symbols that tell JavaScript how to combine, compare, or transform values — think of them as the verbs of the language. Modern JavaScript adds smart operators that handle missing values and safely dig into complex data without crashing.

**Real-world analogy:** Imagine you are filling out a form at a doctor's office. Some fields are optional — if you leave "Middle Name" blank, the clerk writes "N/A" as a default. If a section like "Emergency Contact" is missing entirely, the clerk skips every sub-field (phone, address) rather than having a meltdown.

- The form fields = variables and object properties in your code
- The clerk checking "is this blank?" = the nullish coalescing operator (`??`) picking a default only when a value is truly missing
- The clerk skipping sub-fields when a whole section is absent = optional chaining (`?.`) stopping safely instead of throwing an error
- Filling a new form by copying all fields from an old one and changing just a few = the spread operator (`...`) copying and updating objects

---

## Overview

JavaScript operators are symbols that perform operations on operands (values and variables). Beyond basic arithmetic and comparison operators, JavaScript includes powerful modern operators that handle null/undefined values, safely access nested properties, and work with collections. Understanding these operators is crucial for writing clean, defensive code.

## Logical Operators

### AND (&&)

Returns the first falsy value or the last value if all are truthy:

```javascript
// Basic usage
true && true;           // true
true && false;          // false
false && true;          // false

// Short-circuit evaluation
console.log("first") && console.log("second");
// Logs: "first", "second" (both execute)

false && console.log("won't run");
// Doesn't log (short-circuits)

// Returns actual values, not just booleans
5 && 10;                // 10 (both truthy, return last)
0 && 5;                 // 0 (first falsy)
"" && "hello";          // "" (first falsy)
"a" && "b" && "c";      // "c" (all truthy, return last)
```

### Common && Patterns

```javascript
// Conditional execution
isLoggedIn && renderProfile();

// Guard pattern
user && user.address && console.log(user.address.street);

// Default values (old pattern, replaced by ??)
const value = options && options.timeout && options.timeout || 5000;

// React conditional rendering
{isLoading && <Spinner />}
{hasError && <ErrorMessage error={error} />}
```

### OR (||)

Returns the first truthy value or the last value if all are falsy:

```javascript
// Basic usage
true || false;          // true
false || true;          // true
false || false;         // false

// Short-circuit evaluation
true || console.log("won't run");
false || console.log("will run");

// Returns actual values
5 || 10;                // 5 (first truthy)
0 || 5;                 // 5 (0 is falsy)
"" || "default";        // "default"
null || undefined || "value";  // "value"
```

### Common || Patterns

```javascript
// Default values (legacy pattern)
function greet(name) {
  name = name || "Guest";  // Problem: rejects falsy values like ""
  return `Hello, ${name}`;
}

// Chaining fallbacks
const value = localStorage.getItem('key') || sessionStorage.getItem('key') || 'default';

// Configuration merging
const config = {
  timeout: options.timeout || 5000,
  retries: options.retries || 3
};
```

### NOT (!)

Converts to boolean and negates:

```javascript
// Single negation (convert to boolean and negate)
!true;              // false
!false;             // true
!0;                 // true
!"hello";           // false
!null;              // true

// Double negation (convert to boolean)
!!0;                // false
!!"hello";          // true
!![];               // true
!!{};               // true

// Practical usage
if (!user) {
  throw new Error("User not found");
}

// Type checking
const isString = typeof value === 'string';
const isNotString = typeof value !== 'string';
```

### Combining Logical Operators

```javascript
// Precedence: ! > && > ||
true || false && false;     // true (same as: true || (false && false))
!true || false && true;     // false (same as: (!true) || (false && true))

// Use parentheses for clarity
const canAccess = (isLoggedIn && isAdmin) || isDeveloper;

// Complex conditions
const isValid = 
  value !== null &&
  value !== undefined &&
  (typeof value === 'string' || typeof value === 'number');
```

## Nullish Coalescing Operator (??)

Returns the right operand when the left is null or undefined (and ONLY null/undefined):

```javascript
// Basic usage
null ?? "default";           // "default"
undefined ?? "default";      // "default"

// Key difference from ||
0 ?? "default";              // 0 (0 is not nullish!)
"" ?? "default";             // "" (empty string is not nullish!)
false ?? "default";          // false (false is not nullish!)

// Compare with ||
0 || "default";              // "default" (0 is falsy)
"" || "default";             // "default" (empty string is falsy)
false || "default";          // "default" (false is falsy)
```

### Use Cases

```javascript
// Configuration with valid falsy values
const config = {
  port: userConfig.port ?? 3000,           // 0 is valid port
  verbose: userConfig.verbose ?? false,    // false is valid setting
  timeout: userConfig.timeout ?? 5000,     // 0 would be valid
  prefix: userConfig.prefix ?? "",         // empty string valid
};

// Form inputs
const formData = {
  quantity: inputValue ?? 1,  // 0 quantity is meaningful
  notes: textInput ?? "",     // empty string is valid
};

// API responses
const data = response.data ?? { items: [] };  // null response
```

### Cannot Chain with && or ||

```javascript
// Error: Must use parentheses
value ?? "default" || "fallback";  // SyntaxError

// Correct: Use parentheses
(value ?? "default") || "fallback";
value ?? ("default" || "fallback");
```

## Optional Chaining (?.)

Safely accesses nested properties without explicit null checks:

```javascript
// Traditional approach (verbose)
const street = user && user.address && user.address.street;

// Optional chaining (concise)
const street = user?.address?.street;

// Returns undefined if any part is null/undefined
const value = obj?.prop?.nested?.deep;  // undefined if any part nullish
```

### Property Access

```javascript
const user = {
  name: "Alice",
  address: {
    city: "Boston"
  }
};

// Safe property access
user?.name;                    // "Alice"
user?.address?.city;           // "Boston"
user?.address?.zip;            // undefined (property doesn't exist)
user?.contact?.email;          // undefined (contact is undefined)

// Stops at first nullish value
const nonExistent = null;
nonExistent?.prop?.nested;     // undefined (stops at nonExistent)
```

### Method Calls

```javascript
const user = {
  getName() {
    return "Alice";
  }
};

// Safe method calls
user?.getName?.();             // "Alice"
user?.getAge?.();              // undefined (method doesn't exist)
user?.address?.getZip?.();     // undefined (address undefined)

// Real-world example
const result = optional?.callback?.();
element?.addEventListener?.("click", handler);
```

### Array Element Access

```javascript
const arr = [1, 2, 3];

arr?.[0];                      // 1
arr?.[10];                     // undefined
arr?.[arr.length - 1];         // 3

// With null/undefined arrays
const nullArr = null;
nullArr?.[0];                  // undefined (not TypeError!)

// Dynamic property access
const prop = "name";
obj?.[prop];                   // Safe dynamic access
```

### Combining with Nullish Coalescing

```javascript
// Safe access with default value
const city = user?.address?.city ?? "Unknown";
const email = user?.contact?.email ?? "no-email@example.com";

// Chaining multiple operations
const result = data?.items?.[0]?.value ?? defaultValue;

// Configuration access
const timeout = config?.api?.timeout ?? 5000;
const retries = config?.api?.retries ?? 3;
```

### Common Patterns

```javascript
// API response handling
const title = response?.data?.article?.title ?? "Untitled";

// Event handling
const handleClick = (event) => {
  const value = event?.target?.value;
  const dataset = event?.currentTarget?.dataset;
};

// Deep object access
const getValue = (obj) => {
  return obj?.level1?.level2?.level3?.level4?.value;
};

// Function chains
const data = await fetch(url)
  ?.then(res => res.json())
  ?.catch(err => console.error(err));
```

### Short-Circuiting Behavior

```javascript
let x = 0;

// Does not execute if nullish
null?.prop.x++;         // undefined (x++ never executes)
undefined?.method(x++); // undefined (x++ never executes)

console.log(x);         // 0 (unchanged)

// Compares with regular chaining
obj.method(x++);        // TypeError if obj is null
                       // AND x is incremented before error!
```

## Spread Operator (...)

### Array Spreading

```javascript
// Creating new array with existing elements
const arr1 = [1, 2, 3];
const arr2 = [...arr1];           // [1, 2, 3] (shallow copy)

// Concatenating arrays
const arr1 = [1, 2];
const arr2 = [3, 4];
const combined = [...arr1, ...arr2];  // [1, 2, 3, 4]

// Adding elements
const original = [2, 3];
const withStart = [1, ...original];   // [1, 2, 3]
const withEnd = [...original, 4];     // [2, 3, 4]
const withBoth = [1, ...original, 4]; // [1, 2, 3, 4]
```

### Object Spreading

```javascript
// Creating new object (shallow copy)
const obj1 = { a: 1, b: 2 };
const obj2 = { ...obj1 };             // { a: 1, b: 2 }

// Merging objects (later properties override)
const defaults = { timeout: 5000, retries: 3 };
const custom = { retries: 5 };
const config = { ...defaults, ...custom };  // { timeout: 5000, retries: 5 }

// Adding properties
const user = { name: "Alice" };
const userWithAge = { ...user, age: 30 };   // { name: "Alice", age: 30 }

// Overriding properties
const original = { a: 1, b: 2 };
const modified = { ...original, b: 3 };     // { a: 1, b: 3 }
```

### Function Arguments

```javascript
// Spreading array into function arguments
const numbers = [1, 2, 3, 4, 5];
Math.max(...numbers);                 // 5 (same as Math.max(1, 2, 3, 4, 5))

// Practical examples
const dates = [new Date(2024, 1, 1), new Date(2024, 6, 1)];
const earliest = new Date(Math.min(...dates));

// Multiple spreads
const arrays = [[1, 2], [3, 4], [5, 6]];
const flattened = [].concat(...arrays);  // [1, 2, 3, 4, 5, 6]
```

### Common Patterns

```javascript
// Cloning arrays
const original = [1, 2, 3];
const clone = [...original];

// Removing duplicates
const withDuplicates = [1, 2, 2, 3, 3, 4];
const unique = [...new Set(withDuplicates)];  // [1, 2, 3, 4]

// Converting iterables to arrays
const str = "hello";
const chars = [...str];                  // ['h', 'e', 'l', 'l', 'o']

const nodeList = document.querySelectorAll('div');
const array = [...nodeList];

// Immutable updates in React
const [items, setItems] = useState([1, 2, 3]);
setItems([...items, 4]);                 // Add item
setItems(items.filter(x => x !== 2));    // Remove item
setItems([...items.slice(0, 1), 99, ...items.slice(2)]);  // Replace item

// Merging configurations
const defaultConfig = { theme: 'light', lang: 'en' };
const userConfig = { theme: 'dark' };
const finalConfig = { ...defaultConfig, ...userConfig };
// { theme: 'dark', lang: 'en' }
```

## Rest Operator (...)

Same syntax as spread, but used for gathering/collecting:

### Function Parameters

```javascript
// Gathering remaining arguments
function sum(...numbers) {
  return numbers.reduce((total, n) => total + n, 0);
}

sum(1, 2, 3, 4, 5);  // 15

// With regular parameters
function greet(greeting, ...names) {
  return `${greeting}, ${names.join(' and ')}!`;
}

greet("Hello", "Alice", "Bob", "Charlie");
// "Hello, Alice and Bob and Charlie!"

// Must be last parameter
function invalid(...rest, last) {}  // SyntaxError
function valid(first, ...rest) {}   // OK
```

### Array Destructuring

```javascript
// Gathering rest of array
const [first, ...rest] = [1, 2, 3, 4, 5];
console.log(first);  // 1
console.log(rest);   // [2, 3, 4, 5]

// Multiple destructuring
const [a, b, ...others] = [1, 2, 3, 4];
console.log(a);      // 1
console.log(b);      // 2
console.log(others); // [3, 4]

// Skipping elements
const [first, , third, ...rest] = [1, 2, 3, 4, 5];
console.log(first);  // 1
console.log(third);  // 3
console.log(rest);   // [4, 5]
```

### Object Destructuring

```javascript
// Gathering rest of object properties
const person = { name: "Alice", age: 30, city: "Boston", job: "Engineer" };
const { name, age, ...rest } = person;

console.log(name);   // "Alice"
console.log(age);    // 30
console.log(rest);   // { city: "Boston", job: "Engineer" }

// Removing properties
const { password, ...publicUser } = user;
// publicUser has all properties except password

// Useful pattern: extracting few, keeping rest
const { id, ...updateData } = formData;
await updateUser(id, updateData);
```

### Common Patterns

```javascript
// Wrapper functions
function myLogger(...args) {
  console.log('[LOG]', ...args);
}

myLogger('Error:', 404, 'Not found');  // [LOG] Error: 404 Not found

// Partial application
function multiply(multiplier, ...numbers) {
  return numbers.map(n => n * multiplier);
}

const double = (...nums) => multiply(2, ...nums);
double(1, 2, 3);  // [2, 4, 6]

// Collecting middleware arguments
function compose(...functions) {
  return (arg) => functions.reduceRight((result, fn) => fn(result), arg);
}

// Prop forwarding in React
function Button({ variant, ...props }) {
  return <button className={variant} {...props} />;
}
```

## Comparison: Spread vs Rest

```javascript
// SPREAD: Expands elements
const arr = [1, 2, 3];
console.log(...arr);        // 1 2 3 (spread)
const newArr = [...arr, 4]; // [1, 2, 3, 4] (spread)

// REST: Collects elements
function sum(...nums) {     // rest (collect)
  return nums.reduce((a, b) => a + b);
}

const [first, ...rest] = arr;  // rest (collect)
```

## Operator Precedence

Understanding precedence prevents bugs:

```javascript
// Precedence (high to low for these operators):
// 1. Optional chaining (?.)
// 2. NOT (!)
// 3. AND (&&)
// 4. OR (||)
// 5. Nullish coalescing (??)

// Examples
!true && false;           // false (! first, then &&)
true || false && false;   // true (same as: true || (false && false))

// ?? cannot be mixed with && or ||
value ?? 0 || 1;          // SyntaxError
(value ?? 0) || 1;        // OK
value ?? (0 || 1);        // OK

// Optional chaining
obj?.prop && other;       // OK (different operators)
obj?.prop ?? default;     // OK
```

## Best Practices

### 1. Use ?? for Default Values with Falsy Validity

```javascript
// Bad: Rejects valid falsy values
const count = input || 0;  // 0 input becomes 0
const enabled = flag || false;  // false becomes false

// Good: Only null/undefined get defaults
const count = input ?? 0;
const enabled = flag ?? false;
```

### 2. Use Optional Chaining for Deep Access

```javascript
// Bad: Verbose and error-prone
const street = user && user.address && user.address.street;

// Good: Concise and safe
const street = user?.address?.street;
```

### 3. Spread for Immutable Updates

```javascript
// Bad: Mutates original
const addItem = (arr, item) => {
  arr.push(item);
  return arr;
};

// Good: Creates new array
const addItem = (arr, item) => [...arr, item];
```

### 4. Rest Parameters Over arguments

```javascript
// Bad: Using arguments object
function sum() {
  return Array.from(arguments).reduce((a, b) => a + b);
}

// Good: Rest parameters
function sum(...nums) {
  return nums.reduce((a, b) => a + b);
}
```

### 5. Combine Operators Thoughtfully

```javascript
// Excellent pattern: safe access with default
const value = obj?.prop?.nested ?? defaultValue;

// Common React pattern
const title = data?.article?.title ?? "Untitled";
{title && <h1>{title}</h1>}
```

## Common Pitfalls

### 1. Shallow Copying with Spread

```javascript
const original = { a: 1, nested: { b: 2 } };
const copy = { ...original };

copy.nested.b = 99;
console.log(original.nested.b);  // 99 (nested object shared!)
```

### 2. Optional Chaining Doesn't Help with null returns

```javascript
const getUser = () => null;
const name = getUser()?.name;  // undefined (not the user's name!)

// Still need to handle null returns
const user = getUser();
const name = user?.name ?? "Unknown";
```

### 3. Spread Order Matters

```javascript
const obj = { a: 1, b: 2 };
const result1 = { ...obj, a: 99 };  // { a: 99, b: 2 } (a overridden)
const result2 = { a: 99, ...obj };  // { a: 1, b: 2 } (99 overridden)
```

## Resources

### Documentation

- [MDN: Logical Operators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_Operators)
- [MDN: Nullish Coalescing](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing)
- [MDN: Optional Chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining)
- [MDN: Spread Syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)
- [MDN: Rest Parameters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters)

### Articles

- [Optional Chaining in Depth](https://javascript.info/optional-chaining)
- [Nullish Coalescing Explained](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing_operator)
