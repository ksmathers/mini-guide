---
title: spl-01-introduction-to-javascript-and-node
description: Introductory course on JavaScript and Node.js for experienced programmers new to the language
published: true
date: 2026-03-07T00:00:00.000Z
tags: javascript, node, learning
editor: markdown
dateCreated: 2026-03-07T00:00:00.000Z
---

# Module 1: Introduction to JavaScript and Node.js

## 📋 Course Overview

This module introduces JavaScript as a **language** and **runtime environment** (Node.js) for developers who already understand programming concepts such as variables, functions, loops, and data structures. The focus is entirely on server-side and backend JavaScript — no browser DOM, no front-end frameworks.

### What You'll Learn
- The JavaScript type system and its quirks
- Variables, control flow, functions, closures, and the `this` keyword
- Arrays, objects, and destructuring — including destructuring in function parameters
- Asynchronous programming: callbacks, Promises (`all`, `allSettled`, `race`), and `async`/`await`
- The Node.js module system — CommonJS, ES Modules, dynamic `import()`, and `__dirname`
- Using `npm` to manage dependencies
- Writing a simple HTTP server from scratch

> 📌 **About "Foundation for Module 2" markers:** Sections marked with this callout cover patterns used directly in the next course (*Application Development with Electron*). Pay extra attention to these before moving on.

