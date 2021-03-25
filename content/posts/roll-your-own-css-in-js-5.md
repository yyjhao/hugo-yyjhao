+++
title = "Roll Your Own CSS-in-JS Library (5) - Tagged Templates"
date = 2021-03-24T01:00:00Z
+++

In this post, we will explore a different CSS-in-JS syntax.

# CSS-in-JS but using CSS syntax

End result:

<iframe src="https://codesandbox.io/embed/elastic-mendeleev-ojy02?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="elastic-mendeleev-ojy02"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

Notice that we get to write CSS similar to template strings rather than
as objects.

This has a few advantages:

1. Our CSS code now looks closer to vanilla CSS.
2. We can sequence CSS properties.
3. We can duplicate CSS properties to support fallback behavior.

In practice, 1 is probably the strongest driver for adopting
this syntax, as typically there's no benefit in sequencing CSS
properties except for fallback behavior. However, even with the
object syntax, one can support fallback behavior by adding a
post-processor to the rule generation pipeline.

# What is <code>styled``</code>

## ES6 tagged template

[mdn](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#tagged_templates)
has more information. We will provide a quick summary here.

ES6 introduces a new string interpolation syntax aside from template srings.
The "\`" can be immediately preceded by some function name. For example
<code>styled\`abc ${someVar} de ${someMoreVar} fg\`</code>. This is
treated similar to `styled(["abc ", " de ", " fg"], someVar, someMoreVar)`.
Note that `styled` need not return a string.

As a quick example to get us more familiar with this syntax, let's
implement this using a similar approach in our
[second post]({{< relref "/posts/roll-your-own-css-in-js-2.md" >}}).

```typescript
function styled(strings: string, ...expressions: string[]) {
    const dynamicFactory = styleDef.css(strings, ...expressions);
    const result: any = React.forwardRef((props: T, ref) => {
        return React.createElement("div", {
            ...props,
            className: [...css(props), (<any>props).className ?? ""].join(" ")
            ref
        });
    });
    return result;
}
```

Notice that we delegated the class name creation to a different function
similar to our previous approach. Let's implement our new `StyleDef`. This
will end up pretty similar to what we have before, except that instead of
generating stylesheets from an object, we can concatenate the strings
instead.

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

  style<Props>(
    strings: string[],
    ...expressions: (props: Props) => string[]
  ): (props: Props) => string {
    const memo = new Map<string, string>();
    return (props) => {
      const cssBody = strings
        .reduce((accu, cur, index) => {
          accu.push(cur);
          if (index < expressions.length) {
            accu.push(expressions[index](props));
          }
        }, [])
        .join("");
      if (memo.has(cssBody)) {
        return memo.get(cssBody)!;
      }
      const selector = this.#generateSelector();
      memo.set(cssBody, selector);
      const ruleString = `.${selector}: ${cssBody}`;
      this.#sheet.insertRule(ruleString, this.#sheet.cssRules.length);
      return selector;
    };
  }

  #generateSelector = () => {
    return `${this.#prefix}-${this.#selectorCounter++}`;
  };
}
```

Notice that this simple approach is missing an important feature -
nested CSS selectors. To support this, we will have to implement some kind
parsers, which we will do in the next section.

# CSS variables again?

We may also want to consider implementing this with a similar approach that
uses CSS variables like in our
[third post]({{< relref "/posts/roll-your-own-css-in-js-3.md" >}}).
This is actually a lot harder. Consider the following code snippet:

```typescript
const Block = styled`
    border: 1px solid black;
    color: ${(props) => props.color};
    height: ${(props) => props.height}px;
    ${(props) => (props.hasWidth ? "width: 100px;" : "")}
`;
```

If we simply replace the functions with the variables like what we did before, it will
only work for the color property. `height` will look like this

```
height: var(--some-var)px;
```

which is not valid.

We can fix the issue with `height` if we can somehow detect that
there's some residue string before and after a dynamic css value, and
update the function to include it. In the above example, we will transform
`${props => props.height}px` into `${props => props.height + "px"}`.

However, `width` simply won't work at all because a css variable can't be
a property-value pair. We have to disallow patterns like this.

Since we plan to handle nested style, let's write a parser and fix both
issues along the way.

## High level approach

We will first parse the template strings interpolated with functions into an
AST. Then, we can walk through the AST, collect CSS rules and convert the
nested selectors into proper CSS selectors. While we are collecting CSS
rules, we can look at strings before and after functions and note them as
'partial CSS values', which allows us to transform the function as we
discussed above.

## Defining the syntax

With the introduction of nested selectors, we will need to update our 'CSS' syntax
a bit. We will allow not just CSS declaration (`property: value;`) in a block (wrapped
by `{` and `}`), we will also allow selectors which can open their own block.
i.e. the following is valid

```scss
.classname {
  height: 1px;
  .descendant {
    height: 1px;
    .descendant-2 {
      height: 1px;
    }
  }
}
```

Note that when generating CSS rules we will always wrap the definition in some class
name so our underlying syntax doesn't need to allow declaration in the global scope,
i.e. the following is not valid to our parser

```
height: 1px;
```

... because we will convert it to

```css
.generated-class {
  height: 1px;
}
```

Our AST then should consist of the follow nodes:

1. nestable selector node
2. declaration (property-value pair)

...where a nestable selector node can contain other nestable selector nodes or declarations.

With the addition of string templates, it's more interesting. Now we need to split out the
declaration node, because it be either

1. property, followed by value, or
2. property, followed a partial value start, then a function, then a partial value end.

## Visitor pattern

Recall the visitor pattern we touched on in the previous post. Based on our discussion above,
we can define a visitor function like so:

```typescript
type VisitorState<T> =
  | {
      type:
        | "partial_value_start"
        | "partial_value_end"
        | "nest"
        | "property"
        | "value";
      value: string;
    }
  | {
      type: "close";
    }
  | {
      type: "dynamic_function";
      value: (props: T) => string;
    };

