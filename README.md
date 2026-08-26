# Declarations in Conditionals

ECMAScript proposal for allowing variable declarations inside conditional statements (e.g. `if`/`while`).

Authors:
 - Devin Rousso

Stage: 1

[Candidate spec text](https://tc39.es/proposal-Declarations-in-Conditionals/) is available.

## Overview

When programming in C++, an extremely useful feature is to be able to declare a variable _and_ have it be evaluated inside a conditional:

```cpp
if (auto* ptr = getPtr()) {
    /* ... */
}

if (auto* ptr = getPtr(); ptr && ptr->value) {
    /* ... */
}
```

Adding this capability to JavaScript would be very useful for the following reasons:
 - avoid having to retype the variable name
 - finer-grain "control" over the visibility of the variable
 - allow authors to write performance-"safe" code without having to know the specific details of the code being called (see example below)

In the case of JavaScript, however, there should be some limitations:
 - only using `let`, `const`, `using`, and `await using`
 - only for `if` and `while`
 - only exposed in the `if` block (i.e. not in the `else`)
 - comma separated list, destructuring, etc. require a second expression after a `;` to be provided in order to clarify what exactly is being tested

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
if (const colorScheme = localStorage.getItem("color-scheme")) {
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