### Prerequisites
- Comfortable with at least one other programming language (Python, Java, C#, Go, etc.)
- Basic familiarity with a terminal and command line

### Environment
All exercises run entirely in a terminal — no browser required.

---

## ⚙️ Part 1: Environment Setup

### Install Node.js

#### macOS — fnm via Homebrew

**fnm** (Fast Node Manager) is the recommended option on macOS. It lets you switch Node versions per project and is faster and simpler than the older nvm.

```bash
brew install fnm

# Add to ~/.zshrc (or ~/.bashrc)
eval "$(fnm env --use-on-cd --shell zsh)"

# Reload your shell, then install the current LTS
fnm install --lts
fnm use --lts

# Verify
node --version   # e.g. v22.x.x
npm --version    # e.g. 10.x.x
```

> **Note:** If you only need one version of Node and prefer simplicity, `brew install node` also works. If your team or existing documentation uses **nvm**, that is fine too — the commands are nearly identical (`nvm install --lts`, `nvm use --lts`).

#### Linux / Proxmox LXC

Use fnm the same way (`curl -fsSL https://fnm.vercel.app/install | bash`), or use the [NodeSource binary distributions](https://github.com/nodesource/distributions) for a system-wide apt install. Avoid `apt install nodejs` directly — the version in most distro repos is too old.

```bash
# NodeSource — installs the current LTS via apt
curl -fsSL https://deb.nodesource.com/setup_lts.x | bash -
apt install -y nodejs

# Verify
node --version
npm --version
```

### The Node REPL

Node ships with an interactive REPL (Read-Eval-Print Loop). It is the fastest way to experiment with the language.

```bash
node
```

```
Welcome to Node.js v22.x.x.
Type ".help" for more information.
> 2 + 2
4
> "hello".toUpperCase()
'HELLO'
> .exit
```

Throughout this guide, examples prefixed with `>` can be run directly in the REPL. Longer examples should be saved to a `.js` file and run with `node filename.js`.

---

## ✅ Part 2: The JavaScript Language

### 2.1 Variables and Data Types

JavaScript has **three** variable declaration keywords. Use `const` by default, `let` when you need reassignment, and treat `var` as legacy code you will encounter but should not write.

```javascript
const name = "Alice";       // block-scoped, cannot be reassigned
let   count = 0;            // block-scoped, can be reassigned
var   old = true;           // function-scoped — avoid in new code
```

#### Primitive Types

JavaScript has seven primitive types:

| Type        | Example                    | Notes                                      |
|-------------|----------------------------|--------------------------------------------|
| `number`    | `42`, `3.14`, `NaN`, `Infinity` | All numbers are 64-bit floats         |
| `bigint`    | `9007199254740993n`        | Arbitrary-precision integers               |
| `string`    | `"hello"`, `'world'`, `` `hi` `` | Immutable sequence of UTF-16 code units |
| `boolean`   | `true`, `false`            |                                            |
| `null`      | `null`                     | Intentional absence of a value             |
| `undefined` | `undefined`                | Variable declared but not yet assigned     |
| `symbol`    | `Symbol("id")`             | Unique, opaque identifier                  |

Check a type with the `typeof` operator:

```javascript
typeof 42           // "number"
typeof "hello"      // "string"
typeof true         // "boolean"
typeof undefined    // "undefined"
typeof null         // "object"  ← famous historical bug, null is NOT an object
typeof Symbol()     // "symbol"
typeof {}           // "object"
typeof []           // "object"  ← use Array.isArray() to distinguish
typeof function(){} // "function"
```

#### Type Coercion — the big gotcha

JavaScript will silently convert types in many contexts. Understanding this prevents subtle bugs.

```javascript
// Loose equality (==) coerces types — almost always a bad idea
0   == false   // true  ← coercion
""  == false   // true
null == undefined // true

// Strict equality (===) does NOT coerce — always use this
0   === false   // false
""  === false   // false

// Addition with strings triggers string concatenation
"3" + 4    // "34"   (number coerced to string)
3   + "4"  // "34"

// Other math operators coerce strings to numbers
"6" - 2    // 4
"6" * 2    // 12
"abc" * 2  // NaN
```

#### Why `===` Is the Community Standard — a Brief History

JavaScript was created in 10 days, and the original spec allowed very broad implicit type coercion through `==`. This leads to famously surprising results:

```javascript
0    == ""     // true  ← empty string coerces to 0
0    == "0"    // true  ← "0" coerces to 0, but "" == "0" is false!
""   == "0"    // false ← coercion is not transitive
[]   == ![]    // true  ← deeply unintuitive: both sides coerce to 0
null == undefined // true  ← the one case where this is intentional
```

These were not theoretical edge cases — they caused real bugs in production codebases. If you come from Python, you are used to trusting `==` and writing `if value:` freely because Python's type system is strict enough to make those safe. In JavaScript, that instinct gets you into trouble: `if (value == true)` does not behave the way you expect for most values.

Over time the JavaScript community responded with a norm: **always use `===` unless you have a specific reason not to**. This convention is now:
- Enforced by default in popular linters (ESLint, StandardJS)
- Taught as the baseline in every modern JavaScript resource
- Considered "the safe choice" because it removes an entire class of bugs

The one deliberate exception you will sometimes see in real code is `value == null`, which is true for both `null` *and* `undefined` — a concise way to check "this value was never set." Everything else: use `===`.

> **Rule of thumb:** always use `===` and `!==`. The only intentional use of `==` in modern code is `value == null` as a shorthand for `value === null || value === undefined`.

#### Template Literals

Use backtick strings (`` ` ``) for interpolation and multi-line strings:

```javascript
const host = "tamlyn.ank.com";
const port = 3000;

const url = `http://${host}:${port}/api`;
console.log(url); // http://tamlyn.ank.com:3000/api

const multiLine = `
  SELECT *
  FROM users
  WHERE active = true
`;
```

---

### 2.2 Operators and Control Flow

#### Comparison and Logical Operators

```javascript
// Comparison — always use strict equality
x === y    // strict equal
x !== y    // strict not-equal
x >  y
x >= y
x <  y
x <= y

// Logical
a && b     // AND — returns first falsy value or last value
a || b     // OR  — returns first truthy value or last value
!a         // NOT

// Nullish coalescing — returns right side only if left is null or undefined
const port = config.port ?? 3000;   // use 3000 only if config.port is null/undefined

// Optional chaining — safely access nested properties
const city = user?.address?.city;   // undefined instead of a TypeError
```

#### Truthy and Falsy

Every value is either truthy or falsy in a boolean context. The **falsy** values are:
`false`, `0`, `""` (empty string), `null`, `undefined`, `NaN`

Everything else is truthy — including `[]`, `{}`, `"0"`, and `"false"`.

#### if / else / switch

```javascript
const status = 404;

if (status === 200) {
  console.log("OK");
} else if (status === 404) {
  console.log("Not Found");
} else {
  console.log("Other:", status);
}

switch (status) {
  case 200:
    console.log("OK");
    break;
  case 404:
    console.log("Not Found");
    break;
  default:
    console.log("Other:", status);
}
```

#### Loops

```javascript
// Classic for
for (let i = 0; i < 5; i++) {
  console.log(i);
}

// while
let n = 10;
while (n > 0) {
  n -= 3;
}

// for...of  — iterate values (arrays, strings, Maps, Sets, generators)
const colors = ["red", "green", "blue"];
for (const color of colors) {
  console.log(color);
}

// for...in  — iterate keys of an object (use with care on arrays)
const config = { host: "localhost", port: 8080 };
for (const key in config) {
  console.log(key, config[key]);
}
```

---

### 2.3 Functions

JavaScript functions are **first-class values** — they can be stored in variables, passed as arguments, and returned from other functions.

#### Declaration vs Expression vs Arrow

```javascript
// Function declaration — hoisted to the top of its scope
function add(a, b) {
  return a + b;
}

// Function expression — not hoisted
const multiply = function(a, b) {
  return a * b;
};

// Arrow function — shorter syntax, does NOT have its own `this`
const divide = (a, b) => a / b;

// Arrow with a body (when you need more than one statement)
const greet = (name) => {
  const msg = `Hello, ${name}!`;
  return msg;
};
```

#### Default and Rest Parameters

```javascript
function connect(host, port = 3000) {
  console.log(`Connecting to ${host}:${port}`);
}

connect("localhost");        // Connecting to localhost:3000
connect("localhost", 8080);  // Connecting to localhost:8080

// Rest parameter — collects remaining arguments into an array
function sum(...numbers) {
  return numbers.reduce((total, n) => total + n, 0);
}

sum(1, 2, 3, 4);  // 10
```

#### Closures and Scope

A closure is a function that **captures variables from its surrounding scope**. This is one of JavaScript's most powerful features.

```javascript
function makeCounter(start = 0) {
  let count = start;   // captured by the inner functions

  return {
    increment() { count++; },
    decrement() { count--; },
    value()     { return count; },
  };
}

const counter = makeCounter(10);
counter.increment();
counter.increment();
console.log(counter.value()); // 12

// The variable `count` is private — nothing outside can access it directly
```

#### The `this` Keyword

`this` refers to the object that is the **current execution context** of a function. Unlike most languages, the value of `this` in JavaScript is determined by *how the function is called*, not where it is defined.

```javascript
// In a method call, `this` is the object before the dot
const server = {
  name: "tamlyn",
  describe() {
    return `Server: ${this.name}`;  // this === server
  },
};
console.log(server.describe());  // "Server: tamlyn"

// Detached call — `this` becomes undefined in strict mode
const fn = server.describe;
fn();  // TypeError: Cannot read properties of undefined (reading 'name')
```

The most common place this trips developers up is inside **callbacks** on class methods:

```javascript
class Poller {
  constructor(interval) {
    this.interval = interval;
    this.count = 0;
  }

  // ❌ Regular function: `this` is lost when setInterval calls it
  startBroken() {
    setInterval(function () {
      this.count++;          // `this` is undefined here — TypeError
    }, this.interval);
  }

  // ✅ Arrow function: inherits `this` from the enclosing start() call
  start() {
    setInterval(() => {
      this.count++;          // `this` is the Poller instance ✓
      console.log(this.count);
    }, this.interval);
  }
}
```

Two reliable patterns to keep `this` correct when a method is used as a callback:

```javascript
class Handler {
  constructor() {
    this.log = [];
    // Pattern 1: bind in the constructor — works for any method
    this.onData = this.onData.bind(this);
  }

  onData(chunk) {
    this.log.push(chunk);  // `this` is always the Handler instance
  }
}

// Pattern 2: class field arrow function (preferred modern syntax)
class HandlerModern {
  log = [];

  // Defined as a class field — not a method, so `this` is always the instance
  onData = (chunk) => {
    this.log.push(chunk);
  };
}
```

> 📌 **Foundation for Module 2 (Electron):** Electron requires passing methods as callbacks to `ipcMain.on()`, `app.on()`, and other event APIs. If those methods are plain class functions, `this` will be `undefined` when Electron invokes them. Use **class field arrow functions** (`onEvent = () => { ... }`) consistently in your Electron main process code to avoid this class of bug entirely.

---

### 2.4 Arrays

JavaScript arrays are dynamic, heterogeneous, and zero-indexed.

```javascript
const servers = ["sierra", "tango", "victor", "xray"];

// Access
servers[0]            // "sierra"
servers.at(-1)        // "xray"  (negative indexing with .at())
servers.length        // 4

// Mutating methods
servers.push("yankee");       // append
servers.pop();                // remove last
servers.unshift("alpha");     // prepend
servers.shift();              // remove first
servers.splice(1, 2);         // remove 2 elements at index 1

// Non-mutating methods (return new arrays)
const upper = servers.map(s => s.toUpperCase());
const long  = servers.filter(s => s.length > 5);
const found = servers.find(s => s.startsWith("t"));    // "tango"
const idx   = servers.findIndex(s => s === "victor");  // 2
const has   = servers.includes("xray");                // true

// Aggregation
const total = [1, 2, 3, 4].reduce((acc, n) => acc + n, 0); // 10

// Iteration (side effects only — prefer map/filter for transformations)
servers.forEach(s => console.log(s));

// Flatten, slice, sort
const nested = [[1, 2], [3, 4]].flat();         // [1, 2, 3, 4]
const copy   = servers.slice(1, 3);             // elements at index 1 and 2
const sorted = [...servers].sort();             // sort a copy (sort mutates!)
```

#### Spread Operator

```javascript
const a = [1, 2, 3];
const b = [4, 5, 6];
const merged = [...a, ...b];          // [1, 2, 3, 4, 5, 6]
const withExtra = [...a, 99, ...b];   // [1, 2, 3, 99, 4, 5, 6]
```

#### Destructuring

```javascript
const [first, second, ...rest] = ["a", "b", "c", "d"];
// first = "a", second = "b", rest = ["c", "d"]

// Skip elements with commas
const [, , third] = ["x", "y", "z"];
// third = "z"

// Default values — used when the element at that index is undefined
const [a = 10, b = 20] = [5];
// a = 5  (from array), b = 20  (index 1 was undefined, default used)

// Swap two variables without a temp variable
let x = 1, y = 2;
[x, y] = [y, x];
// x = 2, y = 1
```

---

### 2.5 Objects

Objects are key-value maps. Keys are always strings (or Symbols); values can be anything.

```javascript
const server = {
  hostname: "sierra.ank.com",
  ip:       "10.0.42.38",
  cores:    16,
  memory:   128,
  isOnline: true,
};

// Property access
server.hostname       // "sierra.ank.com"
server["hostname"]    // same — bracket notation useful for dynamic keys

// Add / update / delete
server.rack = "A1";
server.cores = 32;
delete server.isOnline;

// Check existence
"hostname" in server          // true
server.hasOwnProperty("ip")  // true
```

#### Object Destructuring

```javascript
const { hostname, ip, cores = 8 } = server;
// cores defaults to 8 if not present on the object

// Rename on destructure
const { hostname: host, ip: address } = server;
// host = "sierra.ank.com", address = "10.0.42.38"

// Nested destructuring
const { memory, ...rest } = server;
// memory = 128, rest = { hostname, ip, cores, ... }

// Rename AND provide a default in the same destructure
const { hostname: name = "unknown", cores: cpuCount = 1 } = server;
// name = "sierra.ank.com"  (from server.hostname)
// cpuCount = 16            (from server.cores)
```

#### Destructuring in Function Parameters

Destructuring directly in a parameter list is one of the most common and readable patterns in modern JavaScript. It turns any object argument into what looks like named parameters — with defaults built in.

```javascript
// ❌ Without destructuring — must index into the argument inside the body
function printServer(server) {
  console.log(`${server.hostname} (${server.ip}) — ${server.memory}GB`);
}

// ✅ With parameter destructuring — named, self-documenting, with defaults
function printServer({ hostname, ip, memory = 64 }) {
  console.log(`${hostname} (${ip}) — ${memory}GB`);
}

printServer({ hostname: "sierra", ip: "10.0.42.38", memory: 128 });
// "sierra (10.0.42.38) — 128GB"

// ✅ Rest collects any extra properties into a single object
function startService({ name, port = 3000, ...options }) {
  console.log(`Starting ${name} on port ${port}`);
  console.log("Extra config:", options);
}

startService({ name: "api", port: 8080, debug: true, timeout: 5000 });
// Starting api on port 8080
// Extra config: { debug: true, timeout: 5000 }

// ✅ Works identically in arrow functions and async functions
const describe = ({ hostname, ip }) => `${hostname} @ ${ip}`;
const load = async ({ filePath, encoding = "utf8" }) => fs.readFile(filePath, encoding);
```

> 📌 **Foundation for Module 2 (Electron):** Every Electron IPC handler receives a structured payload object. You will write these signatures constantly:
> ```javascript
> // Electron ipcMain handler — preview of Module 2 syntax
> ipcMain.handle("read-file", async (event, { filePath, encoding = "utf8" }) => {
>   return fs.readFile(filePath, encoding);
> });
> ```
> Fluency with parameter destructuring is required before starting Module 2.

#### Spread and Object Merging

```javascript
const defaults = { port: 3000, timeout: 5000, retries: 3 };
const overrides = { port: 8080, retries: 5 };

const config = { ...defaults, ...overrides };
// { port: 8080, timeout: 5000, retries: 5 }
// Later keys override earlier ones
```

#### Shorthand Property and Method Syntax

```javascript
const host = "localhost";
const port = 3000;

// Shorthand: if variable name matches key name
const config = { host, port };
// Equivalent to { host: host, port: port }

// Method shorthand
const obj = {
  value: 42,
  double() {            // instead of double: function() { ... }
    return this.value * 2;
  },
};
```

#### Useful Object Utilities

```javascript
const obj = { a: 1, b: 2, c: 3 };

Object.keys(obj)    // ["a", "b", "c"]
Object.values(obj)  // [1, 2, 3]
Object.entries(obj) // [["a", 1], ["b", 2], ["c", 3]]

// Build an object from entries
const doubled = Object.fromEntries(
  Object.entries(obj).map(([k, v]) => [k, v * 2])
);
// { a: 2, b: 4, c: 6 }
```

---

### 2.6 Classes

JavaScript classes are syntactic sugar over prototype-based inheritance.

```javascript
class Service {
  #name;       // private field (# prefix)
  #port;

  constructor(name, port) {
    this.#name = name;
    this.#port = port;
    this.connections = 0;
  }

  get address() {
    return `${this.#name}:${this.#port}`;
  }

  connect() {
    this.connections++;
    console.log(`Connected to ${this.address}`);
  }

  static create(name, port) {
    return new Service(name, port);
  }
}

class DatabaseService extends Service {
  constructor(name, port, database) {
    super(name, port);
    this.database = database;
  }

  connect() {
    super.connect();
    console.log(`Using database: ${this.database}`);
  }
}

const db = new DatabaseService("postgres", 5432, "wiki");
db.connect();
console.log(db.connections); // 1
// db.#name  — SyntaxError: private field
```

---

### 2.7 Error Handling

```javascript
// try / catch / finally
try {
  const data = JSON.parse("{ bad json }");
} catch (err) {
  console.error("Parse failed:", err.message);
} finally {
  console.log("This always runs");
}

// Throwing custom errors
function requirePositive(n) {
  if (n <= 0) {
    throw new RangeError(`Expected a positive number, got ${n}`);
  }
  return n;
}

// Custom error subclass
class ConfigError extends Error {
  constructor(key, message) {
    super(message);
    this.name = "ConfigError";
    this.key = key;
  }
}

try {
  throw new ConfigError("DB_HOST", "DB_HOST environment variable is not set");
} catch (err) {
  if (err instanceof ConfigError) {
    console.error(`Config problem with key '${err.key}': ${err.message}`);
  } else {
    throw err;  // re-throw anything you don't handle
  }
}
```

---

### 2.8 Asynchronous JavaScript

This is one of the most important topics for Node.js development. JavaScript is **single-threaded** — there is only one execution thread. Asynchronous I/O is handled by an **event loop** that processes callbacks when operations complete.

#### The Problem

```javascript
// This is sequential and BLOCKS the thread for every file read
// (Don't do this in production — shown here for contrast)
const fs = require("fs");
const data = fs.readFileSync("/etc/hostname", "utf8");
console.log(data);
```

#### Callbacks (the original pattern)

```javascript
const fs = require("fs");

fs.readFile("/etc/hostname", "utf8", (err, data) => {
  if (err) {
    console.error("Failed:", err.message);
    return;
  }
  console.log("Hostname:", data.trim());
});

console.log("This prints BEFORE the file is read");
```

Callbacks become unwieldy when operations depend on each other ("callback hell"):

```javascript
readFile(pathA, (err, a) => {
  if (err) return handleErr(err);
  readFile(pathB, (err, b) => {
    if (err) return handleErr(err);
    writeFile(pathC, a + b, (err) => {
      if (err) return handleErr(err);
      console.log("Done");
    });
  });
});
```

#### Promises

A `Promise` represents a value that will be available in the future. It is either *pending*, *fulfilled*, or *rejected*.

```javascript
const fs = require("fs/promises");  // promise-based fs API

fs.readFile("/etc/hostname", "utf8")
  .then(data => {
    console.log("Hostname:", data.trim());
    return fs.readFile("/etc/os-release", "utf8");
  })
  .then(osData => {
    console.log("OS:", osData.slice(0, 80));
  })
  .catch(err => {
    console.error("Error:", err.message);
  })
  .finally(() => {
    console.log("Done");
  });

// Create a promise manually
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

delay(1000).then(() => console.log("1 second later"));

// Promise.all — runs concurrently; rejects immediately if ANY promise rejects
Promise.all([
  fs.readFile("/etc/hostname", "utf8"),
  fs.readFile("/etc/os-release", "utf8"),
]).then(([hostname, os]) => {
  console.log(hostname.trim(), os.slice(0, 40));
});

// Promise.allSettled — waits for ALL to finish regardless of failure
// Each result is { status: "fulfilled", value } or { status: "rejected", reason }
Promise.allSettled([
  fs.readFile("/etc/hostname", "utf8"),
  fs.readFile("/nonexistent", "utf8"),   // this one will fail
]).then((results) => {
  for (const result of results) {
    if (result.status === "fulfilled") {
      console.log("OK:", result.value.trim());
    } else {
      console.warn("Failed:", result.reason.message);
    }
  }
});
// Use allSettled when partial failure is acceptable (e.g. reading optional config files)

// Promise.race — resolves or rejects with the FIRST promise to settle
// Classic use: implementing a timeout wrapper
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`Timed out after ${ms}ms`)), ms)
  );
  return Promise.race([promise, timeout]);
}

