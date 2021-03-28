+++
title = "Roll Your Own CSS-in-JS Library (2) - Dynamic Styles"
date = 2021-03-12T01:00:00Z
series = ["roll-your-own-css-in-js"]
+++

In our [last post]({{< relref "/posts/roll-your-own-css-in-js.md" >}})
we developed a simple CSS-in-JS library, but it was missing an important
feature - supporting dynamic styles. We will patch that in this post.

# End result

Dynamic styles are more useful when we are building components. In this
post, we will use React as our component library. Here's how our library
may be used in production:

<iframe src="https://codesandbox.io/embed/youthful-thunder-iio7y?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="youthful-thunder-iio7y"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

# Dynamic class names

This is the simplest approach that requires making the smallest change to
our little library. Our approach will be to generate the class name based on the
props being passed, then supply that to our react component. This means
that `Block` under the hood should look like this:

```typescript
const classNameGenerator = styleDef.dynamicStyle({
  border: "1px solid black",
  background: props.mode === "light" ? "white" : "black",
  fontSize: props.size === "big" ? "2em" : "1em",
});
const Block = (props: Props) => {
  return <div className={classNameGenerator(props)}>{props.children}</div>;
};
```

A simple implementation based on what we did previously may look like this

```typescript
class StyleDef {
  dynamicStyle<Props>(
    declaration: (props: Props) => CssLikeObject
  ): (props: Props) => string {
    return (props) => {
      return this.staticStyle(declaration(props));
    };
  }

  //...
}
```

Done? Well, this works, but there's a big issue here - every time
`staticStyle` is executed, a new css class name and a set of corresponding
rules are created! We can fix this by doing a bit of memoization.

```typescript
class StyleDef {
  dynamicStyle<Props>(
    declaration: (props: Props) => CssLikeObject
  ): (props: Props) => string {
    // Makes from css rule to class name
    const memo = new Map<string, string>();
    return (props) => {
      // JSON.stringify is not deterministic, but good enough
      // for our demo.
      const hash = JSON.stringify(props);
      if (memo.has(hash)) {
        return memo.get(hash)!;
      }
      const result = this.staticStyle(declaration(props));
      memo.set(hash, result);
      return result;
    };
  }
  //...other code
}
```

This fixes the performance issue we talked about above and is probably a
good enough MVP, but there are still other issues.

# Issues

## Unbounded number of class names

In our toy example there can only be 4 possible sets of styles. But it's easy to
construct something that can have an infinite number of possible props:

```typescript
styleDef.dynamicStyle((left: number) => {
  left;
});
```

One solution is to use inline styles for these properties.

## No more descendant selectors

With `staticStyle` we can do something like this

```typescript
const someClass = styleDef.staticStyle({
  color: "green",
});

const otherClass = styleDef.staticStyle({
  ":hover": {
    [someClass]: {
      color: "red",
    },
  },
});
```

This is not trivial with `dynamicStyle`. One solution here is to use CSS
variables, like this:

```typescript
const someClass = styleDef.staticStyle({
  color: "var(--someClassColor)",
});

const otherClass = styleDef.dynamicStyles((props: { isRed: boolean }) => ({
  "--someClassColor": "green",
  ":hover": {
    "--someClassColor": props.isRed ? "red" : "blue",
  },
}));
```

## Generating CSS code during build time is hard

Unless we can iterate through all possible values of `props` at build time,
we won't be able to generate all CSS rules. We can also settle with some
'default props' and just generate that single one, but that's not ideal.

## Large generated stylesheet

To some extend this can be an issue with our previous static-only library as
well, but having dynamic styles makes the issue more pronounced. In our
example, even though `border` stays the same, each new class will repeat
this property. We can split out the static styles into its own
`staticStyle` call, but that's a bit awkward.

One solution here is to use the atomic css pattern, detailed below.

# Optimization - Atomic CSS

