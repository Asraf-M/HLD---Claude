# 30 Most Asked JavaScript Interview Questions

---

## Table of Contents

1. [var vs let vs const](#1-var-vs-let-vs-const)
2. [Hoisting](#2-hoisting)
3. [Closures](#3-closures)
4. [this keyword](#4-this-keyword)
5. [call, apply, bind](#5-call-apply-bind)
6. [Event Loop](#6-event-loop)
7. [Promises](#7-promises)
8. [async/await](#8-asyncawait)
9. [== vs ===](#9--vs-)
10. [Shallow Copy vs Deep Copy](#10-shallow-copy-vs-deep-copy)
11. [Prototypal Inheritance](#11-prototypal-inheritance)
12. [Event Bubbling & Capturing](#12-event-bubbling--capturing)
13. [Debouncing & Throttling](#13-debouncing--throttling)
14. [Currying](#14-currying)
15. [Spread & Rest Operator](#15-spread--rest-operator)
16. [Destructuring](#16-destructuring)
17. [Map, Filter, Reduce](#17-map-filter-reduce)
18. [null vs undefined](#18-null-vs-undefined)
19. [typeof & instanceof](#19-typeof--instanceof)
20. [Arrow Functions vs Regular Functions](#20-arrow-functions-vs-regular-functions)
21. [setTimeout & setInterval](#21-settimeout--setinterval)
22. [Callback Hell & How to Avoid It](#22-callback-hell--how-to-avoid-it)
23. [Memoization](#23-memoization)
24. [Generator Functions](#24-generator-functions)
25. [WeakMap & WeakSet](#25-weakmap--weakset)
26. [Symbol](#26-symbol)
27. [Proxy & Reflect](#27-proxy--reflect)
28. [Optional Chaining & Nullish Coalescing](#28-optional-chaining--nullish-coalescing)
29. [Promise.all, Promise.race, Promise.allSettled, Promise.any](#29-promiseall-promiserace-promiseallsettled-promiseany)
30. [Object.freeze vs Object.seal](#30-objectfreeze-vs-objectseal)

---

## 1. var vs let vs const

**Q: What are the differences between `var`, `let`, and `const`?**

| Feature | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function-scoped | Block-scoped | Block-scoped |
| Hoisting | ✅ Hoisted (initialized as `undefined`) | ✅ Hoisted (NOT initialized — TDZ) | ✅ Hoisted (NOT initialized — TDZ) |
| Re-declaration | ✅ Allowed | ❌ Not allowed | ❌ Not allowed |
| Re-assignment | ✅ Allowed | ✅ Allowed | ❌ Not allowed |

```js
// Scope difference
function test() {
    if (true) {
        var a = 10;    // function-scoped — accessible outside if block
        let b = 20;    // block-scoped — only inside if block
        const c = 30;  // block-scoped — only inside if block
    }
    console.log(a);  // 10
    console.log(b);  // ReferenceError: b is not defined
}

// TDZ (Temporal Dead Zone)
console.log(x);  // undefined (var is hoisted with value undefined)
var x = 5;

console.log(y);  // ReferenceError (let is in TDZ)
let y = 5;

// const with objects — reference is constant, content is NOT
const obj = { name: "John" };
obj.name = "Jane";   // ✅ allowed — modifying property
obj = {};            // ❌ TypeError — reassigning reference
```

---

## 2. Hoisting

**Q: What is hoisting in JavaScript?**

Hoisting is JavaScript's behavior of moving **declarations** (not initializations) to the top of their scope during the compilation phase.

```js
// What you write:
console.log(greet());  // "Hello!"
console.log(num);      // undefined

function greet() { return "Hello!"; }
var num = 42;

// How JS engine sees it (after hoisting):
function greet() { return "Hello!"; }   // function fully hoisted
var num;                                 // only declaration hoisted
console.log(greet());                    // "Hello!"
console.log(num);                        // undefined
num = 42;
```

### Hoisting Rules

| Declaration | Hoisted? | Value before initialization |
|---|---|---|
| `function declaration` | ✅ Fully hoisted (body included) | Callable |
| `var` | ✅ Declaration hoisted | `undefined` |
| `let` / `const` | ✅ Hoisted but NOT initialized | ❌ ReferenceError (TDZ) |
| `function expression` | Only the `var` part is hoisted | `undefined` |
| `class` | Hoisted but NOT initialized | ❌ ReferenceError (TDZ) |

```js
// Function expression — NOT fully hoisted
console.log(sayHi);    // undefined (var hoisted, not the function)
console.log(sayHi());  // TypeError: sayHi is not a function
var sayHi = function() { return "Hi"; };
```

---

## 3. Closures

**Q: What is a closure?**

A **closure** is a function that **remembers** the variables from its outer (enclosing) scope even after the outer function has returned.

```js
function outer() {
    let count = 0;                    // outer variable
    return function inner() {         // closure — remembers count
        count++;
        console.log(count);
    };
}

const counter = outer();  // outer() returns inner, outer is done
counter();  // 1  ← still has access to count!
counter();  // 2
counter();  // 3
```

### Practical Use Cases

```js
// 1. Data Privacy (encapsulation)
function createWallet(initialBalance) {
    let balance = initialBalance;  // private variable
    return {
        deposit(amount)  { balance += amount; },
        withdraw(amount) { balance -= amount; },
        getBalance()     { return balance; }
    };
}
const wallet = createWallet(100);
wallet.deposit(50);
wallet.getBalance();    // 150
// wallet.balance       // undefined — can't access directly

// 2. Classic interview trap: var in loop
for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 1000);
}
// Output: 3, 3, 3  ← all share the same i (var is function-scoped)

// Fix with closure (IIFE):
for (var i = 0; i < 3; i++) {
    (function(j) {
        setTimeout(() => console.log(j), 1000);
    })(i);
}
// Output: 0, 1, 2

// Fix with let (block-scoped):
for (let i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 1000);
}
// Output: 0, 1, 2
```

---

## 4. this keyword

**Q: How does `this` work in JavaScript?**

`this` refers to the **object that is calling the function**. Its value depends on **how** the function is called, not where it's defined.

| Context | `this` refers to |
|---|---|
| Global (non-strict) | `window` (browser) / `global` (Node) |
| Global (strict) | `undefined` |
| Object method | The object calling the method |
| `new` keyword | The newly created object |
| Arrow function | Inherits `this` from enclosing scope (lexical) |
| `call/apply/bind` | Explicitly set |

```js
// 1. Object method
const user = {
    name: "Alice",
    greet() { console.log(this.name); }
};
user.greet();  // "Alice" — this = user

// 2. Lost context
const fn = user.greet;
fn();  // undefined — this = window (non-strict) / undefined (strict)

// 3. Arrow function — lexical this
const user2 = {
    name: "Bob",
    greet: () => console.log(this.name)
};
user2.greet();  // undefined — arrow inherits outer this (window)

// 4. new keyword
function Person(name) {
    this.name = name;  // this = new object
}
const p = new Person("Eve");
console.log(p.name);  // "Eve"
```

---

## 5. call, apply, bind

**Q: What are `call()`, `apply()`, and `bind()`?**

All three explicitly set the value of `this`.

| Method | Executes immediately? | Arguments |
|---|---|---|
| `call()` | ✅ Yes | Comma-separated |
| `apply()` | ✅ Yes | Array |
| `bind()` | ❌ No — returns new function | Comma-separated |

```js
const person = { name: "Alice" };

function greet(city, country) {
    console.log(`${this.name} from ${city}, ${country}`);
}

// call — immediate, comma args
greet.call(person, "Delhi", "India");     // "Alice from Delhi, India"

// apply — immediate, array args
greet.apply(person, ["Delhi", "India"]);  // "Alice from Delhi, India"

// bind — returns new function, call later
const boundGreet = greet.bind(person, "Delhi", "India");
boundGreet();  // "Alice from Delhi, India"
```

### Polyfill for `bind` (Common Interview Question)

```js
Function.prototype.myBind = function(context, ...args) {
    const fn = this;
    return function(...newArgs) {
        return fn.apply(context, [...args, ...newArgs]);
    };
};

const bound = greet.myBind(person, "Delhi");
bound("India");  // "Alice from Delhi, India"
```

---

## 6. Event Loop

**Q: How does the JavaScript Event Loop work?**

JavaScript is **single-threaded** but achieves async behavior through the **Event Loop**.

```
  ┌──────────────────────────────────────────┐
  │          Call Stack (LIFO)                │
  │   Executes synchronous code one by one   │
  └───────────────┬──────────────────────────┘
                  │ when async completes
  ┌───────────────▼──────────────────────────┐
  │       Callback Queues                    │
  │  ┌──────────────────────────────┐        │
  │  │ Microtask Queue (Priority)  │        │
  │  │ Promise.then, queueMicrotask│        │
  │  │ MutationObserver            │        │
  │  └──────────────────────────────┘        │
  │  ┌──────────────────────────────┐        │
  │  │ Macrotask Queue             │        │
  │  │ setTimeout, setInterval     │        │
  │  │ I/O, UI rendering           │        │
  │  └──────────────────────────────┘        │
  └──────────────────────────────────────────┘

  Event Loop: if call stack is empty →
      1. Drain ALL microtasks first
      2. Pick ONE macrotask
      3. Repeat
```

```js
console.log("1");                          // sync → call stack

setTimeout(() => console.log("2"), 0);     // macrotask queue

Promise.resolve().then(() => console.log("3"));  // microtask queue

console.log("4");                          // sync → call stack

// Output: 1, 4, 3, 2
// Explanation:
//   1 → sync (immediate)
//   4 → sync (immediate)
//   3 → microtask (priority over macrotask)
//   2 → macrotask (after all microtasks)
```

---

## 7. Promises

**Q: What is a Promise?**

A **Promise** is an object representing the eventual completion or failure of an async operation.

### Three States

```
  ┌─────────┐    resolve(value)    ┌───────────┐
  │ PENDING │ ───────────────────▶ │ FULFILLED │ → .then(value)
  └────┬────┘                      └───────────┘
       │
       │      reject(error)        ┌──────────┐
       └──────────────────────────▶│ REJECTED │ → .catch(error)
                                   └──────────┘
```

```js
// Creating a Promise
const fetchData = new Promise((resolve, reject) => {
    setTimeout(() => {
        const success = true;
        if (success) resolve("Data loaded");
        else reject("Error occurred");
    }, 1000);
});

// Consuming
fetchData
    .then(data => console.log(data))      // "Data loaded"
    .catch(err => console.error(err))
    .finally(() => console.log("Done"));  // always runs

// Chaining
fetch("/api/user")
    .then(res => res.json())
    .then(user => fetch(`/api/posts/${user.id}`))
    .then(res => res.json())
    .then(posts => console.log(posts))
    .catch(err => console.error(err));
```

---

## 8. async/await

**Q: What is async/await and how does it relate to Promises?**

`async/await` is **syntactic sugar** over Promises — makes async code look synchronous.

```js
// With Promises
function getUser() {
    return fetch("/api/user")
        .then(res => res.json())
        .then(user => fetch(`/api/posts/${user.id}`))
        .then(res => res.json());
}

// With async/await — same logic, cleaner syntax
async function getUser() {
    const res = await fetch("/api/user");
    const user = await res.json();
    const postRes = await fetch(`/api/posts/${user.id}`);
    return await postRes.json();
}

// Error handling
async function getData() {
    try {
        const res = await fetch("/api/data");
        const data = await res.json();
        return data;
    } catch (err) {
        console.error("Failed:", err);
    }
}
```

| Feature | Detail |
|---|---|
| `async` function | Always returns a Promise |
| `await` | Pauses execution until Promise resolves |
| Error handling | Use `try/catch` instead of `.catch()` |
| Top-level await | Supported in ES modules |

---

## 9. == vs ===

**Q: What is the difference between `==` and `===`?**

| Operator | Name | Type Coercion |
|---|---|---|
| `==` | Loose equality | ✅ Converts types before comparing |
| `===` | Strict equality | ❌ No conversion — type must also match |

```js
// == (type coercion)
0 == ""           // true  ← both coerced to 0
0 == "0"          // true  ← "0" coerced to 0
"" == "0"         // false ← string comparison
false == "0"      // true  ← both coerced to 0
null == undefined // true  ← special case
NaN == NaN        // false ← NaN is never equal to anything

// === (strict — no coercion)
0 === ""          // false ← number vs string
0 === "0"         // false
false === "0"     // false
null === undefined// false ← different types
```

> **Rule:** Always use `===` unless you intentionally need type coercion.

---

## 10. Shallow Copy vs Deep Copy

**Q: What is the difference between shallow copy and deep copy?**

| Copy Type | Nested Objects | Modification Effect |
|---|---|---|
| **Shallow** | Copies references only | Changes in nested objects affect original |
| **Deep** | Copies everything recursively | Fully independent |

```js
const original = { name: "Alice", address: { city: "Delhi" } };

// SHALLOW COPY — nested objects still shared
const shallow = { ...original };
shallow.name = "Bob";             // ✅ doesn't affect original
shallow.address.city = "Mumbai";  // ❌ AFFECTS original!
console.log(original.address.city); // "Mumbai"

// DEEP COPY methods
// 1. structuredClone (modern, best)
const deep1 = structuredClone(original);

// 2. JSON trick (loses functions, Date, undefined, etc.)
const deep2 = JSON.parse(JSON.stringify(original));

// 3. Recursive function
function deepClone(obj) {
    if (obj === null || typeof obj !== "object") return obj;
    const copy = Array.isArray(obj) ? [] : {};
    for (let key in obj) {
        if (obj.hasOwnProperty(key)) {
            copy[key] = deepClone(obj[key]);
        }
    }
    return copy;
}
```

---

## 11. Prototypal Inheritance

**Q: How does prototypal inheritance work in JavaScript?**

Every object has an internal link (`__proto__`) to another object called its **prototype**. When a property is not found on the object, JS looks up the prototype chain.

```
  myObj → myObj.__proto__ (Parent.prototype) → Object.prototype → null
```

```js
// Constructor + prototype
function Animal(name) {
    this.name = name;
}
Animal.prototype.speak = function() {
    return `${this.name} makes a sound`;
};

function Dog(name, breed) {
    Animal.call(this, name);  // call parent constructor
    this.breed = breed;
}
Dog.prototype = Object.create(Animal.prototype);  // inherit
Dog.prototype.constructor = Dog;
Dog.prototype.bark = function() { return "Woof!"; };

const dog = new Dog("Rex", "Labrador");
dog.speak();  // "Rex makes a sound" — found on Animal.prototype
dog.bark();   // "Woof!" — found on Dog.prototype

// ES6 class syntax (same prototype chain under the hood)
class Animal {
    constructor(name) { this.name = name; }
    speak() { return `${this.name} makes a sound`; }
}
class Dog extends Animal {
    constructor(name, breed) {
        super(name);
        this.breed = breed;
    }
    bark() { return "Woof!"; }
}
```

---

## 12. Event Bubbling & Capturing

**Q: What is event bubbling and event capturing?**

```
  Event Capturing (top → down)        Event Bubbling (bottom → up)
  ┌─────────────────────────┐         ┌─────────────────────────┐
  │ document                │    1    │ document                │    3
  │  ┌────────────────────┐ │   ↓    │  ┌────────────────────┐ │   ↑
  │  │ parent             │ │   ↓    │  │ parent             │ │   ↑
  │  │  ┌───────────────┐ │ │   ↓    │  │  ┌───────────────┐ │ │   ↑
  │  │  │ child (click) │ │ │   ↓    │  │  │ child (click) │ │ │   ↑
  │  │  └───────────────┘ │ │        │  │  └───────────────┘ │ │
  │  └────────────────────┘ │        │  └────────────────────┘ │
  └─────────────────────────┘        └─────────────────────────┘
         Phase 1                              Phase 3
```

```js
const parent = document.getElementById("parent");
const child = document.getElementById("child");

// Bubbling (default) — inner → outer
parent.addEventListener("click", () => console.log("Parent"));
child.addEventListener("click", () => console.log("Child"));
// Click child → "Child", "Parent"

// Capturing — outer → inner (3rd param = true)
parent.addEventListener("click", () => console.log("Parent"), true);
child.addEventListener("click", () => console.log("Child"), true);
// Click child → "Parent", "Child"

// Stop propagation
child.addEventListener("click", (e) => {
    e.stopPropagation();  // prevents bubbling to parent
    console.log("Child only");
});
```

### Event Delegation

```js
// Instead of adding listener to every li, add to parent ul
document.getElementById("list").addEventListener("click", (e) => {
    if (e.target.tagName === "LI") {
        console.log("Clicked:", e.target.textContent);
    }
});
```

---

## 13. Debouncing & Throttling

**Q: What is debouncing and throttling?**

| Technique | Fires when | Use case |
|---|---|---|
| **Debounce** | After user **stops** for X ms | Search input, form validation |
| **Throttle** | At most once every X ms | Scroll, resize, mouse move |

```js
// DEBOUNCE — waits until user stops typing
function debounce(fn, delay) {
    let timer;
    return function(...args) {
        clearTimeout(timer);
        timer = setTimeout(() => fn.apply(this, args), delay);
    };
}

const search = debounce((query) => {
    console.log("Searching:", query);
}, 300);
// User types: H → He → Hel → Hello
// Only ONE call after 300ms pause: "Searching: Hello"

// THROTTLE — executes at most once per interval
function throttle(fn, limit) {
    let inThrottle = false;
    return function(...args) {
        if (!inThrottle) {
            fn.apply(this, args);
            inThrottle = true;
            setTimeout(() => inThrottle = false, limit);
        }
    };
}

window.addEventListener("scroll", throttle(() => {
    console.log("Scroll event");
}, 200));
// Fires at most once every 200ms during scroll
```

---

## 14. Currying

**Q: What is currying?**

Currying transforms a function with **multiple arguments** into a sequence of functions each taking **one argument**.

```js
// Normal function
function add(a, b, c) { return a + b + c; }
add(1, 2, 3);  // 6

// Curried version
function curriedAdd(a) {
    return function(b) {
        return function(c) {
            return a + b + c;
        };
    };
}
curriedAdd(1)(2)(3);  // 6

// Arrow function shorthand
const curriedAdd = a => b => c => a + b + c;

// Practical: reusable config
const multiply = a => b => a * b;
const double = multiply(2);
const triple = multiply(3);
double(5);  // 10
triple(5);  // 15

// Generic curry utility
function curry(fn) {
    return function curried(...args) {
        if (args.length >= fn.length) {
            return fn.apply(this, args);
        }
        return (...nextArgs) => curried(...args, ...nextArgs);
    };
}

const sum = curry((a, b, c) => a + b + c);
sum(1)(2)(3);    // 6
sum(1, 2)(3);    // 6
sum(1)(2, 3);    // 6
```

---

## 15. Spread & Rest Operator

**Q: What is the difference between spread and rest operators?**

Both use `...` but serve **opposite** purposes.

| Operator | Purpose | Where |
|---|---|---|
| **Spread** | Expands elements | Function calls, arrays, objects |
| **Rest** | Collects remaining elements | Function parameters, destructuring |

```js
// SPREAD — expands
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5];       // [1, 2, 3, 4, 5]

const obj1 = { a: 1, b: 2 };
const obj2 = { ...obj1, c: 3 };     // { a: 1, b: 2, c: 3 }

Math.max(...arr1);                    // 3

// REST — collects
function sum(...nums) {              // nums = [1, 2, 3, 4]
    return nums.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3, 4);  // 10

const [first, ...rest] = [10, 20, 30, 40];
// first = 10, rest = [20, 30, 40]

const { name, ...others } = { name: "Alice", age: 25, city: "Delhi" };
// name = "Alice", others = { age: 25, city: "Delhi" }
```

---

## 16. Destructuring

**Q: What is destructuring in JavaScript?**

Extracts values from arrays or properties from objects into distinct variables.

```js
// ARRAY destructuring
const [a, b, c] = [1, 2, 3];      // a=1, b=2, c=3
const [x, , z] = [1, 2, 3];       // x=1, z=3 (skip 2nd)
const [p = 10, q = 20] = [5];     // p=5, q=20 (default)

// Swap variables
let m = 1, n = 2;
[m, n] = [n, m];                   // m=2, n=1

// OBJECT destructuring
const user = { name: "Alice", age: 25, city: "Delhi" };
const { name, age } = user;        // name="Alice", age=25

// Rename
const { name: userName } = user;   // userName = "Alice"

// Default values
const { country = "India" } = user; // country = "India"

// Nested
const data = { user: { name: "Bob", address: { city: "Mumbai" } } };
const { user: { address: { city } } } = data;  // city = "Mumbai"

// Function parameters
function display({ name, age }) {
    console.log(`${name} is ${age}`);
}
display(user);  // "Alice is 25"
```

---

## 17. Map, Filter, Reduce

**Q: Explain `map()`, `filter()`, and `reduce()` with examples.**

| Method | Purpose | Returns |
|---|---|---|
| `map()` | Transform each element | New array (same length) |
| `filter()` | Keep elements that pass a test | New array (≤ length) |
| `reduce()` | Reduce array to a single value | Single value |

```js
const nums = [1, 2, 3, 4, 5];

// MAP — transform each element
const doubled = nums.map(n => n * 2);        // [2, 4, 6, 8, 10]

// FILTER — keep elements matching condition
const evens = nums.filter(n => n % 2 === 0); // [2, 4]

// REDUCE — accumulate into single value
const sum = nums.reduce((acc, n) => acc + n, 0);  // 15

// Chaining
const result = nums
    .filter(n => n % 2 !== 0)   // [1, 3, 5]
    .map(n => n * 10)           // [10, 30, 50]
    .reduce((a, b) => a + b, 0); // 90

// REDUCE — advanced: group by
const people = [
    { name: "Alice", dept: "HR" },
    { name: "Bob", dept: "Eng" },
    { name: "Carol", dept: "HR" }
];
const grouped = people.reduce((acc, person) => {
    acc[person.dept] = acc[person.dept] || [];
    acc[person.dept].push(person.name);
    return acc;
}, {});
// { HR: ["Alice", "Carol"], Eng: ["Bob"] }
```

---

## 18. null vs undefined

**Q: What is the difference between `null` and `undefined`?**

| Feature | `undefined` | `null` |
|---|---|---|
| Meaning | Variable declared but not assigned | Intentional absence of value |
| Type | `"undefined"` | `"object"` (historical bug) |
| Set by | JS engine automatically | Developer explicitly |
| `==` comparison | `null == undefined` → `true` | `null == undefined` → `true` |
| `===` comparison | `null === undefined` → `false` | Different types |

```js
let a;             // undefined — declared, not assigned
let b = null;      // null — intentionally empty

typeof undefined;  // "undefined"
typeof null;       // "object" (JS bug, kept for compatibility)

// When undefined occurs automatically:
function foo(x) { return x; }
foo();                    // undefined — no argument passed

const obj = {};
obj.name;                 // undefined — property doesn't exist

function bar() {}
bar();                    // undefined — no return statement

// Checking
if (value == null)  { }   // catches both null AND undefined
if (value === null) { }   // only null
if (value === undefined) { } // only undefined
```

---

## 19. typeof & instanceof

**Q: What is the difference between `typeof` and `instanceof`?**

| Operator | Purpose | Returns |
|---|---|---|
| `typeof` | Check **primitive type** | String |
| `instanceof` | Check if object is **instance of a class/constructor** | Boolean |

```js
// typeof
typeof 42;           // "number"
typeof "hello";      // "string"
typeof true;         // "boolean"
typeof undefined;    // "undefined"
typeof null;         // "object"     ← JS bug
typeof {};           // "object"
typeof [];           // "object"     ← arrays are objects
typeof function(){}; // "function"
typeof Symbol();     // "symbol"

// instanceof — checks prototype chain
const arr = [1, 2, 3];
arr instanceof Array;   // true
arr instanceof Object;  // true (Array inherits from Object)

class Animal {}
class Dog extends Animal {}
const d = new Dog();
d instanceof Dog;       // true
d instanceof Animal;    // true (prototype chain)

// Reliable type checking
Array.isArray([1, 2]);                    // true
Object.prototype.toString.call([1, 2]);   // "[object Array]"
Object.prototype.toString.call(null);     // "[object Null]"
```

---

## 20. Arrow Functions vs Regular Functions

**Q: What are the differences between arrow functions and regular functions?**

| Feature | Regular Function | Arrow Function |
|---|---|---|
| `this` | Dynamic (depends on caller) | Lexical (inherits from parent scope) |
| `arguments` object | ✅ Available | ❌ Not available |
| `new` keyword | ✅ Can be constructor | ❌ Cannot use `new` |
| `prototype` | ✅ Has `.prototype` | ❌ No `.prototype` |
| Hoisting | ✅ (function declaration) | ❌ (treated as expression) |

```js
// this difference
const obj = {
    name: "Alice",
    regular: function() { console.log(this.name); },
    arrow: () => console.log(this.name)
};
obj.regular();  // "Alice" — this = obj
obj.arrow();    // undefined — this = outer scope (window)

// arguments object
function regular() { console.log(arguments); }
regular(1, 2, 3);  // [1, 2, 3]

const arrow = () => console.log(arguments);
arrow(1, 2, 3);    // ReferenceError: arguments is not defined
// Use rest params instead: const arrow = (...args) => console.log(args);

// Cannot use as constructor
const Foo = () => {};
new Foo();  // TypeError: Foo is not a constructor
```

---

## 21. setTimeout & setInterval

**Q: Explain `setTimeout` and `setInterval`.**

| Function | Executes | Times |
|---|---|---|
| `setTimeout(fn, ms)` | After delay | **Once** |
| `setInterval(fn, ms)` | Every interval | **Repeatedly** |

```js
// setTimeout — execute once after delay
const timerId = setTimeout(() => {
    console.log("Runs after 2 seconds");
}, 2000);
clearTimeout(timerId);  // cancel before it runs

// setInterval — execute repeatedly
let count = 0;
const intervalId = setInterval(() => {
    count++;
    console.log("Tick:", count);
    if (count === 5) clearInterval(intervalId);  // stop after 5
}, 1000);

// setTimeout with 0 delay — still async!
console.log("A");
setTimeout(() => console.log("B"), 0);
console.log("C");
// Output: A, C, B  (B goes to macrotask queue)

// Recursive setTimeout (better than setInterval for varying delays)
function poll() {
    fetch("/api/status")
        .then(res => res.json())
        .then(data => {
            console.log(data);
            setTimeout(poll, 3000);  // schedule next after completion
        });
}
```

---

## 22. Callback Hell & How to Avoid It

**Q: What is callback hell and how do you fix it?**

**Callback hell** is deeply nested callbacks that make code unreadable (the "pyramid of doom").

```js
// ❌ Callback Hell
getUser(userId, function(user) {
    getOrders(user.id, function(orders) {
        getOrderDetails(orders[0].id, function(details) {
            getShipping(details.shippingId, function(shipping) {
                console.log(shipping);
                // more nesting...
            });
        });
    });
});

// ✅ Fix 1: Promises
getUser(userId)
    .then(user => getOrders(user.id))
    .then(orders => getOrderDetails(orders[0].id))
    .then(details => getShipping(details.shippingId))
    .then(shipping => console.log(shipping))
    .catch(err => console.error(err));

// ✅ Fix 2: async/await
async function fetchShipping(userId) {
    const user = await getUser(userId);
    const orders = await getOrders(user.id);
    const details = await getOrderDetails(orders[0].id);
    const shipping = await getShipping(details.shippingId);
    return shipping;
}
```

---

## 23. Memoization

**Q: What is memoization?**

An optimization technique that **caches** results of expensive function calls and returns the cached result for the same inputs.

```js
// Generic memoize function
function memoize(fn) {
    const cache = {};
    return function(...args) {
        const key = JSON.stringify(args);
        if (cache[key] !== undefined) {
            console.log("From cache");
            return cache[key];
        }
        const result = fn.apply(this, args);
        cache[key] = result;
        return result;
    };
}

// Example: expensive calculation
const factorial = memoize(function(n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
});

factorial(5);  // computes: 120
factorial(5);  // from cache: 120
factorial(4);  // from cache: 24 (already computed as part of 5!)

// Fibonacci with memoization
const fib = memoize(function(n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
});
fib(40);  // instant (without memo: ~1 billion recursive calls)
```

---

## 24. Generator Functions

**Q: What are generator functions?**

Functions that can **pause** and **resume** execution. They return an **iterator** object.

```js
function* numberGenerator() {
    yield 1;       // pause, return 1
    yield 2;       // pause, return 2
    yield 3;       // pause, return 3
}

const gen = numberGenerator();
gen.next();  // { value: 1, done: false }
gen.next();  // { value: 2, done: false }
gen.next();  // { value: 3, done: false }
gen.next();  // { value: undefined, done: true }

// Practical: Infinite sequence
function* idGenerator() {
    let id = 1;
    while (true) {
        yield id++;
    }
}
const ids = idGenerator();
ids.next().value;  // 1
ids.next().value;  // 2
ids.next().value;  // 3 (generates forever, but one at a time)

// Iterable with for...of
function* range(start, end) {
    for (let i = start; i <= end; i++) yield i;
}
for (const n of range(1, 5)) {
    console.log(n);  // 1, 2, 3, 4, 5
}

// Two-way communication
function* chat() {
    const name = yield "What is your name?";
    const age = yield `Hello ${name}! How old are you?`;
    return `${name} is ${age} years old`;
}
const c = chat();
c.next();           // { value: "What is your name?" }
c.next("Alice");    // { value: "Hello Alice! How old are you?" }
c.next(25);         // { value: "Alice is 25 years old", done: true }
```

---

## 25. WeakMap & WeakSet

**Q: What are `WeakMap` and `WeakSet`?**

| Feature | Map / Set | WeakMap / WeakSet |
|---|---|---|
| Key type | Any value | **Objects only** |
| Garbage collection | ❌ Prevents GC (strong ref) | ✅ Allows GC (weak ref) |
| Iterable | ✅ `for...of`, `.size` | ❌ Not iterable, no `.size` |
| Use case | General-purpose | Private data, caching, DOM metadata |

```js
// WeakMap — keys must be objects, garbage-collectable
const cache = new WeakMap();

function process(obj) {
    if (cache.has(obj)) return cache.get(obj);
    const result = /* expensive computation */ obj.value * 2;
    cache.set(obj, result);
    return result;
}

let data = { value: 42 };
process(data);          // computes and caches
process(data);          // returns from cache
data = null;            // object is GC'd → WeakMap entry auto-removed

// WeakSet — track objects without preventing GC
const visited = new WeakSet();

function track(obj) {
    if (visited.has(obj)) {
        console.log("Already visited");
        return;
    }
    visited.add(obj);
    console.log("First visit");
}

// Practical: private data
const privateData = new WeakMap();
class User {
    constructor(name, password) {
        this.name = name;
        privateData.set(this, { password });  // private!
    }
    checkPassword(pw) {
        return privateData.get(this).password === pw;
    }
}
```

---

## 26. Symbol

**Q: What is `Symbol` in JavaScript?**

A **primitive** data type that creates a **unique, immutable** identifier. Primarily used for object property keys to avoid name collisions.

```js
// Every Symbol is unique
const s1 = Symbol("id");
const s2 = Symbol("id");
s1 === s2;  // false — always unique, even with same description

// As object property key — hidden from normal iteration
const user = {
    name: "Alice",
    [Symbol("id")]: 123
};
Object.keys(user);               // ["name"] — symbol key hidden
JSON.stringify(user);             // '{"name":"Alice"}' — symbol hidden
Object.getOwnPropertySymbols(user);  // [Symbol(id)] — explicit access

// Well-known Symbols
class MyArray {
    static [Symbol.hasInstance](instance) {
        return Array.isArray(instance);
    }
}
[] instanceof MyArray;  // true

// Symbol.iterator — make object iterable
const range = {
    from: 1,
    to: 5,
    [Symbol.iterator]() {
        let current = this.from;
        const last = this.to;
        return {
            next() {
                return current <= last
                    ? { value: current++, done: false }
                    : { done: true };
            }
        };
    }
};
for (const n of range) console.log(n);  // 1, 2, 3, 4, 5

// Global Symbol registry
const s3 = Symbol.for("shared");
const s4 = Symbol.for("shared");
s3 === s4;  // true — same global symbol
```

---

## 27. Proxy & Reflect

**Q: What are `Proxy` and `Reflect`?**

**Proxy** intercepts operations on an object (get, set, delete, etc.). **Reflect** provides default behavior for those operations.

```js
const user = { name: "Alice", age: 25 };

const proxy = new Proxy(user, {
    get(target, prop) {
        console.log(`Reading ${prop}`);
        return Reflect.get(target, prop);   // default get behavior
    },
    set(target, prop, value) {
        if (prop === "age" && typeof value !== "number") {
            throw new TypeError("Age must be a number");
        }
        console.log(`Setting ${prop} = ${value}`);
        return Reflect.set(target, prop, value);
    },
    deleteProperty(target, prop) {
        if (prop === "name") {
            throw new Error("Cannot delete name");
        }
        return Reflect.deleteProperty(target, prop);
    }
});

proxy.name;          // logs: "Reading name" → "Alice"
proxy.age = 30;      // logs: "Setting age = 30"
proxy.age = "old";   // TypeError: Age must be a number
delete proxy.name;   // Error: Cannot delete name

// Practical: Validation, logging, reactive systems (Vue 3 uses Proxy)
```

---

## 28. Optional Chaining & Nullish Coalescing

**Q: What are `?.` and `??` operators?**

### Optional Chaining `?.`

Safely access nested properties — returns `undefined` instead of throwing if intermediate value is `null`/`undefined`.

```js
const user = { address: { city: "Delhi" } };

// Without optional chaining
const zip = user && user.address && user.address.zip;  // undefined

// With optional chaining
const zip = user?.address?.zip;  // undefined (no error)

// Works with methods and arrays
user.getAge?.();       // undefined if getAge doesn't exist
const arr = null;
arr?.[0];              // undefined (no error)
```

### Nullish Coalescing `??`

Returns right side only when left side is `null` or `undefined` (NOT for `0`, `""`, `false`).

```js
const value = null ?? "default";     // "default"
const value2 = undefined ?? "default"; // "default"
const value3 = 0 ?? "default";      // 0  ← NOT null/undefined
const value4 = "" ?? "default";     // "" ← NOT null/undefined
const value5 = false ?? "default";  // false

// vs OR operator ||
0 || "default";      // "default"  ← treats 0 as falsy
0 ?? "default";      // 0          ← only null/undefined

// Combined usage
const city = response?.data?.user?.address?.city ?? "Unknown";
```

---

## 29. Promise.all, Promise.race, Promise.allSettled, Promise.any

**Q: Explain the different Promise combinators.**

| Method | Resolves when | Rejects when |
|---|---|---|
| `Promise.all` | **All** fulfill | **Any one** rejects |
| `Promise.race` | **First** settles (fulfill or reject) | **First** settles |
| `Promise.allSettled` | **All** settle | **Never** rejects |
| `Promise.any` | **Any one** fulfills | **All** reject |

```js
const p1 = Promise.resolve(1);
const p2 = Promise.resolve(2);
const p3 = Promise.reject("Error");

// Promise.all — all must succeed
Promise.all([p1, p2])
    .then(results => console.log(results));  // [1, 2]

Promise.all([p1, p2, p3])
    .catch(err => console.log(err));         // "Error" (p3 failed)

// Promise.race — first to settle wins
Promise.race([
    new Promise(resolve => setTimeout(() => resolve("slow"), 2000)),
    new Promise(resolve => setTimeout(() => resolve("fast"), 500))
]).then(result => console.log(result));       // "fast"

// Promise.allSettled — wait for ALL, never rejects
Promise.allSettled([p1, p2, p3])
    .then(results => console.log(results));
// [
//   { status: "fulfilled", value: 1 },
//   { status: "fulfilled", value: 2 },
//   { status: "rejected", reason: "Error" }
// ]

// Promise.any — first to fulfill wins (ignores rejections)
Promise.any([p3, p1, p2])
    .then(result => console.log(result));    // 1 (first fulfilled)

Promise.any([p3, Promise.reject("err2")])
    .catch(err => console.log(err));         // AggregateError (all rejected)
```

---

## 30. Object.freeze vs Object.seal

**Q: What is the difference between `Object.freeze()` and `Object.seal()`?**

| Feature | `Object.freeze()` | `Object.seal()` |
|---|---|---|
| Add new properties | ❌ | ❌ |
| Remove properties | ❌ | ❌ |
| Modify existing values | ❌ | ✅ |
| Change property descriptors | ❌ | ❌ |
| Depth | **Shallow** only | **Shallow** only |

```js
// Object.freeze — nothing can change
const frozen = Object.freeze({ name: "Alice", age: 25 });
frozen.name = "Bob";     // ❌ silently fails (throws in strict mode)
frozen.city = "Delhi";   // ❌ can't add
delete frozen.name;      // ❌ can't delete
console.log(frozen);     // { name: "Alice", age: 25 }

// Object.seal — can modify values, can't add/remove keys
const sealed = Object.seal({ name: "Alice", age: 25 });
sealed.name = "Bob";     // ✅ allowed
sealed.city = "Delhi";   // ❌ can't add
delete sealed.name;      // ❌ can't delete
console.log(sealed);     // { name: "Bob", age: 25 }

// Both are SHALLOW — nested objects are NOT frozen/sealed
const obj = Object.freeze({ data: { value: 42 } });
obj.data.value = 100;    // ✅ works — nested object NOT frozen!

// Deep freeze
function deepFreeze(obj) {
    Object.freeze(obj);
    Object.getOwnPropertyNames(obj).forEach(prop => {
        if (typeof obj[prop] === "object" && obj[prop] !== null) {
            deepFreeze(obj[prop]);
        }
    });
    return obj;
}

// Checking
Object.isFrozen(frozen);  // true
Object.isSealed(sealed);  // true
Object.isFrozen(sealed);  // false (values can still change)
```

---

## Quick Revision Table

| # | Topic | One-Line Key Point |
|---|---|---|
| 1 | var/let/const | `var` function-scoped, `let`/`const` block-scoped, `const` no reassign |
| 2 | Hoisting | Declarations move to top; `var`=undefined, `let`/`const`=TDZ |
| 3 | Closures | Inner function remembers outer scope even after outer returns |
| 4 | this | Depends on how function is called, arrow inherits lexically |
| 5 | call/apply/bind | Explicitly set `this`; call=commas, apply=array, bind=returns fn |
| 6 | Event Loop | Microtasks (Promise) before macrotasks (setTimeout) |
| 7 | Promises | Async wrapper: pending → fulfilled/rejected |
| 8 | async/await | Syntactic sugar over Promises |
| 9 | == vs === | `==` coerces types, `===` strict comparison |
| 10 | Shallow/Deep Copy | Spread = shallow; `structuredClone` = deep |
| 11 | Prototypal Inheritance | Objects inherit via `__proto__` chain |
| 12 | Event Bubbling | Events propagate child → parent (capturing: parent → child) |
| 13 | Debounce/Throttle | Debounce = after pause; Throttle = at most once per interval |
| 14 | Currying | `f(a,b,c)` → `f(a)(b)(c)` |
| 15 | Spread/Rest | Spread = expand; Rest = collect remaining |
| 16 | Destructuring | Extract values from arrays/objects into variables |
| 17 | Map/Filter/Reduce | Transform / filter / accumulate arrays |
| 18 | null vs undefined | `undefined` = not assigned; `null` = intentionally empty |
| 19 | typeof/instanceof | `typeof` = primitive type; `instanceof` = prototype chain check |
| 20 | Arrow vs Regular | Arrow = lexical `this`, no `arguments`, no constructor |
| 21 | setTimeout/setInterval | Once after delay / repeat every interval |
| 22 | Callback Hell | Fix with Promises or async/await |
| 23 | Memoization | Cache function results for same inputs |
| 24 | Generators | `function*` with `yield` — pause/resume execution |
| 25 | WeakMap/WeakSet | Object keys only, allows garbage collection |
| 26 | Symbol | Unique immutable identifier for property keys |
| 27 | Proxy/Reflect | Intercept object operations (get/set/delete) |
| 28 | ?.  ?? | Optional chaining = safe access; ?? = null/undefined fallback |
| 29 | Promise combinators | all (all pass), race (first), allSettled (all), any (first pass) |
| 30 | freeze vs seal | freeze = read-only; seal = can modify values, no add/remove |