withTimeout(fetch("http://slow-service/api"), 3000)
  .then(res  => console.log("Got response"))
  .catch(err => console.error(err.message));  // "Timed out after 3000ms"

// util.promisify — converts an old-style Node.js callback API into a Promise-returning function
// Required signature: function(arg1, ..., callback(err, result))
const { promisify } = require("util");
const dns = require("dns");

const lookup = promisify(dns.lookup);

lookup("tamlyn.ank.com").then(({ address }) => {  // object destructuring on the result
  console.log("IP:", address);
});
// Use promisify for any third-party callback library you can't swap for a promise-based one
```

> 📌 **Foundation for Module 2 (Electron):** `ipcRenderer.invoke()` returns a Promise. You will frequently use `Promise.all` to fire multiple IPC calls concurrently, `allSettled` when fetching data from optional sources, and `withTimeout` to guard against a hung main process. These patterns appear throughout real Electron applications.

#### async / await — the modern approach

`async`/`await` is syntactic sugar over Promises. It makes asynchronous code read like synchronous code.

```javascript
const fs = require("fs/promises");

async function readSystemInfo() {
  try {
    const hostname = await fs.readFile("/etc/hostname", "utf8");
    const os       = await fs.readFile("/etc/os-release", "utf8");

    return {
      hostname: hostname.trim(),
      os:       os.split("\n")[0],
    };
  } catch (err) {
    console.error("Could not read system info:", err.message);
    throw err;
  }
}