What we are doing doesn't exactly match
[atomic css](https://css-tricks.com/lets-define-exactly-atomic-css/),
but the end result is similar.

The apporach is to programmatically break down each 'class' into
multiple classes, each with only one css property. Then, we will similarly
memoize the class names for each rules. This way, if a CSS property-value
is repeated across multiple style definitions, all of those styles will
share the same class name.

For example, suppose we render our example `Block` twice, once with
`{ mode: "light"; size: "big" }` and once with
`{ mode: "dark"; size: "big" }`. This will create the following css rules:

```css
.c1 {
  border: 1px solid black;
}

.c2 {
  background: white;
}

.c3 {
  font-size: 2em;
}

.c4 {
  background: black;
}
```

We will then pass to the first call the following classnames:
`"c1 c2 c3"`, and to the second call: `"c1 c4 c3"`.

## Implementation

Firstly, we will write a helper to split out each style declaration into
multiple atomic declarations.

```typescript
function isolateDeclaration(
  declaration: CssLikeObject,
  prefix: string[] = []
): CssLikeObject[] {
  return Object.keys(declaration)
    .sort()
    .flatMap((key) => {
      const value = (<any>declaration)[key];

      if (value instanceof Object) {
        return isolateDeclaration((<any>declaration)[key], [...prefix, key]);
      } else {
        // Expand the prefix array, e.g. if prefix = ['a', 'b'],
        // we return {a: {b: declaration}}
        return prefix.reduceRight(
          (result, cur) => {
            return {
              [cur]: result,
            };
          },
          {
            [key]: declaration[key],
          }
        );
      }
    });
}
```

And example output:

```
> isolateDeclaration({
...     ":hover": {
.....         color: "green"
.....     },
...     color: "red"
... })
[ { ':hover': { color: 'green' } }, { color: 'red' } ]
```

Next we will update our `staticStyle` and `dynamicStyle` method.

```typescript
class StyleDef {
  #memo = new Map<string, string>();
  staticStyle(declaration: CssLikeObject): string[] {
    return isolateDeclaration(declaration).map((d) => {
      const hash = JSON.stringify(d);
      if (this.#memo.has(hash)) {
        return this.#memo.get(hash);
      }
      const selector = this.#generateSelector();
      const ruleStrings = nestedDeclarationToRuleStrings(selector, declaration);
      ruleStrings.forEach((ruleString) =>
        this.#sheet.insertRule(ruleString, this.#sheet.cssRules.length)
      );
      this.#memo.set(hash, selector);
      return selector;
    });
  }

  dynamicStyle<Props>(
    declaration: (props: Props) => CssLikeObject
  ): (props: Props) => string[] {
    return (props) => {
      return this.staticStyle(declaration(props));
    };
  }
  //...
}
```

We updated the function interface to return an array instead of a single string, because
we will now most likely return multiple class names. Moreover, we move the memoization
into the class state since we want each staticStyle call to share the memoization.

Interestingly, we also reverted our optimization for `dynamicStyle` since memoization
is now done at the `staticStyle` level, so we no longer need to worry about
creating a new rule per call.

## React factory

Finally let's wrap this up by implementing what we have promised at the start of this
post - a React factory to create styled components.

```typescript
const styleDef = new StyleDef("somePrefix");

function styled<T extends {}>(
  declaration: (props: T) => CssLikeObject
) {
  const dynamicFactory = styleDef.dynamicStyle(declaration);
  const result: any = React.forwardRef((props: T, ref) => {
      return React.createElement("div", {
          ...props,
          className: [...dynamicFactory(props), (<any>props).className ?? ""].join(" ")
          ref
      });
  });
  return result;
}
```

# Are dynamic styles really necessary?

Probably not. After all, without CSS-in-JS, the usual pattern is to have some
classes defined in css files, then reference them in the UI code. We can replicate
that easily by generating these classes with static styles. This allows for writing
CSS code in the same file/dependency tree as the JavaScript code, and the dynamic-ness
is handled separately by the JavaScript UI logic instead of encapsulated in the
CSS-in-JS magic. That being said, dynamic styles does provide convenience and some
sort of elegance (depending on who you ask).

# Other ideas

We haven't really dealt with the other issues mentioned above other than the issue of
generating a large stylesheet. Our API for the react component is also a bit awkward:
if we intend the component to only have static styles, we have to pass something like
`() => CssLikeObject`. We can work around this by trying to allow both `(T) => CssLikeObject`
and `CssLikeObject`, but can we come up with something more elegant than that?

In our [next post]({{< relref "/posts/roll-your-own-css-in-js-3.md" >}}),
we will resolve all of these issues by switching to a different approach -
using CSS variables.
