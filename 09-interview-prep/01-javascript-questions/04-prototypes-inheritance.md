# Prototypes and Inheritance - JavaScript Interview Questions

## Table of Contents
- [Core Concepts](#core-concepts)
- [Common Interview Questions](#common-interview-questions)
- [Advanced Questions](#advanced-questions)
- [Practical Examples](#practical-examples)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Red Flags to Avoid](#red-flags-to-avoid)
- [Key Takeaways](#key-takeaways)

## Core Concepts

### What is a Prototype?

In JavaScript, every object has an internal property called `[[Prototype]]` (accessed via `__proto__` or `Object.getPrototypeOf()`). This prototype is a reference to another object, creating a chain used for property lookup.

### Prototype Chain

When you access a property on an object, JavaScript:
1. Looks for the property on the object itself
2. If not found, looks on the object's prototype
3. Continues up the chain until reaching `Object.prototype`
4. Returns `undefined` if not found anywhere

### Key Points
- All functions have a `prototype` property
- Objects inherit from their constructor's `prototype`
- The prototype chain ends at `Object.prototype`
- Prototype is used for method sharing and inheritance

## Common Interview Questions

### Question 1: Explain Prototypes and the Prototype Chain

**Expected Answer:**

Every JavaScript object has a hidden internal property `[[Prototype]]` that references another object. This creates a chain of objects that JavaScript uses for property and method lookup.

```javascript
// Constructor function
function Person(name) {
  this.name = name;
}

// Add method to prototype
Person.prototype.sayHello = function() {
  console.log(`Hello, I'm ${this.name}`);
};

const john = new Person('John');

// Property lookup chain:
// 1. john.sayHello - not found on john
// 2. john.__proto__.sayHello - found on Person.prototype
john.sayHello(); // "Hello, I'm John"

// Understanding the chain
console.log(john.__proto__ === Person.prototype); // true
console.log(Person.prototype.__proto__ === Object.prototype); // true
console.log(Object.prototype.__proto__ === null); // true (end of chain)
```

**Visualizing the Chain:**
```javascript
john
  |
  |__proto__ --> Person.prototype
                    |
                    |__proto__ --> Object.prototype
                                      |
                                      |__proto__ --> null
```

**What Interviewers Look For:**
- Understanding of prototype chain lookup
- Knowledge of `__proto__` vs `prototype`
- Explanation of how methods are shared
- Understanding that the chain ends at null

### Question 2: What's the Difference Between `__proto__` and `prototype`?

**Expected Answer:**

These are two different properties that are often confused:

**`prototype`:**
- Property of constructor functions
- Used as the prototype for objects created with `new`
- Only functions have this property

**`__proto__`:**
- Property of all objects
- References the object's prototype
- Part of the prototype chain

```javascript
function Animal(name) {
  this.name = name;
}

Animal.prototype.speak = function() {
  console.log(`${this.name} makes a sound`);
};

const dog = new Animal('Dog');

// prototype - on constructor
console.log(typeof Animal.prototype); // 'object'
console.log(Animal.prototype.speak); // function

// __proto__ - on instance
console.log(dog.__proto__ === Animal.prototype); // true
console.log(dog.prototype); // undefined (instances don't have prototype)

// Setting prototype
function Cat(name) {
  this.name = name;
}

Cat.prototype = Object.create(Animal.prototype);
Cat.prototype.constructor = Cat;

const cat = new Cat('Fluffy');
console.log(cat.__proto__ === Cat.prototype); // true
console.log(cat.__proto__.__proto__ === Animal.prototype); // true
```

**Modern Alternative (Preferred):**
```javascript
// Instead of __proto__, use:
Object.getPrototypeOf(dog); // Get prototype
Object.setPrototypeOf(obj, prototype); // Set prototype (avoid, slow)
Object.create(prototype); // Create with specific prototype
```

### Question 3: Implement Inheritance Using Prototypes

**Interview Question:** "Show me three ways to implement inheritance in JavaScript."

**Method 1: Classical Prototype Inheritance**
```javascript
// Parent constructor
function Animal(name) {
  this.name = name;
}

Animal.prototype.eat = function() {
  console.log(`${this.name} is eating`);
};

// Child constructor
function Dog(name, breed) {
  Animal.call(this, name); // Call parent constructor
  this.breed = breed;
}

// Inherit from Animal
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

// Add Dog-specific methods
Dog.prototype.bark = function() {
  console.log(`${this.name} barks`);
};

const myDog = new Dog('Buddy', 'Golden Retriever');
myDog.eat();  // "Buddy is eating" (inherited)
myDog.bark(); // "Buddy barks" (own method)

console.log(myDog instanceof Dog);    // true
console.log(myDog instanceof Animal); // true
```

**Method 2: ES6 Classes**
```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }
  
  eat() {
    console.log(`${this.name} is eating`);
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name); // Call parent constructor
    this.breed = breed;
  }
  
  bark() {
    console.log(`${this.name} barks`);
  }
  
  // Override parent method
  eat() {
    super.eat(); // Call parent method
    console.log('Dog finished eating');
  }
}

const myDog = new Dog('Buddy', 'Golden Retriever');
myDog.eat();  // Uses overridden method
myDog.bark(); // "Buddy barks"
```

**Method 3: Object.create() Pattern**
```javascript
const Animal = {
  init(name) {
    this.name = name;
    return this;
  },
  
  eat() {
    console.log(`${this.name} is eating`);
  }
};

const Dog = Object.create(Animal);

Dog.init = function(name, breed) {
  Animal.init.call(this, name);
  this.breed = breed;
  return this;
};

Dog.bark = function() {
  console.log(`${this.name} barks`);
};

const myDog = Object.create(Dog).init('Buddy', 'Golden Retriever');
myDog.eat();  // "Buddy is eating"
myDog.bark(); // "Buddy barks"
```

**What Interviewers Look For:**
- Understanding of multiple inheritance patterns
- Proper use of `call()` for parent constructor
- Correct prototype chain setup
- Knowledge of `super` in classes
- Understanding of `instanceof`

### Question 4: What is Object.create() and How Does it Differ from Constructor Functions?

**Expected Answer:**

`Object.create()` creates a new object with the specified prototype, providing more direct control over the prototype chain.

**Object.create():**
```javascript
const personProto = {
  greet() {
    console.log(`Hello, I'm ${this.name}`);
  }
};

const john = Object.create(personProto);
john.name = 'John';
john.greet(); // "Hello, I'm John"

console.log(Object.getPrototypeOf(john) === personProto); // true
```

**Constructor Function:**
```javascript
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function() {
  console.log(`Hello, I'm ${this.name}`);
};

const jane = new Person('Jane');
jane.greet(); // "Hello, I'm Jane"
```

**Key Differences:**

**1. Initialization:**
```javascript
// Object.create - manual initialization
const obj1 = Object.create(proto);
obj1.property = value;

// Constructor - automatic initialization
const obj2 = new Constructor(value);
```

**2. Prototype Assignment:**
```javascript
// Object.create - direct prototype assignment
const child = Object.create(parent);

// Constructor - indirect through prototype property
function Child() {}
Child.prototype = Object.create(Parent.prototype);
```

**3. Property Descriptors:**
```javascript
// Object.create accepts property descriptors
const obj = Object.create(proto, {
  name: {
    value: 'John',
    writable: true,
    enumerable: true,
    configurable: true
  }
});
```

**4. Creating Objects Without Prototype:**
```javascript
// Create object with no prototype
const pureObject = Object.create(null);
console.log(pureObject.toString); // undefined
// Useful for dictionaries/hash maps
```

**Practical Comparison:**
```javascript
// Factory pattern with Object.create
function createPerson(name, age) {
  const person = Object.create(personMethods);
  person.name = name;
  person.age = age;
  return person;
}

const personMethods = {
  greet() {
    console.log(`Hi, I'm ${this.name}`);
  }
};

// vs Constructor pattern
function Person(name, age) {
  this.name = name;
  this.age = age;
}

Person.prototype.greet = function() {
  console.log(`Hi, I'm ${this.name}`);
};
```

### Question 5: Explain How `new` Keyword Works

**Interview Question:** "What happens when you use the `new` keyword? Can you implement your own version?"

**Expected Answer:**

When you use `new`, JavaScript performs these steps:

1. Creates a new empty object
2. Sets the object's prototype to constructor's prototype
3. Executes the constructor with `this` bound to the new object
4. Returns the new object (unless constructor returns an object)

```javascript
function Person(name) {
  this.name = name;
}

const john = new Person('John');

// What happens under the hood:
// 1. const john = {};
// 2. john.__proto__ = Person.prototype;
// 3. Person.call(john, 'John');
// 4. return john;
```

**Custom Implementation:**
```javascript
function myNew(constructor, ...args) {
  // 1. Create new object with constructor's prototype
  const obj = Object.create(constructor.prototype);
  
  // 2. Execute constructor with new object as context
  const result = constructor.apply(obj, args);
  
  // 3. Return object from constructor if it's an object, otherwise return new object
  return (typeof result === 'object' && result !== null) ? result : obj;
}

// Test
function Person(name, age) {
  this.name = name;
  this.age = age;
}

Person.prototype.greet = function() {
  console.log(`Hello, I'm ${this.name}`);
};

const person1 = new Person('John', 30);
const person2 = myNew(Person, 'Jane', 25);

person1.greet(); // "Hello, I'm John"
person2.greet(); // "Hello, I'm Jane"

console.log(person1 instanceof Person); // true
console.log(person2 instanceof Person); // true
```

**Edge Cases:**
```javascript
// Constructor returning an object
function Person(name) {
  this.name = name;
  return { custom: 'object' }; // This will be returned instead
}

const p = new Person('John');
console.log(p.name); // undefined
console.log(p.custom); // 'object'

// Constructor returning primitive
function Animal(name) {
  this.name = name;
  return 'string'; // Ignored, new object returned
}

const a = new Animal('Dog');
console.log(a.name); // 'Dog'
```

## Advanced Questions

### Question 6: Prototypal vs Classical Inheritance

**Interview Question:** "Compare prototypal inheritance in JavaScript with classical inheritance in other languages."

**Expected Answer:**

**Classical Inheritance (Java, C++):**
- Classes are blueprints
- Objects are instances of classes
- Inheritance is class-to-class
- Copy of methods to each instance (or virtual table)

**Prototypal Inheritance (JavaScript):**
- Objects inherit directly from objects
- No classes (ES6 classes are syntactic sugar)
- Inheritance is object-to-object via prototype chain
- Methods are shared via prototype

**Demonstration:**
```javascript
// Classical approach (ES6 classes - syntactic sugar)
class Vehicle {
  constructor(type) {
    this.type = type;
  }
  
  move() {
    console.log(`${this.type} is moving`);
  }
}

class Car extends Vehicle {
  constructor(brand) {
    super('Car');
    this.brand = brand;
  }
}

// Pure prototypal approach
const vehicle = {
  move() {
    console.log(`${this.type} is moving`);
  }
};

const car = Object.create(vehicle);
car.type = 'Car';
car.brand = 'Toyota';

// Both achieve similar results
const myCar1 = new Car('Toyota');
myCar1.move(); // "Car is moving"

car.move(); // "Car is moving"
```

**Prototypal Advantages:**
```javascript
// Dynamic inheritance changes
const animal = {
  eat() { console.log('eating'); }
};

const dog = Object.create(animal);

// Add method to prototype at runtime
animal.sleep = function() {
  console.log('sleeping');
};

dog.sleep(); // Works! Dynamically inherited

// Multiple "inheritance" (mixin pattern)
const canSwim = {
  swim() { console.log('swimming'); }
};

const canFly = {
  fly() { console.log('flying'); }
};

const duck = Object.create(animal);
Object.assign(duck, canSwim, canFly);

duck.eat();  // From animal
duck.swim(); // From canSwim
duck.fly();  // From canFly
```

### Question 7: Implement Deep Clone with Prototype Preservation

**Interview Question:** "Implement a deep clone function that preserves the prototype chain."

**Solution:**
```javascript
function deepClone(obj, hash = new WeakMap()) {
  // Handle primitives and null
  if (obj === null || typeof obj !== 'object') {
    return obj;
  }
  
  // Handle circular references
  if (hash.has(obj)) {
    return hash.get(obj);
  }
  
  // Handle Date
  if (obj instanceof Date) {
    return new Date(obj);
  }
  
  // Handle RegExp
  if (obj instanceof RegExp) {
    return new RegExp(obj);
  }
  
  // Handle Array
  if (Array.isArray(obj)) {
    const arrCopy = [];
    hash.set(obj, arrCopy);
    obj.forEach((item, index) => {
      arrCopy[index] = deepClone(item, hash);
    });
    return arrCopy;
  }
  
  // Handle Object - preserve prototype
  const objCopy = Object.create(Object.getPrototypeOf(obj));
  hash.set(obj, objCopy);
  
  // Copy all properties
  for (const key in obj) {
    if (obj.hasOwnProperty(key)) {
      objCopy[key] = deepClone(obj[key], hash);
    }
  }
  
  return objCopy;
}

// Test with prototype
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function() {
  console.log(`Hello, ${this.name}`);
};

const original = new Person('John');
const cloned = deepClone(original);

cloned.greet(); // "Hello, John"
console.log(cloned instanceof Person); // true
console.log(Object.getPrototypeOf(cloned) === Person.prototype); // true
```

### Question 8: Property Shadowing and Prototype Pollution

**Interview Question:** "Explain property shadowing and prototype pollution vulnerabilities."

**Property Shadowing:**
```javascript
const parent = {
  name: 'parent',
  greet() {
    console.log(`Hello from ${this.name}`);
  }
};

const child = Object.create(parent);
child.name = 'child'; // Shadows parent.name

console.log(child.name); // 'child' (own property)
console.log(parent.name); // 'parent' (unchanged)

child.greet(); // "Hello from child" (uses shadowed name)

// Delete shadowed property
delete child.name;
console.log(child.name); // 'parent' (now uses prototype)
```

**Prototype Pollution Attack:**
```javascript
// Vulnerable code
function merge(target, source) {
  for (let key in source) {
    target[key] = source[key];
  }
  return target;
}

// Attack
const malicious = JSON.parse('{"__proto__": {"isAdmin": true}}');
merge({}, malicious);

// Now ALL objects have isAdmin
const user = {};
console.log(user.isAdmin); // true (!)

// Fixed version
function safeMerge(target, source) {
  for (let key in source) {
    if (source.hasOwnProperty(key) && key !== '__proto__') {
      target[key] = source[key];
    }
  }
  return target;
}

// Or use Object.assign (safe)
Object.assign({}, malicious); // Ignores __proto__

// Or freeze Object.prototype
Object.freeze(Object.prototype);
```

**Protection Strategies:**
```javascript
// 1. Use Object.create(null) for data objects
const safeObj = Object.create(null);
safeObj.__proto__ = 'whatever'; // Just a regular property
console.log(Object.getPrototypeOf(safeObj)); // null

// 2. Use Map for key-value storage
const map = new Map();
map.set('__proto__', 'safe');
console.log(map.get('__proto__')); // 'safe' (no pollution)

// 3. Validate keys
function isValidKey(key) {
  return key !== '__proto__' && 
         key !== 'constructor' && 
         key !== 'prototype';
}

// 4. Use Object.hasOwn (or hasOwnProperty)
const obj = Object.create(parent);
obj.name = 'child';

console.log(Object.hasOwn(obj, 'name')); // true
console.log(Object.hasOwn(obj, 'greet')); // false (on prototype)
```

## Practical Examples

### Example 1: Plugin System with Prototypes

```javascript
// Base plugin system
function PluginSystem() {
  this.plugins = [];
}

PluginSystem.prototype.register = function(plugin) {
  this.plugins.push(plugin);
};

PluginSystem.prototype.execute = function() {
  this.plugins.forEach(plugin => plugin.run());
};

// Individual plugins inherit from base Plugin
function Plugin(name) {
  this.name = name;
}

Plugin.prototype.run = function() {
  console.log(`${this.name} is running`);
};

// Specific plugin types
function LoggerPlugin(name, level) {
  Plugin.call(this, name);
  this.level = level;
}

LoggerPlugin.prototype = Object.create(Plugin.prototype);
LoggerPlugin.prototype.constructor = LoggerPlugin;

LoggerPlugin.prototype.run = function() {
  console.log(`[${this.level}] ${this.name} logging`);
};

// Usage
const system = new PluginSystem();
system.register(new Plugin('Basic'));
system.register(new LoggerPlugin('Advanced', 'INFO'));
system.execute();
```

### Example 2: Method Borrowing

```javascript
const arrayLike = {
  0: 'a',
  1: 'b',
  2: 'c',
  length: 3
};

// Borrow array methods
const result = Array.prototype.map.call(arrayLike, item => item.toUpperCase());
console.log(result); // ['A', 'B', 'C']

// ES6 alternative
Array.from(arrayLike).map(item => item.toUpperCase());

// Create reusable borrowed methods
const slice = Array.prototype.slice;
const push = Array.prototype.push;

function MyArrayLike() {
  this.length = 0;
}

MyArrayLike.prototype.push = function(...items) {
  return push.apply(this, items);
};

MyArrayLike.prototype.slice = function(start, end) {
  return slice.call(this, start, end);
};

const myArray = new MyArrayLike();
myArray.push('a', 'b', 'c');
console.log(myArray.length); // 3
console.log(myArray.slice(0, 2)); // ['a', 'b']
```

### Example 3: Trait/Mixin Pattern

```javascript
// Define traits
const canEat = {
  eat(food) {
    console.log(`${this.name} is eating ${food}`);
  }
};

const canSwim = {
  swim() {
    console.log(`${this.name} is swimming`);
  }
};

const canFly = {
  fly() {
    console.log(`${this.name} is flying`);
  }
};

// Mixin function
function mixin(target, ...sources) {
  Object.assign(target, ...sources);
  return target;
}

// Create animals with different traits
function Duck(name) {
  this.name = name;
}

mixin(Duck.prototype, canEat, canSwim, canFly);

function Fish(name) {
  this.name = name;
}

mixin(Fish.prototype, canEat, canSwim);

const duck = new Duck('Donald');
duck.eat('bread'); // "Donald is eating bread"
duck.swim();       // "Donald is swimming"
duck.fly();        // "Donald is flying"

const fish = new Fish('Nemo');
fish.eat('plankton'); // "Nemo is eating plankton"
fish.swim();          // "Nemo is swimming"
// fish.fly();        // TypeError: fish.fly is not a function
```

## What Interviewers Look For

### 1. Core Understanding
- Clear explanation of prototype chain
- Difference between `__proto__` and `prototype`
- Understanding of property lookup
- Knowledge of prototype-based inheritance

### 2. Practical Skills
- Can implement inheritance correctly
- Knows multiple inheritance patterns
- Understands when to use each pattern
- Can work with Object.create()

### 3. Advanced Knowledge
- Understanding of `new` keyword internals
- Knowledge of prototype pollution
- Can implement custom inheritance utilities
- Familiar with property descriptors

### 4. Modern JavaScript
- ES6 class syntax
- Understanding that classes are syntactic sugar
- When to use classes vs factory functions
- Knowledge of modern alternatives

### 5. Best Practices
- Proper prototype chain setup
- Constructor restoration
- Memory efficiency through shared methods
- Security awareness (prototype pollution)

## Red Flags to Avoid

### 1. Fundamental Misunderstandings
- Confusing `__proto__` and `prototype`
- Not understanding prototype chain lookup
- Thinking classes create new inheritance model

### 2. Common Mistakes
- Forgetting to restore constructor
- Modifying Object.prototype directly
- Using `__proto__` in production code
- Not calling parent constructor

### 3. Security Ignorance
- Unaware of prototype pollution
- Not validating object keys
- Trusting user input for object properties

### 4. Performance Issues
- Creating methods in constructor (not on prototype)
- Unnecessary use of Object.setPrototypeOf
- Not leveraging prototype for method sharing

### 5. Code Quality
- Inconsistent inheritance patterns
- Overly complex prototype chains
- Not using modern alternatives when appropriate

## Key Takeaways

### Essential Concepts
1. **Every object has a prototype**: Accessed via `__proto__` or `Object.getPrototypeOf()`
2. **Functions have prototype property**: Used when creating instances with `new`
3. **Prototype chain enables inheritance**: Methods are shared, not copied
4. **Property lookup walks the chain**: Stops at first match or null
5. **Classes are syntactic sugar**: Built on prototype system

### Best Practices
1. **Use Object.create() for inheritance**: More explicit than constructor manipulation
2. **Share methods via prototype**: Don't define methods in constructor
3. **Prefer ES6 classes**: More readable and familiar to developers
4. **Validate object keys**: Protect against prototype pollution
5. **Use Object.hasOwn**: Check own properties vs inherited

### Common Patterns
1. **Constructor pattern**: Traditional object creation
2. **Factory pattern**: Object.create() based
3. **Class pattern**: ES6 syntax
4. **Mixin pattern**: Composition over inheritance
5. **Module pattern**: Encapsulation with prototypes

### Interview Success Tips
1. **Draw diagrams**: Visualize prototype chains
2. **Explain with examples**: Show actual code
3. **Discuss trade-offs**: When to use each pattern
4. **Mention security**: Prototype pollution awareness
5. **Show modern knowledge**: ES6 classes and alternatives
6. **Explain internals**: How `new` and inheritance work

Remember: Prototypes are the foundation of JavaScript's object system. Understanding them deeply shows mastery of the language's core mechanics, even as modern syntax provides more convenient abstractions.