// An async function always returns a Promise
readSystemInfo().then(info => console.log(info));

// Or use await at the top level (Node.js 14.8+ with ES modules, or inside another async function)
const info = await readSystemInfo();
console.log(info);
```

**Key rules for `async`/`await`:**
- `await` can only be used inside an `async` function (or at the top level of an ES module)
- `await` pauses **only** the current `async` function — the event loop continues
- Always wrap `await` calls in `try/catch` for error handling
- Use `Promise.all([...])` when operations can run concurrently instead of `await`-ing them sequentially

```javascript
// ❌ Sequential — takes 2 seconds total
const a = await fetchA();  // 1 second
const b = await fetchB();  // 1 second

// ✅ Concurrent — takes 1 second total
const [a, b] = await Promise.all([fetchA(), fetchB()]);
```

#### `await` in Loops — a Critical Gotcha

`Array.prototype.forEach` does **not** understand `async`/`await`. Using `await` inside `forEach` will not pause the loop — the awaited Promises are fired and immediately discarded.

```javascript
const files = ["a.txt", "b.txt", "c.txt"];

// ❌ WRONG — forEach ignores the returned Promise
// All three reads fire simultaneously and the outer function does NOT wait for them
files.forEach(async (file) => {
  const content = await fs.readFile(file, "utf8");  // this await is silently lost
  console.log(content);
});
console.log("This prints BEFORE any file is read");

