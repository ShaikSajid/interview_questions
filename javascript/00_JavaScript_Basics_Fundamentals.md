# JavaScript Basics & Fundamentals

## Table of Contents
1. [Data Types](#data-types)
2. [Variables & Scoping](#variables--scoping)
3. [Type Coercion & Conversion](#type-coercion--conversion)
4. [Operators](#operators)
5. [Conditionals & Loops](#conditionals--loops)
6. [Functions](#functions)
7. [Objects & Arrays](#objects--arrays)
8. [Event Emitter Pattern](#event-emitter-pattern)
9. [Prototypes & Inheritance](#prototypes--inheritance)
10. [this Keyword](#this-keyword)
11. [Closures](#closures)
12. [Error Handling](#error-handling)

---

## Data Types

JavaScript has **8 data types**: 7 primitive types and 1 reference type (Object).

### Primitive Data Types

```javascript
// 1. String - Text data
const name = "John";
const greeting = 'Hello';
const template = `Hello ${name}`; // Template literals

// 2. Number - Integers and floating-point numbers
const age = 25;
const price = 19.99;
const negative = -10;
const infinity = Infinity;
const notANumber = NaN;

// 3. BigInt - Large integers (beyond Number.MAX_SAFE_INTEGER)
const bigNumber = 9007199254740991n;
const anotherBig = BigInt("9007199254740991");

// 4. Boolean - true or false
const isActive = true;
const isCompleted = false;

// 5. Undefined - Variable declared but not assigned
let undefinedVar;
console.log(undefinedVar); // undefined

// 6. Null - Intentional absence of value
const emptyValue = null;

// 7. Symbol - Unique identifier (ES6)
const sym1 = Symbol('description');
const sym2 = Symbol('description');
console.log(sym1 === sym2); // false - each Symbol is unique
```

### Reference Type

```javascript
// 8. Object - Collection of key-value pairs
const person = {
  name: "Alice",
  age: 30,
  address: {
    city: "New York",
    country: "USA"
  }
};

// Arrays (special type of Object)
const numbers = [1, 2, 3, 4, 5];
const mixed = [1, "hello", true, null, { key: "value" }];

// Functions (special type of Object)
function greet() {
  return "Hello!";
}

// Date (special type of Object)
const now = new Date();

// RegExp (special type of Object)
const pattern = /[a-z]+/gi;
```

### Type Checking

```javascript
// typeof operator
console.log(typeof "hello");        // "string"
console.log(typeof 42);             // "number"
console.log(typeof true);           // "boolean"
console.log(typeof undefined);      // "undefined"
console.log(typeof null);           // "object" (JavaScript quirk!)
console.log(typeof Symbol());       // "symbol"
console.log(typeof 100n);           // "bigint"
console.log(typeof {});             // "object"
console.log(typeof []);             // "object"
console.log(typeof function() {}); // "function"

// Better type checking for arrays and null
Array.isArray([]);                 // true
Array.isArray({});                 // false

const value = null;
value === null;                    // true

// instanceof for complex types
const date = new Date();
console.log(date instanceof Date); // true
console.log(date instanceof Object); // true
```

---

## Variables & Scoping

### Variable Declarations

```javascript
// var - Function scoped, can be redeclared and updated (avoid using)
var x = 10;
var x = 20; // Redeclaration allowed
x = 30;     // Update allowed

// let - Block scoped, cannot be redeclared, can be updated (ES6)
let y = 10;
// let y = 20; // ❌ Error: Cannot redeclare
y = 20;        // ✅ Update allowed

// const - Block scoped, cannot be redeclared or reassigned (ES6)
const z = 10;
// const z = 20; // ❌ Error: Cannot redeclare
// z = 20;       // ❌ Error: Cannot reassign

// Note: const objects/arrays can have their contents modified
const user = { name: "John" };
user.name = "Jane";  // ✅ Allowed
user.age = 30;       // ✅ Allowed
// user = {};        // ❌ Error: Cannot reassign

const arr = [1, 2, 3];
arr.push(4);         // ✅ Allowed
arr[0] = 10;         // ✅ Allowed
// arr = [];         // ❌ Error: Cannot reassign
```

### Scoping

```javascript
// Global Scope
const globalVar = "I'm global";

function outerFunction() {
  // Function Scope
  var functionVar = "I'm function scoped";
  
  if (true) {
    // Block Scope
    let blockVar = "I'm block scoped";
    const blockConst = "I'm also block scoped";
    var notBlockScoped = "I'm function scoped, not block!";
    
    console.log(blockVar);        // ✅ Accessible
    console.log(functionVar);     // ✅ Accessible
  }
  
  // console.log(blockVar);         // ❌ Error: blockVar not defined
  console.log(notBlockScoped);     // ✅ Accessible (var is function scoped)
  console.log(functionVar);        // ✅ Accessible
}

// Hoisting
console.log(hoistedVar);  // undefined (var is hoisted)
var hoistedVar = "I'm hoisted";

// console.log(notHoisted); // ❌ Error: Cannot access before initialization
let notHoisted = "I'm not hoisted the same way";

// Function hoisting
sayHello(); // ✅ Works! Function declarations are fully hoisted

function sayHello() {
  console.log("Hello!");
}

// sayGoodbye(); // ❌ Error: Cannot access before initialization
const sayGoodbye = function() {
  console.log("Goodbye!");
};
```

### Temporal Dead Zone (TDZ)

```javascript
// The time between entering scope and variable declaration
{
  // TDZ starts
  // console.log(value); // ❌ ReferenceError: Cannot access before initialization
  
  let value = 42; // TDZ ends
  console.log(value); // ✅ 42
}
```

---

## Type Coercion & Conversion

### Implicit Coercion (Automatic)

```javascript
// String coercion
console.log("5" + 3);        // "53" (number to string)
console.log("Hello" + true); // "Hellotrue"

// Number coercion
console.log("5" - 3);        // 2 (string to number)
console.log("5" * "2");      // 10
console.log("10" / "2");     // 5
console.log("10" % "3");     // 1

// Boolean coercion
console.log(true + true);    // 2 (true = 1, false = 0)
console.log(true + false);   // 1

// Comparison coercion
console.log("5" == 5);       // true (loose equality)
console.log("5" === 5);      // false (strict equality)
console.log(null == undefined);  // true
console.log(null === undefined); // false
```

### Explicit Conversion

```javascript
// To String
String(123);                 // "123"
String(true);                // "true"
String(null);                // "null"
String(undefined);           // "undefined"
(123).toString();            // "123"
123 + "";                    // "123"

// To Number
Number("123");               // 123
Number("123.45");            // 123.45
Number("123abc");            // NaN
Number(true);                // 1
Number(false);               // 0
Number(null);                // 0
Number(undefined);           // NaN
parseInt("123");             // 123
parseInt("123.45");          // 123 (removes decimal)
parseFloat("123.45");        // 123.45
+"123";                      // 123 (unary plus)

// To Boolean
Boolean(1);                  // true
Boolean(0);                  // false
Boolean("");                 // false
Boolean("hello");            // true
Boolean(null);               // false
Boolean(undefined);          // false
Boolean({});                 // true
Boolean([]);                 // true
!!value;                     // Double negation converts to boolean
```

### Truthy and Falsy Values

```javascript
// Falsy values (only 8 in JavaScript)
false
0
-0
0n (BigInt zero)
"" (empty string)
null
undefined
NaN

// Everything else is truthy!
true
42
"0"
"false"
[]
{}
function() {}
```

---

## Operators

### Arithmetic Operators

```javascript
let a = 10, b = 3;

console.log(a + b);   // 13 (Addition)
console.log(a - b);   // 7  (Subtraction)
console.log(a * b);   // 30 (Multiplication)
console.log(a / b);   // 3.333... (Division)
console.log(a % b);   // 1  (Modulus/Remainder)
console.log(a ** b);  // 1000 (Exponentiation - ES7)

// Increment/Decrement
let x = 5;
console.log(x++);     // 5 (post-increment, returns then increments)
console.log(x);       // 6
console.log(++x);     // 7 (pre-increment, increments then returns)
console.log(x--);     // 7 (post-decrement)
console.log(--x);     // 5 (pre-decrement)
```

### Comparison Operators

```javascript
console.log(5 == "5");    // true (loose equality)
console.log(5 === "5");   // false (strict equality)
console.log(5 != "5");    // false
console.log(5 !== "5");   // true

console.log(5 > 3);       // true
console.log(5 < 3);       // false
console.log(5 >= 5);      // true
console.log(5 <= 3);      // false
```

### Logical Operators

```javascript
// AND (&&) - Returns first falsy value or last value
console.log(true && true);     // true
console.log(true && false);    // false
console.log("hello" && 42);    // 42
console.log(0 && "hello");     // 0

// OR (||) - Returns first truthy value or last value
console.log(true || false);    // true
console.log(false || false);   // false
console.log(0 || "hello");     // "hello"
console.log("" || 0);          // 0

// NOT (!) - Inverts boolean
console.log(!true);            // false
console.log(!false);           // true
console.log(!"hello");         // false
console.log(!0);               // true

// Nullish Coalescing (??) - Returns right side if left is null/undefined
console.log(null ?? "default");      // "default"
console.log(undefined ?? "default"); // "default"
console.log(0 ?? "default");         // 0 (not null/undefined)
console.log("" ?? "default");        // "" (not null/undefined)
```

### Other Operators

```javascript
// Ternary Operator
const age = 18;
const canVote = age >= 18 ? "Yes" : "No";

// Optional Chaining (?.)
const user = { name: "John", address: { city: "NY" } };
console.log(user?.address?.city);        // "NY"
console.log(user?.contact?.phone);       // undefined (no error)

// Spread Operator (...)
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5];            // [1, 2, 3, 4, 5]

const obj1 = { a: 1, b: 2 };
const obj2 = { ...obj1, c: 3 };          // { a: 1, b: 2, c: 3 }

// Rest Parameters
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b, 0);
}
console.log(sum(1, 2, 3, 4));            // 10
```

---

## Conditionals & Loops

### Conditionals

```javascript
// if...else if...else
const score = 85;

if (score >= 90) {
  console.log("A");
} else if (score >= 80) {
  console.log("B");
} else if (score >= 70) {
  console.log("C");
} else {
  console.log("F");
}

// switch statement
const day = "Monday";

switch (day) {
  case "Monday":
    console.log("Start of the week");
    break;
  case "Friday":
    console.log("End of the week");
    break;
  case "Saturday":
  case "Sunday":
    console.log("Weekend!");
    break;
  default:
    console.log("Midweek");
}
```

### Loops

```javascript
// for loop
for (let i = 0; i < 5; i++) {
  console.log(i); // 0, 1, 2, 3, 4
}

// while loop
let count = 0;
while (count < 5) {
  console.log(count);
  count++;
}

// do...while loop (executes at least once)
let num = 0;
do {
  console.log(num);
  num++;
} while (num < 5);

// for...of (iterates over values)
const fruits = ["apple", "banana", "orange"];
for (const fruit of fruits) {
  console.log(fruit);
}

// for...in (iterates over keys/indices)
const person = { name: "John", age: 30 };
for (const key in person) {
  console.log(`${key}: ${person[key]}`);
}

// Array methods (forEach, map, filter, etc.)
fruits.forEach((fruit, index) => {
  console.log(`${index}: ${fruit}`);
});

// break and continue
for (let i = 0; i < 10; i++) {
  if (i === 3) continue;  // Skip 3
  if (i === 7) break;     // Stop at 7
  console.log(i);         // 0, 1, 2, 4, 5, 6
}
```

---

## Functions

### Function Declaration

```javascript
// Regular function
function add(a, b) {
  return a + b;
}

// Function with default parameters
function greet(name = "Guest") {
  return `Hello, ${name}!`;
}

// Function with rest parameters
function sum(...numbers) {
  return numbers.reduce((total, num) => total + num, 0);
}
```

### Function Expression

```javascript
// Named function expression
const multiply = function mult(a, b) {
  return a * b;
};

// Anonymous function expression
const subtract = function(a, b) {
  return a - b;
};
```

### Arrow Functions

```javascript
// Basic arrow function
const divide = (a, b) => a / b;

// With single parameter (parentheses optional)
const square = x => x * x;

// With multiple statements (need curly braces and return)
const calculate = (a, b) => {
  const sum = a + b;
  const product = a * b;
  return { sum, product };
};

// Implicit return of object (need parentheses)
const createUser = (name, age) => ({ name, age });
```

### Immediately Invoked Function Expression (IIFE)

```javascript
// IIFE - Executes immediately
(function() {
  console.log("I run immediately!");
})();

// IIFE with parameters
(function(name) {
  console.log(`Hello, ${name}!`);
})("John");

// Arrow function IIFE
(() => {
  console.log("Arrow IIFE");
})();
```

### Higher-Order Functions

```javascript
// Function that takes another function as argument
function executeOperation(a, b, operation) {
  return operation(a, b);
}

const result = executeOperation(5, 3, (x, y) => x + y); // 8

// Function that returns a function
function multiplier(factor) {
  return function(number) {
    return number * factor;
  };
}

const double = multiplier(2);
const triple = multiplier(3);

console.log(double(5));  // 10
console.log(triple(5));  // 15
```

### Callback Functions

```javascript
// Synchronous callback
function processArray(arr, callback) {
  const result = [];
  for (const item of arr) {
    result.push(callback(item));
  }
  return result;
}

const numbers = [1, 2, 3, 4, 5];
const squared = processArray(numbers, x => x * x);

// Asynchronous callback
function fetchData(callback) {
  setTimeout(() => {
    callback({ data: "Some data" });
  }, 1000);
}

fetchData((result) => {
  console.log(result);
});
```

---

## Objects & Arrays

### Objects

```javascript
// Object creation
const person = {
  name: "Alice",
  age: 30,
  city: "New York",
  greet: function() {
    return `Hello, I'm ${this.name}`;
  }
};

// Accessing properties
console.log(person.name);           // Dot notation
console.log(person["age"]);         // Bracket notation

// Adding/Modifying properties
person.email = "alice@example.com";
person.age = 31;

// Deleting properties
delete person.city;

// Checking property existence
console.log("name" in person);      // true
console.log(person.hasOwnProperty("age")); // true

// Object methods
const keys = Object.keys(person);           // Array of keys
const values = Object.values(person);       // Array of values
const entries = Object.entries(person);     // Array of [key, value] pairs

// Object destructuring
const { name, age } = person;
const { name: userName, age: userAge } = person; // Rename

// Object spread
const newPerson = { ...person, country: "USA" };

// Object.assign (copy/merge objects)
const merged = Object.assign({}, person, { role: "developer" });

// Object.freeze (make immutable)
const frozen = Object.freeze({ value: 42 });
// frozen.value = 100; // Silently fails (or throws error in strict mode)

// Object.seal (prevent adding/removing properties)
const sealed = Object.seal({ value: 42 });
sealed.value = 100; // ✅ Allowed
// sealed.newProp = "test"; // ❌ Silently fails
```

### Arrays

```javascript
// Array creation
const numbers = [1, 2, 3, 4, 5];
const mixed = [1, "hello", true, null, { key: "value" }];
const arrayFromConstructor = new Array(5); // Creates array with 5 empty slots

// Accessing elements
console.log(numbers[0]);        // 1 (first element)
console.log(numbers[numbers.length - 1]); // 5 (last element)

// Modifying arrays
numbers.push(6);                // Add to end
numbers.pop();                  // Remove from end
numbers.unshift(0);             // Add to beginning
numbers.shift();                // Remove from beginning

// Array methods
const fruits = ["apple", "banana", "orange"];

// map - Transform each element
const uppercased = fruits.map(fruit => fruit.toUpperCase());

// filter - Keep elements that pass test
const longFruits = fruits.filter(fruit => fruit.length > 5);

// reduce - Reduce to single value
const nums = [1, 2, 3, 4, 5];
const sum = nums.reduce((acc, num) => acc + num, 0);

// find - Find first element that matches
const found = fruits.find(fruit => fruit.startsWith("b")); // "banana"

// findIndex - Find index of first match
const index = fruits.findIndex(fruit => fruit === "orange"); // 2

// some - Check if at least one element passes test
const hasLongFruit = fruits.some(fruit => fruit.length > 6); // false

// every - Check if all elements pass test
const allStrings = fruits.every(fruit => typeof fruit === "string"); // true

// includes - Check if array contains value
console.log(fruits.includes("apple")); // true

// indexOf / lastIndexOf
console.log(fruits.indexOf("banana")); // 1

// slice - Create shallow copy of portion
const sliced = fruits.slice(1, 3); // ["banana", "orange"]

// splice - Modify array (add/remove elements)
const removed = fruits.splice(1, 1, "grape"); // Removes "banana", adds "grape"

// concat - Merge arrays
const moreFruits = fruits.concat(["mango", "kiwi"]);

// join - Convert to string
console.log(fruits.join(", ")); // "apple, grape, orange"

// reverse - Reverse array in place
fruits.reverse();

// sort - Sort array in place
numbers.sort((a, b) => a - b); // Ascending
numbers.sort((a, b) => b - a); // Descending

// flat - Flatten nested arrays
const nested = [1, [2, [3, [4]]]];
console.log(nested.flat());      // [1, 2, [3, [4]]]
console.log(nested.flat(2));     // [1, 2, 3, [4]]
console.log(nested.flat(Infinity)); // [1, 2, 3, 4]

// flatMap - Map then flatten
const arr = [1, 2, 3];
const doubled = arr.flatMap(x => [x, x * 2]); // [1, 2, 2, 4, 3, 6]

// Array destructuring
const [first, second, ...rest] = numbers;

// Array spread
const combined = [...fruits, ...moreFruits];
```

---

## Event Emitter Pattern

The Event Emitter pattern is fundamental in Node.js and useful for creating event-driven architectures.

### Basic Event Emitter Implementation

```javascript
// Simple EventEmitter class
class EventEmitter {
  constructor() {
    this.events = {};
  }
  
  // Subscribe to an event
  on(eventName, callback) {
    if (!this.events[eventName]) {
      this.events[eventName] = [];
    }
    this.events[eventName].push(callback);
    return this;
  }
  
  // Subscribe to event (fires only once)
  once(eventName, callback) {
    const wrapper = (...args) => {
      callback(...args);
      this.off(eventName, wrapper);
    };
    this.on(eventName, wrapper);
    return this;
  }
  
  // Emit an event
  emit(eventName, ...args) {
    const callbacks = this.events[eventName];
    if (callbacks) {
      callbacks.forEach(callback => callback(...args));
    }
    return this;
  }
  
  // Unsubscribe from event
  off(eventName, callback) {
    const callbacks = this.events[eventName];
    if (callbacks) {
      this.events[eventName] = callbacks.filter(cb => cb !== callback);
    }
    return this;
  }
  
  // Remove all listeners for event
  removeAllListeners(eventName) {
    if (eventName) {
      delete this.events[eventName];
    } else {
      this.events = {};
    }
    return this;
  }
  
  // Get listener count
  listenerCount(eventName) {
    const callbacks = this.events[eventName];
    return callbacks ? callbacks.length : 0;
  }
}

// Usage Example
const emitter = new EventEmitter();

// Subscribe to events
emitter.on('data', (data) => {
  console.log('Received data:', data);
});

emitter.on('data', (data) => {
  console.log('Another listener:', data);
});

// Emit event
emitter.emit('data', { value: 42 });
// Output:
// Received data: { value: 42 }
// Another listener: { value: 42 }

// Once listener
emitter.once('connect', () => {
  console.log('Connected! (fires only once)');
});

emitter.emit('connect'); // "Connected! (fires only once)"
emitter.emit('connect'); // Nothing happens

// Unsubscribe
const errorHandler = (error) => {
  console.error('Error:', error);
};

emitter.on('error', errorHandler);
emitter.off('error', errorHandler);
```

### Real-World Event Emitter Example

```javascript
// User Authentication System
class AuthSystem extends EventEmitter {
  constructor() {
    super();
    this.users = new Map();
  }
  
  register(username, password) {
    if (this.users.has(username)) {
      this.emit('register:error', { message: 'User already exists' });
      return false;
    }
    
    this.users.set(username, { password, loginAttempts: 0 });
    this.emit('register:success', { username });
    return true;
  }
  
  login(username, password) {
    const user = this.users.get(username);
    
    if (!user) {
      this.emit('login:error', { message: 'User not found' });
      return false;
    }
    
    if (user.loginAttempts >= 3) {
      this.emit('login:locked', { username });
      return false;
    }
    
    if (user.password !== password) {
      user.loginAttempts++;
      this.emit('login:failed', { username, attempts: user.loginAttempts });
      return false;
    }
    
    user.loginAttempts = 0;
    this.emit('login:success', { username });
    return true;
  }
}

// Usage
const auth = new AuthSystem();

// Listen to events
auth.on('register:success', ({ username }) => {
  console.log(`✅ User ${username} registered successfully`);
});

auth.on('login:success', ({ username }) => {
  console.log(`✅ User ${username} logged in`);
});

auth.on('login:failed', ({ username, attempts }) => {
  console.log(`❌ Login failed for ${username}. Attempts: ${attempts}`);
});

auth.on('login:locked', ({ username }) => {
  console.log(`🔒 Account locked for ${username}`);
});

// Test
auth.register('john', 'password123');
auth.login('john', 'wrongpassword');
auth.login('john', 'wrongpassword');
auth.login('john', 'wrongpassword');
auth.login('john', 'password123'); // Locked!
```

### Event Emitter with Node.js

```javascript
// In Node.js
const EventEmitter = require('events');

class DataProcessor extends EventEmitter {
  processData(data) {
    this.emit('start', { timestamp: Date.now() });
    
    try {
      // Process data
      const result = data.map(item => item * 2);
      
      this.emit('progress', { processed: result.length });
      this.emit('complete', { result });
      
    } catch (error) {
      this.emit('error', error);
    }
  }
}

const processor = new DataProcessor();

processor.on('start', ({ timestamp }) => {
  console.log(`Started at ${timestamp}`);
});

processor.on('progress', ({ processed }) => {
  console.log(`Processed ${processed} items`);
});

processor.on('complete', ({ result }) => {
  console.log('Complete:', result);
});

processor.on('error', (error) => {
  console.error('Error:', error);
});

processor.processData([1, 2, 3, 4, 5]);
```

---

## Prototypes & Inheritance

### Prototype Chain

```javascript
// Every object has a prototype
const obj = {};
console.log(obj.__proto__ === Object.prototype); // true

// Constructor function
function Person(name, age) {
  this.name = name;
  this.age = age;
}

// Add method to prototype
Person.prototype.greet = function() {
  return `Hello, I'm ${this.name}`;
};

Person.prototype.birthday = function() {
  this.age++;
};

// Create instances
const john = new Person("John", 30);
const jane = new Person("Jane", 25);

console.log(john.greet());    // "Hello, I'm John"
john.birthday();
console.log(john.age);        // 31

// Prototype chain
console.log(john.__proto__ === Person.prototype);           // true
console.log(Person.prototype.__proto__ === Object.prototype); // true
console.log(Object.prototype.__proto__);                    // null
```

### Prototypal Inheritance

```javascript
// Parent constructor
function Animal(name) {
  this.name = name;
}

Animal.prototype.speak = function() {
  return `${this.name} makes a sound`;
};

// Child constructor
function Dog(name, breed) {
  Animal.call(this, name); // Call parent constructor
  this.breed = breed;
}

// Set up inheritance
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

// Add dog-specific method
Dog.prototype.bark = function() {
  return `${this.name} barks!`;
};

// Override parent method
Dog.prototype.speak = function() {
  return `${this.name} barks loudly!`;
};

const dog = new Dog("Buddy", "Golden Retriever");
console.log(dog.speak());  // "Buddy barks loudly!"
console.log(dog.bark());   // "Buddy barks!"
```

### ES6 Classes (Syntactic Sugar)

```javascript
// ES6 class syntax
class Animal {
  constructor(name) {
    this.name = name;
  }
  
  speak() {
    return `${this.name} makes a sound`;
  }
  
  static info() {
    return "Animals are living creatures";
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name); // Call parent constructor
    this.breed = breed;
  }
  
  speak() {
    return `${this.name} barks!`;
  }
  
  fetch() {
    return `${this.name} fetches the ball`;
  }
}

const dog = new Dog("Max", "Labrador");
console.log(dog.speak());     // "Max barks!"
console.log(dog.fetch());     // "Max fetches the ball"
console.log(Animal.info());   // "Animals are living creatures"
```

---

## this Keyword

The `this` keyword refers to the context in which a function is executed.

```javascript
// Global context
console.log(this); // Window (browser) or global (Node.js)

// Object method
const user = {
  name: "Alice",
  greet: function() {
    console.log(this.name); // "Alice"
  }
};

user.greet();

// Constructor function
function Person(name) {
  this.name = name;
  this.greet = function() {
    console.log(this.name);
  };
}

const person = new Person("Bob");
person.greet(); // "Bob"

// Arrow functions (lexical this)
const obj = {
  name: "Charlie",
  regularFunc: function() {
    console.log(this.name); // "Charlie"
  },
  arrowFunc: () => {
    console.log(this.name); // undefined (inherits from outer scope)
  }
};

// call, apply, bind
function introduce(greeting, punctuation) {
  return `${greeting}, I'm ${this.name}${punctuation}`;
}

const user1 = { name: "David" };

// call - invoke immediately with arguments
console.log(introduce.call(user1, "Hello", "!"));  // "Hello, I'm David!"

// apply - invoke immediately with array of arguments
console.log(introduce.apply(user1, ["Hi", "."]));  // "Hi, I'm David."

// bind - create new function with bound this
const boundFunc = introduce.bind(user1);
console.log(boundFunc("Hey", "...")); // "Hey, I'm David..."

// this in event handlers
const button = document.createElement('button');
button.addEventListener('click', function() {
  console.log(this); // button element
});

button.addEventListener('click', () => {
  console.log(this); // lexical scope (not button)
});
```

---

## Closures

A closure is a function that has access to variables in its outer (enclosing) scope, even after the outer function has returned.

```javascript
// Basic closure
function outer() {
  const message = "Hello";
  
  function inner() {
    console.log(message); // Has access to outer variable
  }
  
  return inner;
}

const greet = outer();
greet(); // "Hello" - closure remembers 'message'

// Practical example: Counter
function createCounter() {
  let count = 0;
  
  return {
    increment: function() {
      count++;
      return count;
    },
    decrement: function() {
      count--;
      return count;
    },
    getCount: function() {
      return count;
    }
  };
}

const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
console.log(counter.getCount());  // 2
console.log(counter.decrement()); // 1

// Private variables with closures
function BankAccount(initialBalance) {
  let balance = initialBalance; // Private variable
  
  return {
    deposit(amount) {
      balance += amount;
      return balance;
    },
    withdraw(amount) {
      if (amount <= balance) {
        balance -= amount;
        return balance;
      }
      throw new Error("Insufficient funds");
    },
    getBalance() {
      return balance;
    }
  };
}

const account = new BankAccount(1000);
console.log(account.deposit(500));   // 1500
console.log(account.withdraw(200));  // 1300
console.log(account.getBalance());   // 1300
// console.log(account.balance);     // undefined (private)

// Function factory with closures
function multiplier(factor) {
  return function(number) {
    return number * factor;
  };
}

const double = multiplier(2);
const triple = multiplier(3);

console.log(double(5));  // 10
console.log(triple(5));  // 15

// Module pattern
const calculator = (function() {
  let result = 0;
  
  return {
    add(x) {
      result += x;
      return this;
    },
    subtract(x) {
      result -= x;
      return this;
    },
    multiply(x) {
      result *= x;
      return this;
    },
    divide(x) {
      result /= x;
      return this;
    },
    getResult() {
      return result;
    },
    clear() {
      result = 0;
      return this;
    }
  };
})();

calculator.add(10).multiply(2).subtract(5);
console.log(calculator.getResult()); // 15
```

---

## Error Handling

### try...catch...finally

```javascript
// Basic error handling
try {
  // Code that might throw an error
  const result = riskyOperation();
  console.log(result);
} catch (error) {
  // Handle the error
  console.error("An error occurred:", error.message);
} finally {
  // Always executes (cleanup code)
  console.log("Cleanup complete");
}

// Throwing custom errors
function divide(a, b) {
  if (b === 0) {
    throw new Error("Division by zero");
  }
  return a / b;
}

try {
  const result = divide(10, 0);
} catch (error) {
  console.error(error.message); // "Division by zero"
}

// Custom error types
class ValidationError extends Error {
  constructor(message) {
    super(message);
    this.name = "ValidationError";
  }
}

class DatabaseError extends Error {
  constructor(message) {
    super(message);
    this.name = "DatabaseError";
  }
}

function validateUser(user) {
  if (!user.email) {
    throw new ValidationError("Email is required");
  }
  if (!user.password || user.password.length < 8) {
    throw new ValidationError("Password must be at least 8 characters");
  }
}

try {
  validateUser({ email: "test@example.com", password: "123" });
} catch (error) {
  if (error instanceof ValidationError) {
    console.error("Validation failed:", error.message);
  } else if (error instanceof DatabaseError) {
    console.error("Database error:", error.message);
  } else {
    console.error("Unknown error:", error);
  }
}

// Async error handling
async function fetchData(url) {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    const data = await response.json();
    return data;
  } catch (error) {
    console.error("Fetch failed:", error);
    throw error; // Re-throw if needed
  }
}

// Promise error handling
fetchData('https://api.example.com/data')
  .then(data => console.log(data))
  .catch(error => console.error(error))
  .finally(() => console.log("Request complete"));
```

---

## Key Takeaways

1. ✅ JavaScript has **8 data types**: 7 primitives + Object
2. ✅ Use `const` by default, `let` when reassignment needed, avoid `var`
3. ✅ Understand **truthy/falsy** values and type coercion
4. ✅ **Arrow functions** don't have their own `this` (lexical binding)
5. ✅ **Event Emitter** pattern enables event-driven programming
6. ✅ **Prototypes** are the foundation of inheritance in JavaScript
7. ✅ **Closures** allow functions to access outer scope variables
8. ✅ The **`this`** keyword depends on how a function is called
9. ✅ Use **try...catch** for error handling
10. ✅ Modern JavaScript offers powerful array/object methods

---

## Interview Questions

### Q1: What's the difference between `==` and `===`?
**A**: `==` (loose equality) performs type coercion before comparison, while `===` (strict equality) compares both value and type without coercion.

```javascript
5 == "5"   // true (coercion)
5 === "5"  // false (different types)
```

### Q2: Explain closures with an example.
**A**: A closure is when an inner function has access to outer function's variables even after the outer function returns.

```javascript
function outer(x) {
  return function(y) {
    return x + y; // Has access to 'x'
  };
}
const addFive = outer(5);
console.log(addFive(3)); // 8
```

### Q3: What is the Event Loop?
**A**: The Event Loop manages asynchronous operations in JavaScript. It continuously checks the call stack and callback queue, moving callbacks to the stack when it's empty.

### Q4: Explain `this` in different contexts.
**A**: `this` refers to:
- Global scope: global object
- Object method: the object
- Constructor: new instance
- Arrow function: lexical `this` (outer scope)
- Event handler: the element (regular function)

### Q5: What's the difference between `null` and `undefined`?
**A**: 
- `undefined`: Variable declared but not assigned
- `null`: Intentional absence of value (explicitly set)

```javascript
let x;              // undefined
let y = null;       // null
```

This covers the fundamental concepts of JavaScript! Practice these concepts with real examples to solidify your understanding.
