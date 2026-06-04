# Prototypes and Inheritance

## Table of Contents
- [Overview](#overview)
- [Prototype Chain](#prototype-chain)
- [[[Prototype]] Internal Slot](#prototype-internal-slot)
- [Object.create](#objectcreate)
- [Constructor Functions](#constructor-functions)
- [Class Syntax Desugaring](#class-syntax-desugaring)
- [Prototype Methods vs Instance Methods](#prototype-methods-vs-instance-methods)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

JavaScript uses prototypal inheritance, not classical inheritance. Every object in JavaScript has an internal link to another object called its prototype. This prototype object has its own prototype, forming the prototype chain. Understanding prototypes is essential for mastering JavaScript's object model and inheritance mechanisms.

## Prototype Chain

The prototype chain is the mechanism by which JavaScript objects inherit properties and methods from other objects.

### Basic Prototype Chain

```javascript
const animal = {
  eats: true,
  walk() {
    console.log('Animal walks');
  }
};

const rabbit = {
  jumps: true
};

// Set rabbit's prototype to animal
Object.setPrototypeOf(rabbit, animal);

console.log(rabbit.eats);  // true (inherited from animal)
console.log(rabbit.jumps); // true (own property)
rabbit.walk();             // 'Animal walks' (inherited method)

// Prototype chain: rabbit → animal → Object.prototype → null
```

### Property Lookup in Prototype Chain

```javascript
const grandparent = {
  surname: 'Smith',
  greet() {
    console.log(`Hello, I'm ${this.name} ${this.surname}`);
  }
};

const parent = Object.create(grandparent);
parent.occupation = 'Engineer';

const child = Object.create(parent);
child.name = 'Alice';
child.age = 10;

// Property lookup traverses the chain
console.log(child.name);       // 'Alice' (own property)
console.log(child.occupation); // 'Engineer' (from parent)
console.log(child.surname);    // 'Smith' (from grandparent)
child.greet();                 // 'Hello, I'm Alice Smith'

// Chain: child → parent → grandparent → Object.prototype → null
```

### Shadowing Properties

```javascript
const parent = {
  value: 10,
  getValue() {
    return this.value;
  }
};

const child = Object.create(parent);
console.log(child.value); // 10 (inherited)

// Shadowing: Define own property with same name
child.value = 20;
console.log(child.value);        // 20 (own property shadows inherited)
console.log(parent.value);       // 10 (unchanged)
console.log(child.getValue());   // 20 ('this' points to child)

// Check own vs inherited
console.log(child.hasOwnProperty('value')); // true
delete child.value;
console.log(child.value); // 10 (inherited again after deletion)
```

### End of Prototype Chain

```javascript
const obj = {};

// Prototype chain for regular objects
console.log(Object.getPrototypeOf(obj) === Object.prototype); // true
console.log(Object.getPrototypeOf(Object.prototype)); // null (end of chain)

// Create object with no prototype
const noProto = Object.create(null);
console.log(Object.getPrototypeOf(noProto)); // null
console.log(noProto.toString); // undefined (no Object.prototype methods)

// Useful for pure dictionaries (no inherited properties)
const dictionary = Object.create(null);
dictionary['toString'] = 'safe'; // No conflict with Object.prototype.toString
```

## [[Prototype]] Internal Slot

Every JavaScript object has an internal property called `[[Prototype]]` that references its prototype.

### Accessing [[Prototype]]

```javascript
const obj = { value: 42 };

// Modern way: Object.getPrototypeOf()
const proto1 = Object.getPrototypeOf(obj);
console.log(proto1 === Object.prototype); // true

// Legacy way: __proto__ (accessor property, not recommended)
const proto2 = obj.__proto__;
console.log(proto2 === Object.prototype); // true

// Setting prototype
const animal = { eats: true };
const rabbit = {};

Object.setPrototypeOf(rabbit, animal); // Modern way
// rabbit.__proto__ = animal; // Legacy way (not recommended)

console.log(rabbit.eats); // true
```

### __proto__ vs prototype vs [[Prototype]]

```javascript
function User(name) {
  this.name = name;
}

const user = new User('Alice');

// [[Prototype]]: Internal slot (not directly accessible)
// Access via Object.getPrototypeOf(obj)

// __proto__: Accessor property on Object.prototype
// Gets/sets [[Prototype]] (legacy, use Object.getPrototypeOf/setPrototypeOf)
console.log(user.__proto__ === User.prototype); // true

// .prototype: Property on constructor functions
// Used as [[Prototype]] for objects created with 'new'
console.log(User.prototype); // { constructor: User }
console.log(Object.getPrototypeOf(user) === User.prototype); // true

// Relationship:
// user.[[Prototype]] === User.prototype
// User.prototype.constructor === User
```

### Object.getPrototypeOf vs __proto__

```javascript
const obj = {};

// GOOD: Object.getPrototypeOf (standard, performant)
const proto1 = Object.getPrototypeOf(obj);

// BAD: __proto__ (legacy, can be slower)
const proto2 = obj.__proto__;

// Setting prototype
// GOOD: Object.setPrototypeOf (standard, but slow)
const animal = { eats: true };
Object.setPrototypeOf(obj, animal);

// BAD: Mutating __proto__ (legacy, very slow)
obj.__proto__ = animal;

// BEST: Set prototype at creation (fastest)
const rabbit = Object.create(animal);
```

## Object.create

`Object.create()` creates a new object with the specified prototype object.

### Basic Usage

```javascript
const personProto = {
  greet() {
    console.log(`Hello, I'm ${this.name}`);
  },
  
  sayAge() {
    console.log(`I'm ${this.age} years old`);
  }
};

// Create object with personProto as prototype
const alice = Object.create(personProto);
alice.name = 'Alice';
alice.age = 30;

alice.greet();   // 'Hello, I'm Alice'
alice.sayAge();  // 'I'm 30 years old'

// Verify prototype
console.log(Object.getPrototypeOf(alice) === personProto); // true
```

### Object.create with Property Descriptors

```javascript
const proto = {
  describe() {
    return `${this.name}: ${this.value}`;
  }
};

const obj = Object.create(proto, {
  name: {
    value: 'MyObject',
    writable: true,
    enumerable: true,
    configurable: true
  },
  value: {
    value: 42,
    writable: false, // Read-only
    enumerable: true,
    configurable: false
  },
  computed: {
    get() {
      return this.value * 2;
    },
    enumerable: true,
    configurable: true
  }
});

console.log(obj.name);     // 'MyObject'
console.log(obj.value);    // 42
console.log(obj.computed); // 84
console.log(obj.describe()); // 'MyObject: 42'

obj.name = 'NewName';  // Works
obj.value = 100;       // Silently fails (not writable)
console.log(obj.value); // 42 (unchanged)
```

### Inheritance Pattern with Object.create

```javascript
// Parent "class"
const Animal = {
  init(name) {
    this.name = name;
    return this;
  },
  
  eat() {
    console.log(`${this.name} is eating`);
  }
};

// Child "class"
const Dog = Object.create(Animal);

Dog.bark = function() {
  console.log(`${this.name} barks: Woof!`);
};

Dog.init = function(name, breed) {
  Animal.init.call(this, name); // Call parent init
  this.breed = breed;
  return this;
};

// Create instances
const dog1 = Object.create(Dog).init('Buddy', 'Golden Retriever');
dog1.eat();  // 'Buddy is eating' (inherited from Animal)
dog1.bark(); // 'Buddy barks: Woof!' (from Dog)

console.log(dog1.breed); // 'Golden Retriever'

// Prototype chain: dog1 → Dog → Animal → Object.prototype → null
```

### Polyfill for Object.create

```javascript
// Understanding Object.create implementation
if (typeof Object.create !== 'function') {
  Object.create = function(proto, propertiesObject) {
    if (typeof proto !== 'object' && typeof proto !== 'function') {
      throw new TypeError('Object prototype may only be an Object or null');
    }
    
    // Create temporary constructor
    function F() {}
    F.prototype = proto;
    
    // Create new object with proto as prototype
    const obj = new F();
    
    // Add properties if provided
    if (propertiesObject !== undefined) {
      Object.defineProperties(obj, propertiesObject);
    }
    
    return obj;
  };
}
```

## Constructor Functions

Constructor functions are the traditional way to create objects with shared prototypes in JavaScript.

### Basic Constructor

```javascript
function Person(name, age) {
  // Instance properties (unique per object)
  this.name = name;
  this.age = age;
}

// Prototype methods (shared across all instances)
Person.prototype.greet = function() {
  console.log(`Hello, I'm ${this.name}`);
};

Person.prototype.haveBirthday = function() {
  this.age++;
  console.log(`Happy birthday! Now ${this.age}`);
};

// Create instances
const alice = new Person('Alice', 30);
const bob = new Person('Bob', 25);

alice.greet(); // 'Hello, I'm Alice'
bob.greet();   // 'Hello, I'm Bob'

// Methods are shared (same function object)
console.log(alice.greet === bob.greet); // true

// Properties are unique
console.log(alice.age); // 30
console.log(bob.age);   // 25
```

### What 'new' Does

```javascript
function Person(name) {
  this.name = name;
}

const alice = new Person('Alice');

// 'new' performs these steps:
// 1. Create new empty object
// 2. Set its [[Prototype]] to Person.prototype
// 3. Execute Person function with 'this' bound to new object
// 4. Return the object (or explicit return value if object)

// Manual implementation:
function manualNew(constructor, ...args) {
  // Step 1: Create new object
  const obj = {};
  
  // Step 2: Set prototype
  Object.setPrototypeOf(obj, constructor.prototype);
  
  // Step 3: Execute constructor
  const result = constructor.apply(obj, args);
  
  // Step 4: Return object or constructor's return value
  return (typeof result === 'object' && result !== null) ? result : obj;
}

const bob = manualNew(Person, 'Bob');
console.log(bob.name); // 'Bob'
console.log(bob instanceof Person); // true
```

### Constructor Property

```javascript
function User(name) {
  this.name = name;
}

const user = new User('Alice');

// .constructor references the constructor function
console.log(user.constructor === User); // true
console.log(User.prototype.constructor === User); // true

// Can create new instances using constructor
const user2 = new user.constructor('Bob');
console.log(user2.name); // 'Bob'

// GOTCHA: Overwriting prototype breaks constructor reference
User.prototype = {
  greet() {
    console.log(`Hello, ${this.name}`);
  }
  // Missing: constructor: User
};

const user3 = new User('Charlie');
console.log(user3.constructor === User); // false! (now Object)

// FIX: Always set constructor when overwriting prototype
User.prototype = {
  constructor: User, // Restore constructor reference
  greet() {
    console.log(`Hello, ${this.name}`);
  }
};
```

### Inheritance with Constructors

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

// Set up prototype chain
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog; // Fix constructor reference

// Add child methods
Dog.prototype.bark = function() {
  console.log(`${this.name} barks: Woof!`);
};

const dog = new Dog('Buddy', 'Golden Retriever');
dog.eat();  // 'Buddy is eating' (inherited)
dog.bark(); // 'Buddy barks: Woof!'

console.log(dog instanceof Dog);    // true
console.log(dog instanceof Animal); // true

// Prototype chain: dog → Dog.prototype → Animal.prototype → Object.prototype → null
```

## Class Syntax Desugaring

ES6 classes are syntactic sugar over prototypes. Understanding the desugaring helps understand what classes really do.

### Basic Class

```javascript
// ES6 Class
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  
  greet() {
    console.log(`Hello, I'm ${this.name}`);
  }
  
  haveBirthday() {
    this.age++;
  }
}

// Equivalent ES5 (desugared)
function Person(name, age) {
  this.name = name;
  this.age = age;
}

Person.prototype.greet = function() {
  console.log(`Hello, I'm ${this.name}`);
};

Person.prototype.haveBirthday = function() {
  this.age++;
};

// Key differences:
// 1. Class methods are non-enumerable by default
Object.defineProperty(Person.prototype, 'greet', {
  enumerable: false // Classes do this automatically
});

// 2. Classes must be called with 'new'
// Person(); // TypeError in class, works in function

// 3. Classes are in strict mode by default
```

### Class Inheritance

```javascript
// ES6 Class Inheritance
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
}

// Equivalent ES5
function Animal(name) {
  this.name = name;
}

Animal.prototype.eat = function() {
  console.log(`${this.name} is eating`);
};

function Dog(name, breed) {
  Animal.call(this, name); // super(name)
  this.breed = breed;
}

// Set up prototype chain
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

Dog.prototype.bark = function() {
  console.log(`${this.name} barks`);
};

// Additional: Classes also set up static method inheritance
Object.setPrototypeOf(Dog, Animal);
```

### Static Methods

```javascript
// ES6 Static Methods
class MathUtils {
  static add(a, b) {
    return a + b;
  }
  
  static multiply(a, b) {
    return a * b;
  }
}

console.log(MathUtils.add(2, 3)); // 5

// Equivalent ES5
function MathUtils() {}

MathUtils.add = function(a, b) {
  return a + b;
};

MathUtils.multiply = function(a, b) {
  return a * b;
};

// Static methods are on constructor, not prototype
console.log(MathUtils.hasOwnProperty('add')); // true
console.log(MathUtils.prototype.hasOwnProperty('add')); // false
```

### Getters and Setters

```javascript
// ES6 Getters/Setters
class Circle {
  constructor(radius) {
    this._radius = radius;
  }
  
  get radius() {
    return this._radius;
  }
  
  set radius(value) {
    if (value > 0) {
      this._radius = value;
    }
  }
  
  get area() {
    return Math.PI * this._radius ** 2;
  }
}

// Equivalent ES5
function Circle(radius) {
  this._radius = radius;
}

Object.defineProperty(Circle.prototype, 'radius', {
  get() {
    return this._radius;
  },
  set(value) {
    if (value > 0) {
      this._radius = value;
    }
  },
  enumerable: true,
  configurable: true
});

Object.defineProperty(Circle.prototype, 'area', {
  get() {
    return Math.PI * this._radius ** 2;
  },
  enumerable: true,
  configurable: true
});

const circle = new Circle(5);
console.log(circle.radius); // 5 (getter)
console.log(circle.area);   // 78.54 (computed getter)
circle.radius = 10;         // setter
console.log(circle.area);   // 314.16
```

### Private Fields (Modern Classes)

```javascript
// ES2022 Private Fields
class BankAccount {
  #balance; // Private field
  
  constructor(initialBalance) {
    this.#balance = initialBalance;
  }
  
  deposit(amount) {
    this.#balance += amount;
  }
  
  getBalance() {
    return this.#balance;
  }
}

const account = new BankAccount(1000);
account.deposit(500);
console.log(account.getBalance()); // 1500
// console.log(account.#balance); // SyntaxError: Private field

// No direct ES5 equivalent (true privacy)
// Closest approximation uses WeakMap
const _balance = new WeakMap();

function BankAccountES5(initialBalance) {
  _balance.set(this, initialBalance);
}

BankAccountES5.prototype.deposit = function(amount) {
  _balance.set(this, _balance.get(this) + amount);
};

BankAccountES5.prototype.getBalance = function() {
  return _balance.get(this);
};
```

## Prototype Methods vs Instance Methods

Understanding the difference between prototype and instance methods is crucial for memory efficiency.

### Memory Implications

```javascript
// BAD: Instance methods (created per object)
function PersonBad(name) {
  this.name = name;
  
  this.greet = function() { // New function per instance!
    console.log(`Hello, ${this.name}`);
  };
}

const p1 = new PersonBad('Alice');
const p2 = new PersonBad('Bob');

console.log(p1.greet === p2.greet); // false (different functions)
// Memory: 2 objects + 2 function objects = inefficient

// GOOD: Prototype methods (shared)
function PersonGood(name) {
  this.name = name;
}

PersonGood.prototype.greet = function() {
  console.log(`Hello, ${this.name}`);
};

const p3 = new PersonGood('Alice');
const p4 = new PersonGood('Bob');

console.log(p3.greet === p4.greet); // true (same function)
// Memory: 2 objects + 1 shared function = efficient
```

### When to Use Instance Methods

```javascript
// Use instance methods when they need closure over instance-specific data
function Counter(start) {
  let count = start; // Private variable
  
  // Instance method (has closure over 'count')
  this.increment = function() {
    return ++count;
  };
  
  this.decrement = function() {
    return --count;
  };
  
  this.getValue = function() {
    return count;
  };
}

const c1 = new Counter(0);
const c2 = new Counter(10);

c1.increment(); // 1
c2.increment(); // 11
// Each counter has independent 'count' via closure
```

### Hybrid Approach

```javascript
class Person {
  constructor(name, secretKey) {
    this.name = name; // Public property
    
    // Private via closure (instance method needed)
    let _secretKey = secretKey;
    
    this.unlock = function(key) {
      return key === _secretKey;
    };
  }
  
  // Public method (on prototype, shared)
  greet() {
    console.log(`Hello, I'm ${this.name}`);
  }
}

const person = new Person('Alice', 'secret123');
person.greet(); // Shared method (efficient)
console.log(person.unlock('secret123')); // true (instance method with closure)
```

## Common Misconceptions

### Misconception 1: "Classes Are a New Type of Object"

**Reality**: Classes are syntactic sugar for constructor functions and prototypes. There's no "class type" in JavaScript.

```javascript
class MyClass {}

console.log(typeof MyClass); // 'function' (not 'class')
console.log(MyClass.prototype); // {} (has prototype like functions)
console.log(MyClass instanceof Function); // true
```

### Misconception 2: "Prototype Chain Is Only for Inheritance"

**Reality**: The prototype chain is used for all property lookups, not just inheritance.

```javascript
const obj = { own: 'property' };

// Every property access traverses the prototype chain
console.log(obj.toString); // Found in Object.prototype
console.log(obj.hasOwnProperty); // Found in Object.prototype
console.log(obj.valueOf); // Found in Object.prototype
```

### Misconception 3: "Setting Prototype After Creation Is Fine"

**Reality**: Changing an object's prototype after creation is very slow and should be avoided.

```javascript
const obj = {};

// SLOW: Changing prototype after creation
Object.setPrototypeOf(obj, somePrototype); // Deoptimizes object

// FAST: Set prototype at creation
const obj2 = Object.create(somePrototype);
```

### Misconception 4: "All Properties Go Through Prototype Chain"

**Reality**: Own properties are accessed directly; only missing properties trigger prototype lookup.

```javascript
const obj = {
  own: 'value'
};

// 'own' is accessed directly (fast)
console.log(obj.own);

// 'toString' triggers prototype chain lookup (slower)
console.log(obj.toString);

// Check if property is own vs inherited
console.log(obj.hasOwnProperty('own')); // true
console.log(obj.hasOwnProperty('toString')); // false
```

## Performance Implications

### Prototype Chain Length

```javascript
// SLOW: Deep prototype chain
const level1 = {};
const level2 = Object.create(level1);
const level3 = Object.create(level2);
const level4 = Object.create(level3);
const level5 = Object.create(level4);

level1.method = function() { return 'found'; };

// Accessing method traverses 5 levels
level5.method(); // Slow prototype lookup

// BETTER: Shallow prototype chain
const proto = { method() { return 'found'; } };
const obj = Object.create(proto);

obj.method(); // Only 1 level lookup (fast)
```

### Shape/Hidden Class Stability

```javascript
// GOOD: Consistent shape
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
}

const p1 = new Point(1, 2);
const p2 = new Point(3, 4);
// Same hidden class, optimizable

// BAD: Inconsistent shapes
const p3 = new Point(5, 6);
p3.z = 7; // Adds property, changes hidden class
// p3 now has different shape, hurts optimization
```

### Property Access Performance

```javascript
const obj = {
  directProperty: 'fast' // Own property
};

Object.setPrototypeOf(obj, {
  inheritedProperty: 'slower' // Inherited property
});

// Own property access: O(1)
obj.directProperty; // Fast

// Inherited property: O(n) where n is chain depth
obj.inheritedProperty; // Slower (traverses chain)

// Cache inherited properties if accessed frequently
class OptimizedClass {
  constructor() {
    // Cache inherited method reference
    this._cachedMethod = this.inheritedMethod;
  }
  
  callMethod() {
    this._cachedMethod(); // Faster than this.inheritedMethod()
  }
}
```

## Interview Questions

### Question 1: Explain the prototype chain

**Answer**: The prototype chain is JavaScript's inheritance mechanism. Every object has an internal `[[Prototype]]` link to another object. When accessing a property, JavaScript first checks the object itself. If not found, it traverses the prototype chain until the property is found or the chain ends at `null`. This allows objects to inherit properties and methods from other objects.

Example: `obj → Object.prototype → null`

### Question 2: What's the difference between __proto__ and prototype?

**Answer**:
- `__proto__`: Accessor property on `Object.prototype` that gets/sets an object's `[[Prototype]]` (internal slot). Available on all objects.
- `.prototype`: Regular property on constructor functions. When you create an object with `new`, the new object's `[[Prototype]]` is set to the constructor's `.prototype` property.

```javascript
function User() {}
const user = new User();

user.__proto__ === User.prototype; // true
User.prototype.constructor === User; // true
```

### Question 3: How does Object.create() differ from new?

**Answer**:
- `Object.create(proto)`: Creates a new object with `proto` as its prototype. Simpler, doesn't execute a constructor.
- `new Constructor()`: Creates an object, sets its prototype to `Constructor.prototype`, executes the constructor, returns the object.

`Object.create()` is more flexible for setting up inheritance without calling constructors, while `new` is standard for constructor-based instantiation.

### Question 4: How do ES6 classes desugar to ES5?

**Answer**: ES6 classes are syntactic sugar over prototypes:

```javascript
// ES6
class Dog extends Animal {
  constructor(name) {
    super(name);
  }
  bark() {}
}

// Desugars to:
function Dog(name) {
  Animal.call(this, name); // super
}
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
Dog.prototype.bark = function() {};
Object.setPrototypeOf(Dog, Animal); // Static inheritance
```

Key differences: class methods are non-enumerable, classes must be called with `new`, and classes are in strict mode.

### Question 5: Why are prototype methods preferred over instance methods?

**Answer**: Prototype methods are shared across all instances, saving memory. Instance methods create a new function for each object.

```javascript
// Instance method: N objects = N functions
function Bad(name) {
  this.greet = function() { console.log(name); };
}

// Prototype method: N objects = 1 shared function
function Good(name) {
  this.name = name;
}
Good.prototype.greet = function() { console.log(this.name); };
```

Exception: Use instance methods when you need closure over private variables.

## Key Takeaways

1. **Prototypal inheritance**: JavaScript objects inherit from other objects via the prototype chain

2. **[[Prototype]] is internal**: Access via `Object.getPrototypeOf()`, not `__proto__`

3. **Constructor.prototype**: Used as the prototype for objects created with `new`

4. **Object.create()**: Clean way to create objects with specific prototypes

5. **Classes are sugar**: ES6 classes desugar to constructor functions and prototypes

6. **Prototype methods save memory**: Shared across instances vs instance methods created per object

7. **Property lookup traverses chain**: Own properties first, then up the prototype chain

8. **Changing prototype is slow**: Set prototype at creation, not after

9. **Chain depth matters**: Shorter chains have faster property lookups

10. **Hidden class stability**: Consistent object shapes enable optimization

## Resources

### Official Documentation
- [MDN: Inheritance and the prototype chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)
- [MDN: Object.create()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create)
- [MDN: Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)

### Articles
- "A Plain English Guide to JavaScript Prototypes" by Sebastian Porto
- "Understanding Prototypes in JavaScript" by Yehuda Katz
- "JavaScript Prototype in Plain Language" by JavaScript Is Sexy

### Books
- "You Don't Know JS: this & Object Prototypes" by Kyle Simpson
- "JavaScript: The Good Parts" by Douglas Crockford (Prototype chapter)
- "Eloquent JavaScript" by Marijn Haverbeke (Object-Oriented Programming chapter)

### Specifications
- [ECMAScript Specification: Ordinary Object Internal Methods](https://tc39.es/ecma262/#sec-ordinary-object-internal-methods-and-internal-slots)

### Videos
- "Object Prototypes" by Fun Fun Function
- "Prototypal Inheritance" by MPJ
- "Understanding JavaScript Prototypes" by Tyler McGinnis