// ✅ for...of respects await — reads happen one at a time, in order
for (const file of files) {
  const content = await fs.readFile(file, "utf8");
  console.log(content);
}

// ✅ Concurrent reads with ordered results — faster than sequential for...of
const contents = await Promise.all(files.map(f => fs.readFile(f, "utf8")));
contents.forEach(c => console.log(c));
// map() is fine here because it only collects Promises — it doesn't await them
```

> **Rule:** Inside an `async` function, use `for...of` for sequential async iteration. Use `Promise.all(array.map(...))` for concurrent async iteration. Never use `forEach` with `await`.

#### Running `async` Code at the Top Level

`await` is only valid inside an `async` function. In a CommonJS file (the Node.js default), the top-level scope is **not** async, so you need a wrapper:

```javascript
// CommonJS — wrap startup code in an async IIFE
// IIFE = Immediately Invoked Function Expression: defined and called in one step
(async () => {
  const data = await fs.readFile("/etc/hostname", "utf8");
  console.log(data.trim());
})();
//  ^^ the () here immediately calls the async function

// ES Module (.mjs or "type":"module" in package.json) — top-level await is built-in
// No IIFE wrapper needed:
const data = await fs.readFile("/etc/hostname", "utf8");
console.log(data.trim());
```

> 📌 **Foundation for Module 2 (Electron):** Electron's main process is a CommonJS module by default. Your `app.whenReady()` setup code — creating the browser window, loading config, registering IPC handlers — must all run inside an async function. The async IIFE is the standard pattern for this:
> ```javascript
> // Standard Electron main process entry point pattern
> app.whenReady().then(async () => {
>   const config = await loadConfig();
>   createWindow(config);
> });
> ```
> Recognising and writing the async IIFE and `.then(async () => ...)` patterns fluently is required.

---

## ✅ Part 3: Node.js Fundamentals

### 3.1 The Module System

Node.js has two module formats. **CommonJS** (`require`/`module.exports`) is the original format used by most existing packages. **ES Modules** (`import`/`export`) is the modern standard. Know both.

#### CommonJS (`.js` or `.cjs`)

```javascript
// math.js — exporting
function add(a, b) { return a + b; }
function sub(a, b) { return a - b; }
const PI = 3.14159;

