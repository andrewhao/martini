# Martini

Meditations on tree-shaking.

## In the beginning...

We wrote modules in global scope:

```js
var lodash = {
  capitalize: function(s) { /*...*/ }
}

window._ = lodash
```

If you wanted to access `capitalize()` you needed to:

```html
<script src="lodash.js" type="text/javascript" />
```

This would "inject" lodash in `window._`

```javascript
// Anywhere in your script,
_.capitalize
```

And boom! You get it.

## Part 2: modularization

OK, but you have a problem now. You basically must load `lodash.js` everywhere the app is needed. For small web pages, this was fine. But as apps grew in complexity (SPAs), we needed new ways to declare dependencies.

Enter modularization. Because JS has no native module concept out of the box, we need a way to package up code.

### Modularization

Modularization is basically saying "I want to treat a software component as a black box. I want to choose when I can use it."

Here's a simple example of modularization using JS closures:

```javascript
var lodash = (function() {
  let foo = "I am a secret";
  let bar = 42;

  return {
    capitalize: function(s) { /* ... */ },
    secretsOfTheUniverse: function() { return bar; },
    shh: function() { return foo },
  }
})()

// lodash.capitalize('hello') // 'HELLO'
// lodash.secretsOfTheUniverse() // 42
// lodash.shh() // "I am a secret"
```

This is called "IIFE Modularization". It guarantees that nothing can "leak" out of the scope. It guarantees that nothing can pollute the scope either by coming before it. Variables `foo` and `bar` are guaranteed to live only inside the lexical scope of the enclosed expression.

This is also known as the "Revealing Module Pattern". Publicly accessible modules are revealed in the return object of the module. Everything else is private.

Further reading:
- https://ui.dev/javascript-modules-iifes-commonjs-esmodules/
- https://auth0.com/blog/javascript-module-systems-showdown/

## Going further with CommonJS

CommonJS was a server-side spec that took some of these concepts and went in a new direction. In this world:

- The file was the unit of modularization
- Introduction of `require` and `exports` keywords

Loosely, this is how a CommonJS module looks:

```javascript
// file: lodash.js

let foo = "I am a secret";
let bar = 42;

exports.capitalize = function(s) { /* ... */ },
exports.secretsOfTheUniverse = function() { return bar; },
exports.shh = function() { return foo },
```

By default, everything inside the module is private, unless explicitly exported on the `exports` object.

This also means that the file can now be `require`d, like this:

```javascript
// file: index.js
const _ = require('./lodash');
return _.capitalize('I am shouting');
```

#### Tangent: But what about the browser?

But the browser doesn't have `require` or `exports` - what do I do??

That's where tools like [browserify](https://github.com/browserify/browserify) and [webpack](https://github.com/webpack/webpack) come in.

How do these tools work? They run as a preprocessor before code is run and they inline each module into a "module map". Here's an example from:

*Browserify*

```javascript
{
  1: [function (require, module, exports) {
    let foo = "I am a secret";
    let bar = 42;
    
    exports.capitalize = function(s) { /* ... */ };
    exports.secretsOfTheUniverse = function() { return bar; };
    exports.shh = function() { return foo };
  }, {}],
  2: [function (require, module, exports) {
    const _ = require('./lodash.js');
    return _.capitalize('I am shouting');
  }, {"./lodash": 1}]
}
```

*Webpack*

```javascript
// the webpack bootstrap
(function (modules) {
    // The require function
    function __webpack_require__(moduleId) {
        // Check if module is in cache
        // Create a new module (and put it into the cache)
        // Execute the module function
        // Flag the module as loaded
        // Return the exports of the module
    }

    // Load entry module and return exports
    return __webpack_require__(0);
})
/************************************************************************/
([
    // index.js - our application logic
    /* 0 */
    function (module, exports, __webpack_require__) {
      const _ = __webpack_require__(1);
      return _.capitalize('I am shouting');
    },
    // lodash.js
    /* 1 */
    function (module, exports, __webpack_require__) {
        let foo = "I am a secret";
        let bar = 42;
        
        exports.capitalize = function(s) { /* ... */ };
        exports.secretsOfTheUniverse = function() { return bar; };
        exports.shh = function() { return foo };
    }
]);
```

