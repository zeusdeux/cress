# CRESS

**CRESS** stands for **Componentized Reactive ...Erm... Scoped Stylesheets**. Think of **CRESS** components as web components for styling, except without having to use the web components specs.

* [Features](#features)
* [The `this` keyword... in CSS??](#the-this-keyword-in-CSS)
* [Props](#props)
* [Reactivization](#reactivization)
* [Container queries too??](#container-queries-too)
* [What about Shadow DOM? Isn't that what everyone wants?](#what-about-shadow-dom-isnt-that-what-everyone-wants)
* [Config API](#config-api)

## Features

* Completely dependency free
* Completely framework independent
* <1KB minified ES module
* Automatic scoping
* SSR (Server-side Rendering) is easy
* Reactive to prop changes using `MutationObserver`
* `resize` method provided harnessing `ResizeObserver`

## The `this` keyword... in CSS??

Well, no, not really. The word ”this” in a string that looks like CSS is more like it. But it helps solve the scoping issue without having to worry about the CSS OM or having to use some sort of complex regular expression. Instead, you just write 'this' (or a different word you've chosen; it's configurable) wherever you need to represent the target/parent element in the selector.

```css
this button {
  color: red;
}
```

In the constructor ”this” is replaced globally by a special identifier, and that's how styles are scoped:

```css
[data-i="cress-wi5xdkbb9"] button {
  color: red;
}
```

## Props

When you create a new **CRESS** component, you can set some default props and interpolate these into the `css` method:

```js
new Cress('.test', {
  props: {
    color: 'red'
  },
  css(props) {
    return `
      this button {
        color: ${props.color};
      }
    `
  }
});
```

The neat thing is that every element matching `.test` will now share an embedded stylesheet identified by `cress-wi5xdkbb9-red`. The `wi5xdkbb9` part is shared by _all_ `.test` elements, and the `red` part is shared by all `.test` elements _where the `color` prop is `red`_. This saves on redundancy.

Now... if you apply the `data-color="blue"` attribution to one or more of your `.test` elements, their identifier will become `cress-wi5xdkbb9-blue` and a new stylesheet will be created to serve them differently. 

Every default prop can be overridden with an attribute of the pattern `data-[name of prop]`. You don't need to use props (you may just want the scoping feature) but they're v nice. 

## Reactivization

A `MutationObserver` is applied to each of the `.test` elements, so when a prop-specific attribute is changed the new styles are applied. That's the reactive part, and makes **CRESS** components a bit like custom elements. However, they are much easier to SSR: custom elements are [not currently supported by JSDOM, for instance](https://github.com/jsdom/jsdom/issues/1030).

`MutationObserver` is [nearly universally supported](https://caniuse.com/#feat=mutationobserver); [Custom Elements are not nearly universally supported](https://caniuse.com/#feat=mutationobserver&search=custom%20elements).

## Container queries too??

**CRESS** components are initialized with two arguments: a CSS selector, and a config object. As well as the default `props` object and the `css()` method, you can add and adjust some other things too. The `resize()` method lets you do stuff when the (parent) element (the element that matches the selector) is resized. This allows you to create container queries!

Consider the following example:

```js
new Cress('.test', {
  props: {
    fontBig: '1.5rem',
    fontSmall: '0.85rem'
  },
  css(props) {
    return `
      this button {
        font-size: ${props.fontBig};
      }

      this.small button {
        font-size: ${props.fontSmall};
      }
    `
  },
  resize(observed) {
    observed.target.classList.toggle(
      'small',
      observed.contentRect.width < 400
    );
  }
});
```

Now, wherever `.test` elements are narrower than `400px`, they will adopt the `.small` class and the `props.fontSmall` text size. Goodness! (Note that `observed` represents the `ResizeObserver` entry for the `.test` node, and `observed.target` represents the actual node.)

`ResizeObserver` is [not the most well supported observer](https://caniuse.com/#search=resizeObserver) (although it has recently come to Firefox), so it's recommended `resize()` stuff is limited to progressive enhancements only, for production sites, for now.

## What about Shadow DOM? Isn't that what everyone wants?

Nah. Everyone thinks Shadow DOM will mean styles are completely sandboxed, like you're using an `<iframe>`. But they're not, which is cack. Plus they don't accept universal styles (set using `*`) when you _want_ them to.

The identifiers used in **CRESS** components handle styles not leaking _out_ to other elements (think [Vue's implementation](https://vue-loader.vuejs.org/guide/scoped-css.html)). To stop styles leaking _in_, you have the option of _reseting_ them with the `reset` config option:

```js
new Cress('.test', {
  reset: true, // ← added line
  props: {
    color: 'red'
  },
  css(props) {
    return `
      this button {
        color: ${props.color};
      }
    `
  }
});
```

All this does is prepend the parsed CSS with the following:

```css
[data-i="cress-wi5xdkbb9"],
[data-i="cress-wi5xdkbb9"] * { 
  all: initial 
} 
```

And that's all you need, probably.

## Getting started

**CRESS** is just a little ES module, exporting a constructor. You import it like so:

```js
import { Cress } from './path/to/Cress.js';
```

Then you can start creating **CRESS** components. Note the `new` keyword. Here's a very basic example (only the `css()` method of the config object (the second argument) is required):

```js
import { Cress } from './path/to/Cress.js';

new Cress('.my-selector', {
  css() {
    return `
      this {
        font-family: cursive;
      }
    `
  }
});
```

Of course, you can organize your **CRESS** components into their own module files if you want. This is what `MySelector.js` might look like:

```js
import { Cress } from './path/to/Cress.js';

const MyComponent = new Cress('.my-selector', {
  css() {
    return `
      this {
        font-family: cursive;
      }
    `
  }
});

export { MyComponent };
```

## Config API

The second argument to each constructor call in the config object. 

### `css()` (`Function`; required)

This is where you write your component's CSS. If you are using a [props](#props-object) property, you can pass them in using the function's/method's only argument:

```js
css(props) {
  return `
    this button {
      color: ${props.color};
    }
  `
}
```

## `this` (`String`)

By default, the word ”this” represents the current element/parent. You may wish to use a different identifier; perhaps one that is less likely to appear elsewhere in your CSS. You could just use a character you're sure your CSS won't use:

```js
this: '±'
```

Then you'd have to write your [`css` method](#css-function-required) something like this:

```js
css() {
  return `
    ± button {
      color: red;
    }
  `
}
```

### `resize()` (`Function`)

As described under [**Container queries too??**](#container-queries-too), you can use container queries with your **CRESS** components. The only argument to the `resize()` method is the `ResizeObserver` entry for the **CRESS** component node. See the following example for how to access some key properties.

```js
resize(observed) {
  console.log(observed.contentReact.width) // the width, in pixels, of the node
  console.log(observed.contentReact.height) // the height, in pixels, of the node
  console.log(observed.target) // the node itself
}
```

### `reset` (`Boolean`)

To remove all styles that would be inherited or otherwise applied to nodes and their children, just use:

```js
reset: true
```

As described in [**What about Shadow DOM? Isn't that what everyone wants?**](#what-about-shadow-dom-isnt-that-what-everyone-wants), this uses `all: initial`.