module.exports = { add, sub, PI };
// or export a single value:  module.exports = add;
```

```javascript
// main.js — importing
const { add, PI } = require("./math");    // relative path needs "./"
const path        = require("path");      // built-in module, no "./"
const express     = require("express");   // installed package, no "./"

console.log(add(2, 3));  // 5
console.log(PI);         // 3.14159
```

#### ES Modules (`.mjs` or `"type": "module"` in package.json)

```javascript
// math.mjs — exporting
export function add(a, b) { return a + b; }
export const PI = 3.14159;
export default function multiply(a, b) { return a * b; }
```

```javascript
// main.mjs — importing
import multiply, { add, PI } from "./math.mjs";
import { readFile } from "fs/promises";
import path from "path";
```

> **Which to use?** For new projects, ES Modules are preferred. For quick scripts and anything integrating with older Node.js tooling, CommonJS is still common. The examples in Part 4 use CommonJS for maximum compatibility.

#### `__dirname` and `__filename` (CJS) vs `import.meta.url` (ESM)

CommonJS provides two global variables that give you the path to the currently executing file. They are **not available in ESM** without a shim.

```javascript
// CommonJS — available automatically in every .js / .cjs file
console.log(__filename);  // /home/user/my-api/src/index.js  (absolute path to this file)
console.log(__dirname);   // /home/user/my-api/src           (directory containing this file)

// Build a path relative to the SOURCE FILE, not the process working directory
const path = require("path");
const configPath = path.join(__dirname, "../config/settings.json");
//                                      ^^^ one directory up from src/
// This path is correct regardless of where `node` was invoked from
```

In an ES Module, `__dirname` and `__filename` do not exist. Reconstruct them from `import.meta.url`:

```javascript
// ES Module equivalent
import { fileURLToPath } from "url";
import { dirname, join }  from "path";

const __filename = fileURLToPath(import.meta.url);
const __dirname  = dirname(__filename);

// Now use them exactly as you would in CJS
const configPath = join(__dirname, "../config/settings.json");
```

> 📌 **Foundation for Module 2 (Electron):** Electron's main process uses `__dirname` constantly to resolve asset paths that must be correct regardless of where the application is installed — the HTML entry point, preload scripts, icon files, and more. This is one of the first things you will write in any Electron main process file:
> ```javascript
> mainWindow.loadFile(path.join(__dirname, "renderer/index.html"));
> ```
> You must understand `__dirname` and why `path.join` is used with it before starting Module 2.

#### Dynamic `import()` — Loading Modules On Demand

Static `import`/`require` at the top of a file loads a module before any code runs. Dynamic `import()` is a **function** that loads a module at runtime, returning a Promise. Use it for conditional or lazy loading.

```javascript
// Static require — always loads at startup
const config = require("./config");

// Dynamic import — loads only when called, works in both CJS and ESM
async function loadPlugin(name) {
  // The plugin module is only loaded when this function is actually called
  const plugin = await import(`./plugins/${name}.mjs`);
  return plugin.default;   // access the default export from the dynamically loaded module
}

const logger = await loadPlugin("file-logger");
logger.info("Plugin loaded on demand");

// Useful for large optional features you don't want to load unconditionally
```

---

### 3.2 npm and package.json

`npm` is the Node.js package manager. Every Node.js project starts with a `package.json` manifest.

```bash
# Create a new project
mkdir my-api && cd my-api
npm init -y        # generate package.json with defaults

# Install a dependency
npm install express

# Install a dev-only dependency (not bundled in production)
npm install --save-dev nodemon

# Install all dependencies listed in package.json
npm install

# Run a script defined in package.json
npm run dev
```

A typical `package.json`:

```json
{
  "name": "my-api",
  "version": "1.0.0",
  "description": "Example Node.js API",
  "main": "src/index.js",
  "scripts": {
    "start":  "node src/index.js",
    "dev":    "nodemon src/index.js",
    "test":   "node --test"
  },
  "dependencies": {
    "express": "^4.19.2"
  },
  "devDependencies": {
    "nodemon": "^3.1.0"
  }
}
```

> **Commit `package.json` and `package-lock.json`** to version control. Add `node_modules/` to `.gitignore`.

---

### 3.3 Core Built-in Modules

No installation needed — these are part of Node.js.

#### `path` — File path manipulation

```javascript
const path = require("path");

path.join("/home", "user", "data", "file.txt");  // /home/user/data/file.txt
path.resolve("./config.json");                   // absolute path from cwd
path.dirname("/home/user/file.txt");             // /home/user
path.basename("/home/user/file.txt");            // file.txt
path.extname("archive.tar.gz");                  // .gz
path.parse("/home/user/file.txt");
// { root: '/', dir: '/home/user', base: 'file.txt', ext: '.txt', name: 'file' }
```

#### `fs/promises` — File system

```javascript
const fs   = require("fs/promises");
const path = require("path");

async function example() {
  // Read a file
  const content = await fs.readFile("/etc/hostname", "utf8");
  console.log(content.trim());

  // Write a file
  await fs.writeFile("output.txt", "hello\n", "utf8");

  // Append
  await fs.appendFile("output.txt", "world\n");

  // Check existence
  try {
    await fs.access("somefile.txt");
    console.log("exists");
  } catch {
    console.log("does not exist");
  }

  // List directory
  const entries = await fs.readdir(".", { withFileTypes: true });
  for (const entry of entries) {
    console.log(entry.isDirectory() ? "[DIR]" : "     ", entry.name);
  }

  // Create directory (recursive = mkdir -p)
  await fs.mkdir("./data/logs", { recursive: true });

  // Delete
  await fs.unlink("output.txt");
  await fs.rmdir("./data", { recursive: true });

  // File metadata
  const stat = await fs.stat("/etc/hostname");
  console.log("size:", stat.size, "modified:", stat.mtime);
}

