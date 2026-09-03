# Declarations in Conditionals

ECMAScript proposal for allowing variable declarations inside conditional statements (e.g. `if`/`while`).

Authors:
 - Devin Rousso

Stage: 1

[Candidate spec text](https://tc39.es/proposal-Declarations-in-Conditionals/) is available.

## Overview

When programming in C++, an extremely useful feature is to be able to declare a variable inside a conditional before evaluating its condition:

```cpp
if (auto* ptr = getPtr(); ptr && ptr->value) {
    /* ... */
}
```

Adding this capability to JavaScript would be very useful for the following reasons:
 - avoid having to evaluate the initializer more than once
 - finer-grain "control" over the visibility of the variable
 - allow authors to write performance-"safe" code without having to know the specific details of the code being called (see example below)

In the case of JavaScript, however, there should be some limitations:
 - only using `let`, `const`, `using`, and `await using`
 - only for `if` and `while`
 - only exposed in the `if` block (i.e. not in the `else`)
 - declarations always require an explicitly written `;` followed by a second expression that specifies what is tested

## Limitations

In sloppy mode, the following is unfortunately already valid JavaScript

```js
var let = {};
var x = 0;
var y = [42];

if (let[x] = y) { // let[0] = 42
    /* ... */
}
```

where `let` is parsed as an identifier, so `let[x] = y` is a computed property assignment rather than a destructuring declaration.

If `if (let x = y)` were allowed to omit the second expression, developers might reasonably expect the form above to be its destructuring counterpart despite its existing different meaning.

```js
// Create `x` and assign it to `y` and then explicitly check `x` for truthiness.
if (let x = y; x) { /* ... */ }

// Create `x` and assign it to `y` and then implicitly check `x` for truthiness.
if (let x = y) { /* ... */ }

// Create `x` and assign it to `y[0]` and then explicitly check `x` for truthiness.
if (let [x] = y; x) { /* ... */ }

// Create a property on the variable `let` with name equal to `x` and value equal to `y`.
if (let [x] = y) { /* ... */ }
```

That expectation would be especially natural because, once an explicitly written `;` and a second expression are present, both `if (let x = y; x)` and `if (let [x] = y; x)` declare `x` and then test the following expression.

Requiring this form for every declaration avoids the mismatch.

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
if (let data = foo.data; data) {
    for (let item of data) {
        /* A */
    }
} else {
    /* B */
}
```

which evaluates `foo.data` only once while keeping `data` scoped to the first branch.

One could create another variable (e.g. `let data = foo.data;`), but that could potentially keep `foo.data` (via `data`) alive much longer than needed and would "pollute" the scope with an additional variable.

As another example, a non-`module` `<script>` needs an extra block to limit the scope of a temporary binding:

```html
<meta name="color-scheme" content="light dark">
<script>
{
    const colorScheme = localStorage.getItem("color-scheme");
    if (colorScheme) {
        document.querySelector('meta[name="color-scheme"]').content = colorScheme;
    }
}
</script>
```

which becomes unnecessary if we can move the declaration to the condition:

```html
<meta name="color-scheme" content="light dark">
<script>
if (const colorScheme = localStorage.getItem("color-scheme"); colorScheme) {
    document.querySelector('meta[name="color-scheme"]').content = colorScheme;
}
</script>
```

## Transpiler Support

This can be transpiled using blocks and a generated label:

```js
class Foo {
    get data() {
        let result = [];
        /* ... do some expensive work ... */
        return result;
    }
}

let foo = new Foo;
__if0: {
    {
        let data = foo.data;
        if (data) {
            {
                for (let item of data) {
                    /* A */
                }
            }

            break __if0;
        }
    }

    {
        /* B */
    }
}
```

Note that `__if0` represents a fresh label chosen by the transpiler so it cannot conflict with any label in the source.
