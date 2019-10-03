# Declarations in Conditionals

ECMAScript proposal for allowing variable declarations inside conditional statements (e.g. `if`/`while`).

Authors:
 - Devin Rousso

Stage: 1

## Overview

When programming in C++, an extremely useful feature is to be able to declare a variable _and_ have it be evaluated inside a conditional:

```cpp
if (auto* ptr = getPtr()) {
    /* ... */
}
```

Adding this capability to JavaScript would be very useful for the following reasons:
 - avoid having to retype the variable name
 - finer-grain "control" over the visibility of the variable
 - allow authors to write performance-"safe" code without having to know the specific details of the code being called (see example below)

In the case of JavaScript, however, there should be some limitations:
 - only using `let` and `const`
 - only for `if` and `while`
 - destructuring is not allowed (even if only a single variable is "pulled out"), as there's potential confusion as to what's actually being checked (the value vs. whether the desired key/index was present in the object/array)
 
There would be no "new" syntax (it would be the same as declaring a single `let`/`const` variable, except without an ending semicolon) and the variable would only be visible inside the `if`/`while` block.

## Examples

Here's an example of where allowing declarations in conditionals could be useful:

```js
class Foo {
    get data() {
        let result = [];
        /* ... do some expensive work ... */
        return result;
    }
}

let foo = new Foo;
if (foo.data) {
    for (let item of foo.data) {
        /* A */
    }
} else {
    /* B */
}
````

could be replaced by

```js
class Foo {
    get data() {
        let result = [];
        /* ... do some expensive work ... */
        return result;
    }
}

let foo = new Foo;
if (let data = foo.data) {
    for (let item of data) {
        /* A */
    }
} else {
    /* B */
}
```

which allows `foo.data` to only have to be evaluated once, and is much more stylistically succinct.

One could create another variable (e.g. `let data = foo.data;`), but that could potentially keep `foo.data` (via `data`) alive much longer than needed and would "pollute" the scope with an additional variable.

## Transpiler Support

This can (mostly) be transpiled into

```js
class Foo {
    get data() {
        let result = [];
        /* ... do some expensive work ... */
        return result;
    }
}

let foo = new Foo;
(() => {
    {
        let data = foo.data;
        if (data) {
            for (let item of data) {
                /* A */
            }
            return;
        }
    }

    /* B */
})();
```
or

```js
class Foo {
    get data() {
        let result = [];
        /* ... do some expensive work ... */
        return result;
    }
}

let foo = new Foo;
let __test = () => {
    let data = foo.data;
    if (!data)
        return false;

    for (let item of data) {
        /* A */
    }
    return true;
};
if (!__test()) {
    /* B */
}
```

but it wouldn't be _exactly_ the same due to the fact that the transpiled code would change what the last evaluated value in the outer scope would be, thereby changing the evaluation result of the entire program.

This is likely not that big of an issue, however, as this is probably pretty rare (most code tends to be written inside functions, which don't use the last evaluation result as the returned value) and any author/transpiler could just "fall back" to what's currently available (declare the variable outside the conditional).

The bigger issue would be if any of the code inside `A` (and/or `B`, depending on the transpilation approach) has a `return`, as that would need to be propagated outside the wrapper function, which may involve other workarounds (e.g. a `Symbol` could differentiate between a generated value and a transpiled "path").

## Future Work

This could be extended to allowing multiple names to be initialized (such as through destructuring) with a "normal" conditional to be written after the assignment with a `;` (similar to a `for`).

```js
if (let x = 1, y = 2; x || y) {
    /* ... */
}
```

```js
if (let {x, y} = data; x && y) {
    /* ... */
}
```