example();
```

#### `os` — Operating system information

```javascript
const os = require("os");

os.hostname()       // "tamlyn"
os.platform()       // "linux"
os.arch()           // "x64"
os.cpus().length    // number of logical CPUs
os.totalmem()       // total RAM in bytes
os.freemem()        // free RAM in bytes
os.uptime()         // seconds since boot
os.homedir()        // "/root" or "/home/user"
os.tmpdir()         // "/tmp"
os.networkInterfaces() // all network interfaces with IPs
```

#### `http` — HTTP server (low-level)

```javascript
const http = require("http");

const server = http.createServer((req, res) => {
  const { method, url } = req;
  console.log(`${method} ${url}`);

  if (method === "GET" && url === "/") {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ status: "ok", hostname: os.hostname() }));
  } else {
    res.writeHead(404, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ error: "Not Found" }));
  }
});

server.listen(3000, () => {
  console.log("Server listening on http://localhost:3000");
});
```

#### `events` — EventEmitter

Many Node.js APIs (HTTP servers, streams, etc.) are built on `EventEmitter`. Understanding it helps when reading Node.js documentation.

> 📌 **Foundation for Module 2 (Electron):** Electron's entire inter-process communication (IPC) system is built on the EventEmitter pattern. `ipcMain.on("channel", handler)` is functionally equivalent to `emitter.on("event", handler)`. The vocabulary you learn here — `on`, `once`, `emit`, `removeListener` — maps directly to how the Electron main process and renderer process communicate with each other.

```javascript
const { EventEmitter } = require("events");

class JobQueue extends EventEmitter {
  constructor() {
    super();
    this.queue = [];
  }

  add(job) {
    this.queue.push(job);
    this.emit("job:added", job);
  }

  process() {
    const job = this.queue.shift();
    if (!job) {
      this.emit("empty");
      return;
    }
    // simulate work
    setImmediate(() => {
      this.emit("job:done", job);
      this.process();
    });
  }
}

const q = new JobQueue();

q.on("job:added", (job) => console.log("Added:", job));
q.on("job:done",  (job) => console.log("Done: ", job));
q.on("empty",     ()    => console.log("Queue empty"));

q.add("compile");
q.add("test");
q.add("deploy");
q.process();
```

#### `process` — Current process information

```javascript
process.env.NODE_ENV        // "development", "production", etc.
process.env.PORT            // environment variable
process.argv                // command-line arguments as an array
process.cwd()               // current working directory
process.pid                 // process ID
process.exit(0)             // exit with code 0 (success)
process.exit(1)             // exit with code 1 (error)

// Read environment variables with defaults
const PORT = parseInt(process.env.PORT ?? "3000", 10);
const ENV  = process.env.NODE_ENV ?? "development";

// Handle unhandled rejections (always add this)
process.on("unhandledRejection", (reason) => {
  console.error("Unhandled rejection:", reason);
  process.exit(1);
});
```

---

### 3.4 A Complete Mini-Application

This example ties together what you have learned: modules, async/await, HTTP, environment variables, and error handling.

Create the following file structure:

```
my-api/
├── package.json
└── src/
    ├── index.js
    └── routes.js
```

**`package.json`**
```json
{
  "name": "my-api",
  "version": "1.0.0",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js"
  }
}
```

**`src/routes.js`**
```javascript
const os   = require("os");
const fs   = require("fs/promises");
const path = require("path");

async function handleHealth(req, res) {
  res.writeHead(200, { "Content-Type": "application/json" });
  res.end(JSON.stringify({
    status:   "ok",
    hostname: os.hostname(),
    uptime:   os.uptime(),
    memory: {
      total: os.totalmem(),
      free:  os.freemem(),
    },
  }));
}

async function handleFiles(req, res) {
  try {
    const dir     = process.env.FILES_DIR ?? ".";
    const entries = await fs.readdir(dir, { withFileTypes: true });
    const files   = entries
      .filter(e => e.isFile())
      .map(e => ({
        name: e.name,
        path: path.join(dir, e.name),
      }));

    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ files }));
  } catch (err) {
    res.writeHead(500, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ error: err.message }));
  }
}

module.exports = { handleHealth, handleFiles };
```

**`src/index.js`**
```javascript
const http   = require("http");
const { handleHealth, handleFiles } = require("./routes");

const PORT = parseInt(process.env.PORT ?? "3000", 10);

const server = http.createServer(async (req, res) => {
  const { method, url } = req;
  console.log(`${new Date().toISOString()} ${method} ${url}`);

  try {
    if (method === "GET" && url === "/health") {
      await handleHealth(req, res);
    } else if (method === "GET" && url === "/files") {
      await handleFiles(req, res);
    } else {
      res.writeHead(404, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ error: "Not Found" }));
    }
  } catch (err) {
    console.error("Unhandled error:", err);
    res.writeHead(500, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ error: "Internal Server Error" }));
  }
});

server.listen(PORT, () => {
  console.log(`API server running on http://localhost:${PORT}`);
  console.log("Routes:");
  console.log("  GET /health");
  console.log("  GET /files");
});

process.on("SIGTERM", () => {
  console.log("SIGTERM received, shutting down");
  server.close(() => process.exit(0));
});
```

Run it:

```bash
node src/index.js
# API server running on http://localhost:3000