A few observations:

- Both Webpack and Browserify wrap each CJS file in a closure that injects an equivalent `require`, `module`, and `exports` function.
- They both require preprocessing and an entry point in the system.
- Webpack can, with some degree of success, actually detect some dead code and eliminate it in the build. Can you see what it might eliminate from the build above?

Further reading:
- https://benclinkinbeard.com/posts/how-browserify-works/
- https://blog.ag-grid.com/webpack-tutorial-understanding-how-it-works/

### Enter ES Modules

ES modules was a new syntax introduced by the ES2015 standard:

- instead of `require`, it used `import`
- instead of `modules.exports` or `exports` it used `export`.
- possible to do "named" imports and exports, which explicitly chooses a subset of a module to import or export.
- ESM loader system is completely overhauled. Modules load asynchronously but are instatiated and initialized in order.

```javascript
// file: lodash.js

let foo = "I am a secret";
let bar = 42;

export const capitalize = function(s) { /* ... */ },
export const secretsOfTheUniverse = function() { return bar; },
export const shh = function() { return foo },
```

And again, in use:

```javascript
// index.js
import { capitalize } from './lodash';

export default capitalize('I am shouting');
```

This has some benefits:

- First of all, ES modules are *much* easier to statically parse and find dependencies, since you can explicitly declare a subset of a module that you depend on. If we used our commonjs pattern, we need to return regular objects, and that's an expensive operation!

Static analysis of dependency graphs help us know exactly what variable is in scope at exactly what point in time. That's how new tools like [rollup](https://www.rollupjs.org) have incredible tree-shaking capabilities!

Further reading:
- https://exploringjs.com/es6/ch_modules.html#_benefit-dead-code-elimination-during-bundling
- https://redfin.engineering/node-modules-at-war-why-commonjs-and-es-modules-cant-get-along-9617135eeca1
- https://nodejs.org/api/packages.html#determining-module-system

#### What's the catch?

This also has some drawbacks

First of all, ES Module syntax is not universally supported. It's supported in later versions of Node, but *there's a catch*. Node needs you to name ESM files with a `.mjs` ("module" js) extension. That, or your entire package is ESM and you change the package.json `type` field to `"module"`.

That is annoying!

### Introducing babel

So instead, we use `babel` a lot to transpile ES6/ESM syntax back down to ES5 syntax when using Node. Click [here](https://babeljs.io/repl#?browsers=defaults%2C%20not%20ie%2011%2C%20not%20ie_mob%2011&build=&builtIns=false&corejs=3.6&spec=false&loose=false&code_lz=JYWwDg9gTgLgBAbzgYwIZmDVAbYAvAUzgF84AzKCEOAcmwgBNUBnACxoG4AoNDLXQgAoarAtno0AlByA&debug=false&forceAllTransforms=false&shippedProposals=false&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=module&lineWrap=true&presets=env%2Creact%2Cstage-2&prettier=false&targets=&version=7.16.3&externalPlugins=&assumptions=%7B%7D) to see how this transforms the code:

```javascript
import { capitalize } from 'lodash';
capitalize('hello');
```

to

```javascript
"use strict";

var _lodash = require("lodash");

(0, _lodash.capitalize)('hello');
```

### Does babel interfere with tree-shaking?

Yes. Babel by default compiles down to CJS. We want to preserve ESM as much as possible. Google web.dev article suggests we should manually disable this setting:

https://developers.google.com/web/fundamentals/performance/optimizing-javascript/tree-shaking

We must make sure `preset-env` has `modules: false` to prevent babel from touching our ESM code.