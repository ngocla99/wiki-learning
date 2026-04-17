# The Module System

> Sources: Luciano Mammino & Mario Casciaro, 2024
> Raw: [Node.js Design Patterns](../../raw/nodejs/design-patterns.md)

## Overview

JavaScript lacked a built-in module system for years. The Node.js ecosystem adopted CommonJS as its default, while browsers relied on `<script>` tags and later AMD/UMD. In 2015, ES modules (ESM) were standardized as part of ES2015, and Node.js shipped stable ESM support in version 13.2 (2019). Today, ESM is the recommended module system for new projects, though CommonJS remains prevalent in legacy codebases.

## The Need for Modules

A module system addresses three fundamental software engineering needs:

- **Organizing code** -- split a codebase into multiple files for independent development and testing.
- **Code reuse and dependency management** -- share functionality across projects and resolve transitive dependencies without friction.
- **Encapsulation and information hiding** -- expose a clear public API while keeping implementation details private.

## The Revealing Module Pattern

Before ES modules existed, JavaScript developers used the **revealing module pattern** (an IIFE) to achieve encapsulation:

```js
const myModule = (() => {
  const privateFoo = () => {}
  const privateBar = []

  const exported = {
    publicFoo: () => {},
    publicBar: () => {},
  }

  return exported
})()

myModule.publicFoo    // [Function: publicFoo]
myModule.privateFoo   // undefined -- hidden
```

The self-invoking function creates a private scope. Only the returned object is accessible from outside. This concept is the foundation behind CommonJS's `require` wrapper and helps explain why module systems work the way they do.

## ES Modules

### Enabling ESM in Node.js

Node.js treats `.js` files as CommonJS by default. To opt into ESM:

1. **Recommended:** add `"type": "module"` to the nearest `package.json`.
2. Use the `.mjs` file extension (or `.cjs` to force CommonJS).
3. Use the `--experimental-default-type="module"` flag.
4. Use the `--experimental-detect-module` flag (default in Node.js 23+).

```json
{
  "name": "my-project",
  "version": "1.0.0",
  "type": "module"
}
```

### Named Exports and Imports

Everything in an ES module is private by default. Use `export` to make entities public:

```js
// logger.js
export function log(message) {
  console.log(message)
}

export const DEFAULT_LEVEL = 'info'

export const LEVELS = { error: 0, debug: 1, warn: 2, info: 4 }

export class Logger {
  constructor(name) { this.name = name }
  log(message) { console.log(`[${this.name}] ${message}`) }
}
```

Import with selective destructuring or a namespace:

```js
import { log, Logger } from './logger.js'        // named imports
import * as loggerModule from './logger.js'       // namespace import
import { log as myLog } from './logger.js'        // rename to avoid clashes
```

Importing a non-existent export causes a `SyntaxError` at runtime.

### Default Exports and Imports

A module can have a single default export. The consumer chooses the local name freely (no braces needed):

```js
// logger.js
export default class Logger { /* ... */ }

// main.js
import MyLogger from './logger.js'
```

Internally, a default export is a named export with the name `default`. You can verify this via `loggerModule.default` when using a namespace import.

### Mixed Exports

Named and default exports can coexist. When importing, the default must come first:

```js
import mylog, { info } from './logger.js'
```

**Community preference:** Named exports are generally favored over default exports because they enable better IDE autocompletion, automatic imports, and tree shaking. Some linters (e.g., Biome) warn against default exports. Notable exceptions include Node.js core modules and React.

### Module Identifiers (Specifiers)

| Type | Example | Notes |
|------|---------|-------|
| Relative | `./logger.js`, `../utils.js` | Path relative to importing file |
| Absolute | `file:///opt/app/config.js` | Full filesystem path |
| Bare | `fastify`, `node:http` | Resolved from `node_modules` or Node.js core |
| Deep import | `fastify/lib/logger.js` | Path within a package |

Use the `node:` prefix for core modules to avoid ambiguity with third-party packages.

### Static and Dynamic Imports

**Static imports** must appear at the top level and use constant string specifiers. This enables static analysis and tree shaking.

```js
// NOT valid -- imports cannot be inside control flow
if (condition) {
  import module1 from 'module1'  // SyntaxError
}
```