interface NestedCssTemplateVisitor<T, Output> {
  onVisit(state: VisitorState<T>): void;
  done(): Output;
}
```

Note the addition of `"close"` which is necessary to represent that we have exited a nested selector.

## Parsing and visiting

Note that the visitor pattern doesn't require us to supply a tree directly. We can also parse the
string and call the visitor along the way. Following this, we will define 2 functions. The main
one is `walkNestedCssInJs`, which will take the ES6 tagged template and a visitor, then parse
the the former and call the visitor accordingly.

The other one is `advanceNestedCssInJs` which parses some string from the tagged template
and extracts the relevant information. We use the word `advance` here because it's advancing
our state machine - for example, if we are in the `nest` state, we expect to see either
`property` or `nest` next, and if we are in the `property` state, we expect to see `value` next.

```typescript
// We can define this based on VisitorState, but
// we spell it out here for clarity
type ParseState =
  | "partial_value_start"
  | "partial_value_end"
  | "nest"
  | "close"
  | "property"
  | "value";

function advanceNestedCssInJs(
  currentState: WalkState,
  cssInJsString: string
): {
  nextState: WalkState;
  value: string;
} {
  // we will work on this later
}

function walkNestedCssInJs<T, Output>({
  initialState,
  strings,
  expressions,
  visitor,
}: {
  // Require an initial state here so that we can call
  // the visitor to supply a root selector before calling
  // this.
  initialState: ParseState;
  strings: TemplateStringsArray;
  expressions: ((props: T) => string)[];
  visitor: NestedCssTemplateVisitor<T, Output>;
}) {
  let state: ParseState = initialState;
  for (let i = 0; i < strings.length; i++) {
    const s = strings[i];
    let currentState: WalkState = {
      state,
      index: 0,
    };
    while (true) {
      const result = advanceNestedCssInJs(currentState, s);
      currentState = result.nextState;
      visitor.onVisit({
        type: currentState.state,
        value: result.value,
      });
      if (currentState.index === s.length) {
        break;
      }
    }
    state = currentState.state;
    if (i !== strings.length - 1) {
      visitor.onVisit({
        type: "dynamic_function",
        value: expressions[i],
      });
    }
  }
}
```

Next let's implement `advanceNestedCssInJs`. To do so, we will first need to define a few regexes.

Here's our quick and hacky regexes for the various states:

- `close`: `/(})/`
- `property`: `/([a-z-]+):/`
- `value`: `/[^"'{};\s]*(?:["'].*["'])*(?:url\(.*\))*[^"{}';]*;/`
- `selector`: `/([^;{}"\s()]+[^;{}"(]*(?:(?:not)|(?:.*-[a-z]+)\([^;{}"(]*)*)[\s]*{/`

However, due to the existence of 'partial values', it's possible that the regex may not match. As
such, we split out the detection into 2 states - when we have advance through a `property`, we know
that the next token must be a `value` or `partial_value_start`. If the regex fail to match `value`
we can assume that the token is `partial_value_start`.

This lead us to the following algorithm:

```typescript
const reSelectorOrPropertyOrClose = /^\s*(?:([^;{}"\s()]+[^;{}"(]*(?:(?:not)|(?:.*-[a-z]+)\([^;{}"(]*)*)[\s]*{)|(?:([a-z-]+):)|(})\s*/i;
const reValue = /^\s*([^"'{};\s]*(?:["'].*["'])*(?:url\(.*\))*[^"{}';]*;)\s*/i;
const rePartialValueEnd = /([^;]*);\s*/i;

function advanceNestedCssInJs(
  currentState: WalkState,
  cssInJsString: string
): {
  nextState: WalkState;
  value: string;
} {
  switch (currentState.state) {
    case "partial_value_start":
      const nextSemiColonMatch = rePartialValueEnd.exec(cssInJsString);
      if (!nextSemiColonMatch) {
        throw new Error("no match");
      }
      return {
        nextState: {
          state: "partial_value_end",
          index: nextSemiColonMatch.index + nextSemiColonMatch[0].length,
        },
        value: nextSemiColonMatch[1],
      };
    case "close":
    case "nest":
    case "value":
    case "partial_value_end": {
      const match = reSelectorOrPropertyOrClose.exec(
        cssInJsString.substring(currentState.index)
      );
      if (!match) {
        throw new Error("no match");
      }
      if (match[1]) {
        return {
          nextState: {
            state: "nest",
            index: match.index + match[0].length + currentState.index,
          },
          value: match[1],
        };
      } else if (match[2]) {
        return {
          nextState: {
            state: "property",
            index: match.index + match[0].length + currentState.index,
          },
          value: match[2],
        };
      } else if (match[3]) {
        return {
          nextState: {
            state: "close",
            index: match.index + match[0].length + currentState.index,
          },
          value: match[3],
        };
      } else {
        throw new Error("Unexpected");
      }
    }
    case "property": {
      reValue.lastIndex = currentState.index;
      const match = reValue.exec(cssInJsString.substring(currentState.index));
      if (!match) {
        return {
          nextState: {
            state: "partial_value_start",
            index: cssInJsString.length,
          },
          value: cssInJsString.substring(currentState.index).trim(),
        };
      }
      return {
        nextState: {
          state: "value",
          index: match.index + match[0].length + currentState.index,
        },
        value: match[1],
      };
    }
    default:
      throw new Error("unexpected");
  }
}
```

## Converting to flat CSS with CSS variables

Next we will implement a visitor that converts the nested template CSS into a flat,
standard CSS string.

To convert nested selectors, we maintain a stack as we walk through the tree.

To transform the dynamic function, we keep track of `partial_value_start` and
`partial_value_end` so that we can perform the following transformation:

```typescript
const transformed = (props: Props) => {
  return partialValueLeft + lastDynamicFunction(props) + partialValueRight;
};
```

Here's the full visitor source code:

```typescript
class NestedCssTemplateToFlatCssVisitor<Props>
  implements
    NestedCssTemplateVisitor<
      Props,
      {
        rules: string[];
        variablesValueMapping: (
          props: Props
        ) => {
          [variableName: string]: string;
        };
      }
    > {
  #ruleStack: Array<{ selector: string; declarations: string[] }> = [];
  #rules: string[] = [];
  #variablesFuncMapping: {
    [variableName: string]: (props: Props) => string;
  } = {};
  #partialValueLeft: string = "";
  #lastDynamicFunction: ((props: Props) => string) | null = null;

  constructor(public readonly generateVariableName: () => string) {}

  onVisit(state: VisitorState<Props>) {
    switch (state.type) {
      case "partial_value_start":
        this.#partialValueLeft = state.value;
        break;
      case "partial_value_end":
        const variableName = this.generateVariableName();
        const lastDynamicFunction = this.#lastDynamicFunction;
        this.#lastDynamicFunction = null;
        if (!lastDynamicFunction) {
          throw new Error("unexpected");
        }
        const partialValueLeft = this.#partialValueLeft;
        this.#partialValueLeft = "";
        const partialValueRight = state.value;
        this.#variablesFuncMapping[variableName] = (props: Props) => {
          return (
            partialValueLeft + lastDynamicFunction(props) + partialValueRight
          );
        };

        this.#currentRule().declarations.push(`var(${variableName});`);
        break;
      case "nest":
        const selector =
          this.#ruleStack.length === 0
            ? state.value
            : joinSelectors(this.#currentRule().selector, state.value);
        this.#ruleStack.push({
          selector,
          declarations: [],
        });
        break;
      case "close":
        const lastRule = this.#ruleStack.pop();
        if (!lastRule) {
          throw new Error("unexpected");
        }
        this.#rules.push(
          `${lastRule.selector} { ${lastRule.declarations.join(" ")} }`
        );
        break;
      case "property":
        this.#currentRule().declarations.push(`${state.value}:`);
        break;
      case "value":
        this.#currentRule().declarations.push(`${state.value};`);
        break;
      case "dynamic_function":
        this.#lastDynamicFunction = state.value;
        break;
      default:
        const _never: never = state;
    }
  }
  done() {
    return {
      rules: this.#rules,
      variablesValueMapping: (props: Props) => {
        const result: any = {};
        Object.keys(this.#variablesFuncMapping).forEach((v) => {
          result[v] = this.#variablesFuncMapping[v](props);
        });
        return result;
      },
    };
  }

  #currentRule = () => {
    if (this.#ruleStack.length === 0) {
      throw new Error("unexpected");
    }
    return this.#ruleStack[this.#ruleStack.length - 1];
  };
}
```

## Wrapping up

The last part here is again creating our `StyleDef` and `styled` function.

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
    strings: TemplateStringsArray,
    ...expressions: ((props: Props) => string)[]
  ): {
    className: string;
    variablesValueMapping: (props: Props) => CSS.PropertiesHyphen;
  } {
    const selector = this.#generateSelector();
    let counter = 0;
    const visitor = new NestedCssTemplateToFlatCssVisitor(
      () => `--${selector}-${counter++}`
    );

    visitor.onVisit({
      type: "nest",
      value: `.${selector}`,
    });
    walkNestedCssInJs({
      initialState: "nest",
      strings,
      expressions,
      visitor,
    });
    visitor.onVisit({
      type: "close",
    });
    const { rules, variablesValueMapping } = visitor.done();

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

const styleDef = new StyleDef("prefix");

export function styled<Props extends {}>(
  strings: TemplateStringsArray,
  ...expressions: ((props: Props) => string)[]
) {
  const { className, variablesValueMapping } = styleDef.style(
    strings,
    ...expressions
  );
  const result: any = React.forwardRef((props: Props, ref) => {
    return React.createElement("div", {
      ...props,
      className: [className, (<any>props).className ?? ""].join(" "),
      style: {
        ...variablesValueMapping(props),
        ...(<any>props).style,
      },
      ref,
    });
  });
  return result;
}
```

# Build

One issue with this syntax is that unless we can ditch the advance syntax
like nested CSS, we are incurring even more overhead due to the parser.
However, this is the kind of work that's perfect to be moved to build time.

The approach to convert the code in post into a build-time compiler is
very similar to what we have in the
[previous post]({{< relref "/posts/roll-your-own-css-in-js-4.md" >}}),
so we won't go though it again.