curl http://localhost:3000/health
curl http://localhost:3000/files
```

---

## ✅ Part 4: Practice Exercises

### Exercise 1 — Language Fundamentals

Write a function `summarize(servers)` that accepts an array of server objects (each with `hostname`, `ip`, and `memory` fields) and returns:
- The server with the most memory
- The total memory across all servers
- An array of hostnames sorted alphabetically

```javascript
const servers = [
  { hostname: "sierra", ip: "10.0.42.38", memory: 128 },
  { hostname: "tango",  ip: "10.0.42.39", memory: 32  },
  { hostname: "victor", ip: "10.0.42.45", memory: 64  },
  { hostname: "xray",   ip: "10.0.42.46", memory: 64  },
];

// Your solution here

// Expected output:
// { largest: { hostname: "sierra", ... }, total: 288, sorted: ["sierra", "tango", "victor", "xray"] }
```

### Exercise 2 — Async / File System

Write an async function `countLines(filePath)` that reads a text file and returns the number of lines. Then write `countAllLines(dir)` that reads every `.js` file in a given directory concurrently and returns an object mapping filenames to line counts.

```javascript
async function countLines(filePath) { /* ... */ }

async function countAllLines(dir) { /* ... */ }
```

### Exercise 3 — HTTP Server

Extend the mini-application from Part 3 with two new routes:

1. `GET /env` — returns a JSON object of safe environment variables (exclude anything with `SECRET`, `PASSWORD`, or `KEY` in the name)
2. `GET /uptime` — returns `{ uptime: <seconds>, human: "<Xh Ym Zs>" }`

### Exercise 4 — EventEmitter

Build a `RateLimiter` class that extends `EventEmitter`:
- Tracks requests per IP address over a rolling 60-second window
- Emits `"allowed"` when a request is within the limit
- Emits `"blocked"` when a request exceeds a configurable `maxRequests` limit
- Emits `"reset"` when a window expires for an IP

---

## 📋 Key Concepts Summary

| Concept | Key Points |
|---|---|
| **`===` vs `==`** | Always use `===`. `==` performs type coercion. |
| **`const`/`let`** | Default to `const`. Use `let` only when reassignment is needed. Avoid `var`. |
| **Arrow functions** | Shorter syntax; no own `this`. Use for callbacks and lambdas. |
| **Closures** | Inner functions capture variables from their enclosing scope. |
| **Promises / async-await** | `async`/`await` is preferred; it compiles to Promises. Always `try/catch`. |
| **Concurrent vs sequential** | Use `Promise.all()` when independent operations can run at the same time. |
| **`this` keyword** | Determined by *how* a function is called. Arrow functions inherit `this` from the outer scope; regular functions get a new `this` at call time. |
| **Destructuring in params** | `function f({ name, port = 3000 })` — turns any object argument into named parameters with inline defaults. |
| **`Promise.all` vs `allSettled`** | `all` fails fast if any promise rejects. `allSettled` waits for all and reports each as `fulfilled` or `rejected`. |
| **`await` in `forEach`** | `forEach` silently discards returned Promises. Use `for...of` for sequential async loops. |
| **Async IIFE** | `(async () => { await ... })()` — the standard pattern for running async startup code at the CJS module top level. |
| **`__dirname`** | CJS global: the directory of the currently executing file. Essential for building portable asset paths. |
| **CommonJS vs ESM** | `require`/`module.exports` (CommonJS) and `import`/`export` (ESM) coexist. |
| **Event loop** | JavaScript is single-threaded; never block with synchronous I/O in servers. |

---

## 🔧 Troubleshooting Common Mistakes

- **`Cannot read properties of undefined`** — You accessed a property on a value that is `undefined`. Use optional chaining (`?.`) or check for `null`/`undefined` first.
- **`UnhandledPromiseRejection`** — An `async` function threw or a Promise rejected with no `.catch()`. Always `try/catch` inside `async` functions.
- **`require is not defined`** — You are running an ES Module file (`.mjs` or `"type":"module"`) but used `require`. Use `import` instead, or rename the file to `.cjs`.
- **`Cannot use import statement in a module`** — Opposite problem: you used `import` in a CommonJS context. Either add `"type":"module"` to `package.json` or switch to `require`.
- **Callback executed before data is ready** — You forgot to `await` a Promise, or you called a callback-based API and used the result synchronously. Check that every async operation is properly awaited.
- **`await` inside `forEach` does nothing** — `forEach` fires all the callbacks and returns immediately; it does not wait for any returned Promises. Replace `forEach` with `for...of` for sequential async iteration, or `Promise.all(array.map(...))` for concurrent.
- **`this` is `undefined` inside a callback** — A class method was passed as a callback (e.g. to `setTimeout`, `addEventListener`, or an IPC API) and lost its instance binding. Fix it with a class field arrow function (`onClick = () => { ... }`) or by calling `.bind(this)` in the constructor.
- **Port already in use (EADDRINUSE)** — Another process is using the port. Find it with `lsof -i :3000` and kill it, or change your port.

---

## 🔗 What's Next

- **Module 2**: Application Development with Electron — main/renderer processes, IPC, menus, and native file system integration
- **Module 3**: Building REST APIs with Express — routing, middleware, request parsing, and error handling
- **Module 4**: Working with Databases — PostgreSQL with `pg`, SQLite with `better-sqlite3`, and basic query patterns
- **Module 5**: Streams and Buffers — efficient handling of large files and network data
- **Module 6**: Testing — the built-in `node:test` runner, assertions, and test structure

---

*Course version 1.0 — March 2026*