**Dynamic imports** use the `import()` operator, which returns a promise. They allow conditional and runtime-computed module loading:

```js
const selectedLanguage = process.argv[2]
const translationModule = `./strings-${selectedLanguage}.js`
const strings = await import(translationModule)
console.log(strings.HELLO)
```

### Module Resolution Algorithm

Node.js resolves module specifiers in three steps:

1. **File modules** -- specifiers starting with `/`, `./`, or `../` are resolved as filesystem paths (normalized to `file://` URLs).
2. **Core modules** -- bare specifiers are matched against Node.js built-ins (e.g., `http` resolves to `node:http`).
3. **Package modules** -- if no core match, Node.js walks up the directory tree searching `node_modules` folders until the filesystem root.

This algorithm allows each package to depend on its own version of a shared library, elegantly solving **dependency hell**:

```
myApp/
├── foo.js
└── node_modules/
    ├── depA/          ← version used by myApp
    ├── depB/
    │   └── node_modules/
    │       └── depA/  ← version used by depB
    └── depC/
        └── node_modules/
            └── depA/  ← version used by depC
```

You can inspect resolution at runtime with `import.meta.resolve('fastify/lib/logger.js')`.

### Loading Phases

ES module loading happens in three distinct phases:

1. **Construction (parsing)** -- starting from the entry point, the interpreter recursively follows all `import` statements depth-first, loading source from each file. Each module is visited only once, even in cycles.
2. **Instantiation** -- the interpreter walks the dependency tree bottom-up, creating named references in memory for every export. It then links imports to their corresponding exports. No code executes yet.
3. **Evaluation** -- code is executed bottom-up. By the time the entry point runs, all exports have been initialized.

Because construction must complete before any code runs, imports and exports must be static. This separation is what allows ES modules to handle circular dependencies cleanly.

### Read-Only Live Bindings

Imported bindings are **read-only** from the consumer's perspective but **live** -- they reflect changes made inside the exporting module:

```js
// counter.js
export let count = 0
export function increment() { count++ }

// main.js
import { count, increment } from './counter.js'
console.log(count)   // 0
increment()
console.log(count)   // 1
count++              // TypeError: Assignment to constant variable!
```

This is fundamentally different from CommonJS, which shallow-copies the exports object. In CJS, changes to primitive values inside the exporting module are invisible to the consumer.

### Circular Dependencies

ES modules handle circular dependencies gracefully thanks to the three-phase loading process. Consider modules `a.js` and `b.js` that import each other:

```js
// a.js
import * as bModule from './b.js'
export let loaded = false
export const b = bModule
loaded = true

// b.js
import * as aModule from './a.js'
export let loaded = false
export const a = aModule
loaded = true
```

```js
// main.js
import * as a from './a.js'
import * as b from './b.js'
console.log('a ->', a)  // loaded: true, b.loaded: true
console.log('b ->', b)  // loaded: true, a.loaded: true
```

Both modules see each other's fully evaluated state because imports are references, not copies. During parsing, cycles are detected and each module is visited only once. During instantiation, export names are linked as references. By evaluation time, bottom-up execution ensures all values are populated before the entry point runs.

In CommonJS, the same scenario would produce `loaded: false` for the inner references due to shallow copying and eager execution.

### Monkey Patching

A module with no exports can still have side effects by modifying other modules. This is called **monkey patching**:

```js
// colorizeLogger.js
import { logger } from './logger.js'

const originalInfo = logger.info
logger.info = message => originalInfo(`\x1b[32m${message}\x1b[0m`)

// main.js
import { logger } from './logger.js'
import './colorizeLogger.js'       // side-effect-only import

logger.info('Hello, World!')       // prints in green
```

**Key constraint:** You cannot reassign an imported binding (`logger = newObj` throws `TypeError`), but you *can* mutate properties of an imported object (`logger.info = ...`). This distinction matters when designing patchable modules.

Monkey patching is generally discouraged because it introduces hidden side effects and makes behavior unpredictable. It can also be a security risk (CWE-349). However, it is occasionally useful for testing (e.g., mocking filesystem calls in unit tests).

## CommonJS Modules

CommonJS uses `require()` to import and `module.exports` / `exports` to export:

```js
// math.cjs
'use strict'

function add(a, b) { return a + b }
module.exports = { add }

// main.cjs
'use strict'
const { add } = require('./math.cjs')
console.log(add(2, 3)) // 5
```

Key characteristics:

- `require()` is **synchronous** and **dynamic** -- modules are loaded and executed immediately.
- Because it is dynamic, `require()` can be used inside conditionals and loops, and specifiers can be computed at runtime.
- CommonJS does **not** run in strict mode by default (you must add `'use strict'` explicitly).
- Exports are **shallow-copied**, so changes to primitive values inside the exporting module are not visible to consumers.

## ESM vs CJS: Differences and Interoperability

### Strict Mode

ESM runs in strict mode implicitly. CJS does not -- you must opt in with `'use strict'`.

### Top-Level `await`

ESM supports `await` at the module's top level. In CJS, you must wrap it in an `async` function:

```js
// ESM
const data = await loadData()

// CJS
async function main() {
  const data = await loadData()
}
main()
```

### `this` Behavior

In an ESM module's top-level scope, `this` is `undefined`. In CJS, `this === exports`.

### Missing References in ESM

CommonJS globals like `require`, `exports`, `module`, `__filename`, and `__dirname` are not available in ES modules. Use `import.meta` instead:

```js
// Node.js >= 20.11
const __filename = import.meta.filename
const __dirname = import.meta.dirname

// Older versions
import { fileURLToPath } from 'node:url'
import { dirname } from 'node:path'
const __filename = fileURLToPath(import.meta.url)
const __dirname = dirname(__filename)
```

### Import Interoperability

**CJS from ESM:** Use a default import. Named imports from CJS modules are not always supported -- destructure after importing:

```js
import someModule from './someModule.cjs'
const { someFeature } = someModule
```

Alternatively, recreate `require()` inside an ESM file:

```js
import { createRequire } from 'node:module'
const require = createRequire(import.meta.url)
const { someFeature } = require('./someModule.cjs')
```

**ESM from CJS:** Use dynamic `import()` (since `require()` cannot load ESM):

```js
// main.cjs
'use strict'
async function main() {
  const { someFeature } = await import('./someModule.mjs')
  console.log(someFeature)
}
main()
```

The `--experimental-require-module` flag (potentially stable by now) allows `require()` to load ESM directly.

### Importing JSON Files

In CJS, `require('./data.json')` works out of the box. In ESM, there are several options:

```js
// 1. Import attribute (experimental)
import data from './sample.json' with { type: 'json' }

// 2. Dynamic import with attribute
const { default: data } = await import('./sample.json', {
  with: { type: 'json' },
})

// 3. Manual read + parse
import { readFile } from 'node:fs/promises'
const data = JSON.parse(await readFile(new URL('./sample.json', import.meta.url), 'utf-8'))

// 4. createRequire
import { createRequire } from 'node:module'
const require = createRequire(import.meta.url)
const data = require('./sample.json')
```

The `with { type: 'json' }` syntax exists for security -- it prevents the loader from accidentally executing code in a file with a `.json` extension.

## TypeScript Modules

TypeScript is a superset of JavaScript that compiles to JS. When modules are involved, the compiler must:

- **Detect** the input module format (ESM or CJS) for each file.
- **Resolve** module specifiers using rules matching the target host (e.g., Node.js).
- **Emit** output in the correct module format (ESM, CJS, or a mix).
- **Type-check** imported bindings for correctness.

### Key `tsconfig.json` Options

| Option | Recommended Value | Purpose |
|--------|-------------------|---------|
| `module` | `NodeNext` | Sets the output module format and guides detection rules |
| `moduleResolution` | `NodeNext` | Mirrors Node.js resolution (ESM + CJS rules) |
| `verbatimModuleSyntax` | `true` | Requires input syntax to match output format, preventing surprises |

With `verbatimModuleSyntax` enabled, TypeScript forces you to write imports in the form that will be emitted, making the relationship between source and output predictable.

Note that TypeScript can convert between module systems (e.g., ESM source emitted as CJS), but this flexibility can cause subtle interoperability issues. Keeping `verbatimModuleSyntax: true` minimizes these problems.
