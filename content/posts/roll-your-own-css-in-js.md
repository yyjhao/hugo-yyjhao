+++
date = 2021-03-07T01:00:00Z
title = "Roll Your Own CSS-in-JS Library (1) - Introduction"
feature = "image/page-default.webp"
comments = true
+++

CSS-in-JS is an idea to tackle various issues with CSS by writing them in
JavaScript (or whatever else that can compile to JavaScript, such as TypeScript).
Some popular examples include libraries like [styled component](https://styled-components.com/),
[emotion](https://github.com/emotion-js/emotion) and [JSS](https://cssinjs.org/).

In this series, we will build a very simple CSS-in-JS library with minimal dependencies
and walk through the basic ideas. We will use TypeScript to make the API clearer.

_Credit: some code here is inspired by [nano-css](https://github.com/streamich/nano-css)._

# End Result

We will build a simple library that allows us to do this:

```typescript
const s = new StyleDef("ExampleTest");
// A fancy shorthand
const css = s.staticStyle.bind(s);

// No react for now so let's use DOM APIs
const element = document.createElement("div");
element.innerText = "test";
element.className = css({
  color: "blue",
  ":hover": {
    color: "green",
  },
});
document.body.append(element);
```

the result looks like this:

<div class="test-example">
</div>

<script>
var __classPrivateFieldSet = (this && this.__classPrivateFieldSet) || function (receiver, privateMap, value) {
    if (!privateMap.has(receiver)) {
        throw new TypeError("attempted to set private field on non-instance");
    }
    privateMap.set(receiver, value);
    return value;
};
var __classPrivateFieldGet = (this && this.__classPrivateFieldGet) || function (receiver, privateMap) {
    if (!privateMap.has(receiver)) {
        throw new TypeError("attempted to get private field on non-instance");
    }
    return privateMap.get(receiver);
};
var _selectorCounter, _sheet, _prefix, _generateSelector;
function joinSelectors(parentSelector, nestedSelector) {
    if (nestedSelector[0] === ":") {
        return parentSelector + nestedSelector;
    }
    else {
        return `${parentSelector} ${nestedSelector}`;
    }
}
function nestedDeclarationToRuleStrings(rootClassName, declaration) {
    const result = [];
    // We use a helper here to walk through the tree efficiently
    function _helper(selector, declaration, ruleStrings) {
        // We split the props into either nested css rules
        // or plain css props.
        const nestedNames = [];
        const cssProps = {};
        for (let prop in declaration) {
            const value = declaration[prop];
            if (value instanceof Object) {
                nestedNames.push(prop);
            }
            else {
                cssProps[prop] = value;
            }
        }
        const lines = [];
        // Collect all generated css rules.
        lines.push(`${selector} {`);
        for (let prop in cssProps) {
            // collect the top level css rules
            lines.push(`${prop}:${cssProps[prop]};`);
        }
        lines.push("}");
        ruleStrings.push(lines.join("\n"));
        // Go through all nested css rules, generate string css rules
        // and collect them
        nestedNames.forEach((nestedSelector) => _helper(joinSelectors(selector, nestedSelector), declaration[nestedSelector], ruleStrings));
    }
    _helper(rootClassName, declaration, result);
    return result;
}
class StyleDef {
    constructor(prefix) {
        _selectorCounter.set(this, 0);
        _sheet.set(this, void 0);
        _prefix.set(this, void 0);
        _generateSelector.set(this, () => {
            var _a;
            return `${__classPrivateFieldGet(this, _prefix)}-${__classPrivateFieldSet(this, _selectorCounter, (_a = +__classPrivateFieldGet(this, _selectorCounter)) + 1), _a}`;
        });
        const sheetElement = document.createElement("style");
        document.head.appendChild(sheetElement);
        __classPrivateFieldSet(this, _sheet, sheetElement.sheet);
        __classPrivateFieldSet(this, _prefix, prefix);
    }
    staticStyle(declaration) {
        const selector = __classPrivateFieldGet(this, _generateSelector).call(this);
        const ruleStrings = nestedDeclarationToRuleStrings("." + selector, declaration);
        ruleStrings.forEach((ruleString) => __classPrivateFieldGet(this, _sheet).insertRule(ruleString, __classPrivateFieldGet(this, _sheet).cssRules.length));
        return selector;
    }
}
_selectorCounter = new WeakMap(), _sheet = new WeakMap(), _prefix = new WeakMap(), _generateSelector = new WeakMap();

const s = new StyleDef("ExampleTest");
const element = document.createElement("div");
element.innerText = "test";
element.className = s.staticStyle({
    color: "blue",
    ":hover": {
        color: "green",
    },
});
document.querySelector(".test-example").appendChild(element);
</script>

We will expand this to support dynamic styles in the next post. For now, the
library is only limited to static styles.

# Using the CSSStyleSheet API

This is a relatively obscured DOM API that allows programmatic manipulation of CSS rules.
More details can be found on [mdn](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleSheet).

Here's a high level overview of how to use this API for our purpose:

Firstly, we can create a `<style>` tag:

```typescript
const sheetElement = document.createElement("style");
document.head.appendChild(sheetElement);
```

The DOM API allows us to access the stylesheet object representing the styles in
the `<style>` tag by `sheetElement.sheet`.

The stylesheet object allows us to manipulate the css rules, e.g. we can insert
a rule by:

```typescript
sheetElement.sheet.insertRule(ruleString, sheet.cssRules.length);
```

With this API, we can implement a CSS-in-JS library by:

1. Generate a class name, then
2. Attach the rules to that class name, then
3. Assign the class name to some DOM element

# Example implementation

## Defining our in-JS representation

Firstly, let's define the type of CSS objects. Here we want the ability to specify CSS rules
with a nestable object representation so that we can specify rules with selectors.
At the same time we want to be able to _not_ specify the root class name in the
object and have it generated uniquely and when needed.

```typescript
import type * as CSS from "csstype";

type CssLikeObject =
  | {
      [selector: string]: CSS.PropertiesHyphen;
    }
  | CSS.PropertiesHyphen;
```

We use the `csstype` library to set up the basic css types. `CssLikeObject` is either
some css representation or some nested CSS representation object.

Here's an example of what a `CssLikeObject` may look like:

```typescript
{
    "font-size": "10px",
    ":hover": {
        "font-size": "20px"
    },
    ".randomStuff": {
        "font-size": "30px"
    },
}
```

As you can see, the nestable representation allows us to declare richer CSS rules that
supports things like selectors and psudoelements.

## Convert from JS representation to CSS string

But wait, how do we convert from `CssLikeObject` to `string`? First, let's visualize how this
`CssLikeObject` can be turned into css.

Assuming that the generated class name is `.gen-class-1`, the final output should be:

```css
.gen-class-1 {
  font-size: 10px;
  color: green;
}
.gen-class-1:hover {
  font-size: 20px;
  color: red;
}
.gen-class-1 .randomStuff {
  font-size: 30px;
  color: blue;
}
```

We present an algorithm below to generate a set of CSS rules, each represented by
a string. Comments inline.

```typescript
function nestedDeclarationToRuleStrings(
  rootClassName: string,
  declaration: CssLikeObject | CSS.PropertiesHyphen
): string[] {
  const result: string[] = [];
  // We use a helper here to walk through the tree recursively
  function _helper(
    selector: string,
    declaration: CssLikeObject | CSS.PropertiesHyphen,
    ruleStrings: string[]
  ) {
    // We split the props into either nested css rules
    // or plain css props.
    const nestedNames: string[] = [];
    const cssProps: CSS.PropertiesHyphen = {};

    for (let prop in declaration) {
      const value = (<any>declaration)[prop];

      if (value instanceof Object) {
        nestedNames.push(prop);
      } else {
        (<any>cssProps)[prop] = value;
      }
    }

    const lines: string[] = [];
    // Collect all generated css rules.
    lines.push(`${selector} {`);
    for (let prop in cssProps) {
      // collect the top level css rules
      lines.push(`${prop}:${(<any>cssProps)[prop]};`);
    }
    lines.push("}");

    // Each string has to be a complete rule, not just a single
    // property
    ruleStrings.push(lines.join("\n"));

    // Go through all nested css rules, generate string css rules
    // and collect them
    nestedNames.forEach((nestedSelector) =>
      _helper(
        joinSelectors(selector, nestedSelector),
        (<any>declaration)[nestedSelector],
        ruleStrings
      )
    );
  }
  _helper(rootClassName, declaration, result);
  return result;
}
```

Notice that there's `joinSelector` function - this is because we need to treat states and
psudoelements differently from decendant selectors.

```typescript
function joinSelectors(parentSelector: string, nestedSelector: string) {
  if (nestedSelector[0] === ":") {
    return parentSelector + nestedSelector;
  } else {
    return `${parentSelector} ${nestedSelector}`;
  }
}
```

## Generating class names

We haven't really talked about generating class names yet. Here we are going to
go with a simple counter. In some cases a stable hash may be better, but for our
little toy example this is sufficient.

Having a counter introduces some states. Plus, we already introduce some states earlier
by creating a style element. So it's time to make a class for all these. We are going
to call this `StyleDef`.

```typescript
class StyleDef {
  #selectorCounter = 0;
  #sheet: CSSStyleSheet;
  #prefix: string;

  constructor(prefix: string) {
    const sheetElement = document.createElement("style");
    document.head.appendChild(sheetElement);
    this.#sheet = sheetElement.sheet!;
    this.#prefix = prefix;
  }

  staticStyle(declaration: CssLikeObject): string {
    const selector = this.#generateSelector();
    const ruleStrings = nestedDeclarationToRuleStrings(
      "." + selector,
      declaration
    );
    ruleStrings.forEach((ruleString) =>
      this.#sheet.insertRule(ruleString, this.#sheet.cssRules.length)
    );
    return selector;
  }

  #generateSelector = () => {
    return `${this.#prefix}-${this.#selectorCounter++}`;
  };
}
```

We also introduce a `prefix` string so that multiple `StyleDef`s won't create classes that
conflict with each other.

# Final source code

```typescript
import type * as CSS from "csstype";

type CssLikeObject =
  | {
      [selector: string]: CSS.PropertiesHyphen;
    }
  | CSS.PropertiesHyphen;
function joinSelectors(parentSelector: string, nestedSelector: string) {
  if (nestedSelector[0] === ":") {
    return parentSelector + nestedSelector;
  } else {
    return `${parentSelector} ${nestedSelector}`;
  }
}
function nestedDeclarationToRuleStrings(
  rootClassName: string,
  declaration: CssLikeObject | CSS.PropertiesHyphen
): string[] {
  const result: string[] = [];
  function _helper(
    selector: string,
    declaration: CssLikeObject | CSS.PropertiesHyphen,
    ruleStrings: string[]
  ) {
    const nestedNames: string[] = [];
    const cssProps: CSS.PropertiesHyphen = {};

    for (let prop in declaration) {
      const value = (<any>declaration)[prop];

      if (value instanceof Object) {
        nestedNames.push(prop);
      } else {
        (<any>cssProps)[prop] = value;
      }
    }

    const lines: string[] = [];
    lines.push(`${selector} {`);
    for (let prop in cssProps) {
      lines.push(`${prop}:${(<any>cssProps)[prop]};`);
    }
    lines.push("}");

    ruleStrings.push(lines.join("\n"));

    nestedNames.forEach((nestedSelector) =>
      _helper(
        joinSelectors(selector, nestedSelector),
        (<any>declaration)[nestedSelector],
        ruleStrings
      )
    );
  }
  _helper(rootClassName, declaration, result);
  return result;
}
class StyleDef {
  #selectorCounter = 0;
  #sheet: CSSStyleSheet;
  #prefix: string;

  constructor(prefix: string) {
    const sheetElement = document.createElement("style");
    document.head.appendChild(sheetElement);
    this.#sheet = sheetElement.sheet!;
    this.#prefix = prefix;
  }

  staticStyle(declaration: CssLikeObject): string {
    const selector = this.#generateSelector();
    const ruleString = nestedDeclarationToRuleStrings(
      "." + selector,
      declaration
    );
    ruleStrings.forEach((ruleString) =>
      this.#sheet.insertRule(ruleString, this.#sheet.cssRules.length)
    );
    return selector;
  }

  #generateSelector = () => {
    return `${this.#prefix}-${this.#selectorCounter++}`;
  };
}
```

# Issues with CSS-in-JS in general

We are done! Now that we have built a small library, it's time to talk about some issues
with CSS-in-JS frameworks.

## Performance

We can notice a few issues with our little library:

1. Instead of downloading and potentially parsing JS and CSS file in parallel, style code now needs to be
   parsed with JS code in series.
2. There's some overhead in first _generating_ CSS strings through JS, before having them parsed by the browser,
   compare to just giving the browser some raw CSS files.

The point is that there's always some overhead in using CSS-in-JS compared to loading
CSS files directly. For a small to medium project, this is likely not a problem, and
the ease of development will likely outweigh the performance cost. For any massive
projects, this can mean a large performance hit.

## Build step to the rescue?

One solution here is to use CSS-in-JS during dev, but compiles them into CSS for build.
However, this means that there is now a build step that needs maintaining
and takes time. This build step is also potentially more complex than other css preprocessors
like SCSS, because it needs to read JS/TS code, which has their own dependency trees and
then emit CSS code, then replace some of the JS code output with the emitted CSS class
name.

Additionally, using a build script usually means that dynamic styles has to be supported
by CSS variables, which introduce its own limitation. The now deprecated
[emotion static extrator](https://github.com/emotion-js/emotion/blob/master/docs/extract-static.mdx)
and [linaria](https://github.com/callstack/linaria) are two examples.

## Ease of debugging

It used to be that CSSStyleSheetAPI takes away our ability to edit style rules directly in browser
inspectors. This has since been fixed in Chrome and Firefox, though it's still an issue in
["the new IE"](https://www.safari-is-the-new-ie.com/).

# How about Inline styles?

This is one other approach in the CSS-in-JS field. However, it has two issues:

1. A lot of CSS features aren't available - no pseduelements, selectors, media
   queries, states etc.
2. Every single element defines its own styles - this can lead to some concerns
   on performance due to a larger DOM representation per node and likely more DOM mutations

On the other hand, based on what we have just discussed, inline styles also has
some perf advantage over our approach - we get to directly manipulate styles by
modifying attributes of `element.style` rather than getting the browser to parse
our CSS strings, though some parsing still has to be done as the browser needs to
parse the property values.

[Radium](https://formidable.com/open-source/radium/) is an example that uses this approach and
works around 1 by listening on DOM events to trigger special state or media queries.

# ES6 tagged templates

A lot of the popular CSS-in-JS libraries allows for syntax like this

```typescript
const Button = styled.a`
  color: blue;
  ${(props) =>
    props.primary &&
    css`
      color: green;
    `}
`;
```

This is done with
[ES6 tagged templates](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#tagged_templates),
which we will explore in a future post.

# Things to look forward to

Why on earth can't CSSStyleSheet support an API like inline styles, where we can set properties
directly instead of having to pass raw css strings around? Well,
[CSS Typed Object Model](https://developer.mozilla.org/en-US/docs/Web/API/StylePropertyMap/set)
does exactly that. However, it's still in experimental stage, and only Chrome and its bastard child
Edge support it as of now.
