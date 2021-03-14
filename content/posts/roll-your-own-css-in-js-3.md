---
title: "Roll Your Own CSS-in-JS Library (3) - Using CSS Variables"
date: 2021-03-12T21:48:01-08:00
draft: true
---

In this post we will discard our changes in the
[previous post]({{< relref "/posts/roll-your-own-css-in-js-3" >}})
and switch to a new approach.

# End result

<iframe src="https://codesandbox.io/embed/gracious-worker-gl9vn?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="gracious-worker-gl9vn"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

# CSS variables

[CSS variables](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties)
are also known as CSS Custom Properties. At a very high level:

1. We can set some value on a variable using `--variable-name: value`. The
   variable name has to start with `--`, and the definition has to happen
   within some selector or as inline styles.
2. We can then read the value of a variable and supply it to some CSS
   property
   with the `var()` function, e.g. `color: var(--theme-color);`
3. CSS Variables will cascade into inner elements.

# Approach

We will update our CSS rule generation code to support dynamic properties
with CSS variables, we then set the value of each variable using inline
styles. This way, we will generate only a single class name per dynamic
style.

Take this styled component we built in the last post as an example:

```typescript
const Block = styled((props: Props) => ({
  border: "1px solid black",
  background: props.mode === "light" ? "white" : "black",
  fontSize: props.size === "big" ? "2em" : "1em",
}));
```

We should generate a CSS rule that looks like this

```css
.c1 {
  border: 1px solid black;
  background: var(--c1-background);
  font-size: var(--c1-font-size);
}
```

then we need to apply the following inline style:

```jsx
<div
  style={{
    "--c1-background": props.mode === "light" ? "white" : "black",
    "--c1-font-size": props.size === "big" ? "2em" : "1em",
  }}
></div>
```

## Update to `CssLikeObject`

With this approach, we need to know exactly what properties are dynamic. Our
previous API for dynamic styles doesn't work any more. Let's update our
`CssLikeObject` definition:

```typescript
type DynamicCssProperties<Props = {}> = {
  [k in keyof CSS.PropertiesHyphen]:
    | CSS.PropertiesHyphen[k]
    | ((props: Props) => CSS.PropertiesHyphen[k]);
};

type CssLikeObject<Props = {}> =
  | {
      [selector: string]: DynamicCssProperties<Props>;
    }
  | DynamicCssProperties<Props>;
```

With this new API, our styled component now looks like this:

```typescript
const Block = styled({
  border: "1px solid black",
  background: (props: Props) => (props.mode === "light" ? "white" : "black"),
  fontSize: (props: Props) => (props.size === "big" ? "2em" : "1em"),
});
```

## Rebuilding `nestedDeclarationToRuleStrings`

First, let's define the new behavior. `nestedDeclarationToRuleStrings`
should take in a `rootClassName` and a new `CssLikeObject` as before.
However, it's not sufficient to just output some CSS rules - we also
need to know what CSS variables are created and how those CSS variables
should be filled. We can do that with this new function interface:

```typescript
function nestedDeclarationToRuleStrings<Props = {}>(
  rootClassName: string,
  declaration: CssLikeObject<Props>
): {
  rules: string[];
  variablesValueMapping: (props: Props) => CSS.PropertiesHyphen;
};
```

In addition to the original rules string, we also provide a way to compute
the values of all variables that the rules string may contain. We will
place the `rules` strings into our dynamic stylesheets and the
`rootClassName` to the class name of our elements as before, but now we
also need to compute the values of variables and place them into the inline
styles of our elements.

Before doing that, let's update the implementation of
`nestedDeclarationToRuleStrings`

```typescript
function nestedDeclarationToRuleStrings<Props = {}>(
  rootClassName: string,
  declaration: CssLikeObject<Props>
): {
  rules: string[];
  variablesValueMapping: (props: Props) => CSS.PropertiesHyphen;
} {
  const resultRules: string[] = [];
  const resultMapping: {
    [s: string]: (props: Props) => string;
  } = {};

  // Make a new CSS variable and record the mapping
  let count = 0;
  function _makeNewVariable(valueDef: (props: Props) => string) {
    const variableName = `--${rootClassName}-${count}`;
    count++;
    resultMapping[variableName] = valueDef;

    return variableName;
  }

  function _helper(selector: string, declaration: CssLikeObject<Props>) {
    // We split the props into either nested dynamic css rules
    // or plain dynamic css props.
    const nestedNames: string[] = [];
    const cssProps: DynamicCssProperties<Props> = {};

    for (let prop in declaration) {
      const value = (<any>declaration)[prop];

      if (value instanceof Object && typeof value !== "function") {
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
      if (typeof (<any>cssProps)[prop] === "function") {
        // Since this is a function, we know that it should be represented
        // by css variables
        lines.push(`${prop}:var(${_makeNewVariable((<any>cssProps)[prop])});`);
      } else {
        lines.push(`${prop}:${(<any>cssProps)[prop]};`);
      }
    }
    lines.push("}");

    // Each string has to be a complete rule, not just a single
    // property
    resultRules.push(lines.join("\n"));

    // Go through all nested css rules, generate string css rules
    // and collect them
    nestedNames.forEach((nestedSelector) =>
      _helper(
        joinSelectors(selector, nestedSelector),
        (<any>declaration)[nestedSelector]
      )
    );
  }
  _helper("." + rootClassName, declaration);
  return {
    rules: resultRules,
    variablesValueMapping: (props) => {
      const result: { [s: string]: string } = {};
      Object.keys(resultMapping).forEach((k) => {
        result[k] = resultMapping[k](props);
      });
      return result;
    },
  };
}
```

Now we can update our `StyleDef` class. Notice that there's no longer a need
to differentiate static and dynamic styles, so we will remove the
`staticStyle` function and replace it with a more generic `style` function.
All other utilities remain the same.

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

  style<Props = {}>(
    declaration: CssLikeObject<Props>
  ): {
    className: string;
    variablesValueMapping: (props: Props) => CSS.PropertiesHyphen;
  } {
    const selector = this.#generateSelector();
    const { rules, variablesValueMapping } = nestedDeclarationToRuleStrings(
      selector,
      declaration
    );
    rules.forEach((ruleString) =>
      this.#sheet.insertRule(ruleString, this.#sheet.cssRules.length)
    );
    return {
      className: selector,
      variablesValueMapping,
    };
  }

  #generateSelector = () => {
    return `${this.#prefix}-${this.#selectorCounter++}`;
  };
}
```

Next let's create a similar `styled` function

```typescript
const styleDef = new StyleDef("somePrefix");

function styled<T extends {}>(
  declaration: CssLikeObject<T;
) {
  const {
     className,
     variablesValueMapping
  } = styleDef.style(declaration);
  const result: any = React.forwardRef((props: T, ref) => {
      return React.createElement("div", {
          ...props,
          className: [className, (<any>props).className ?? ""].join(" "),
          style: {
             ...variablesValueMapping(props),
             ...(<any>props).style
          },
          ref
      });
  });
  return result;
}
```

This is very similar to what we have before, except we now
