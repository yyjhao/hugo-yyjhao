---
title: "Roll Your Own CSS-in-JS Library (2) - Dynamic Styles"
date: 2021-03-06T21:46:25-08:00
draft: true
---

In our last post we developed a simple CSS-in-JS library, but it was missing an important
feature - supporting dynamic styles. We will patch that in this post.

# What are dynamic styles

It's more interesting to talk about dynamic styles with some ideas of component. In this
post, we will use React as an example. Here's one example:

```typescript
interface Props {
  mode: "light" | "dark";
  size: "big" | "small";
}

const Block = styled((props: Props) => ({
  border: "1px solid black",
  background: props.mode === "light" ? "white" : "black",
  fontSize: props.size === "big" ? "2em" : "1em",
}));

ReactDOM.render(<Block mode="light" size="big" />, container);
```

We created a styled component with some dynamic styles based on the value of props.

# Dynamic Class Names

This is the simplest approach that requires making the smallest change to your little
library. We make it generate the classname based on the props being passed, then supply
that to our react component. This means that `Block` under the hood should look like this:

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
      this.staticStyle(declaration(props));
    };
  }

  //...other code
}
```

This works fine, but there's a big issue here - every time we call static style, we create
a new css class name and a set of corresponding rules! We can fix this by creating some kind
of hash map to avoid re-computation.

```typescript
class StyleDef {
  dynamicStyle<Props>(
    declaration: (props: Props) => CssLikeObject
  ): (props: Props) => string {
    const memo = new Map<string, string>;
    return (props) => {
      const hash = JSON.stringify(props);
      if (memo.has(key)) {
        return memo.get(key);
      }
      const result = this.staticStyle(declaration(props));
      memo.set(key, result);
      return result;
    };
  }
  //...other code
}
```

This fixes the performance issue we talked about above. However, there are still
other issues with this approach.

# Issues

## Unbounded number of class names

In our toy example there can only be 4 combinations of class names. But it's easy to
construct something that can have infinite number of possible props:

```typescript
styleDef.dynamicStyle((left: number) => {
  left;
});
```

The usual solution is to use inline styles for these kind of styles.

## No more descendant selectors

With `staticStyle` we can do something like this

```typescript
const someClass = styleDef.staticStyle({
  //...
});

const otherClass = styleDef.staticStyle({
  [someClass]: {
    //...
  },
});
```

This is not trivial with `dynamicStyle` - there can be multiple class names. We can
try to pre-generate all possible class names assuming that we disallow the unbounded
example above somehow, but that makes our API rather awkward because we need to supply
the values of all possible props.

## Static code gen is impossible

Similar to the issues above, without making the API awkward, we can't pre-generate all
classes at build time.

## Large generated stylesheet

To some extend this is an issue with our previous static-only code as well, but dynamic
styles makes the issue more pronounced. In our example, even though `border` stays the
same, each class will contain this same property. In practice, there may be a lot more
static properties in this kind of dynamic styled components.

# Optimization - atomic css

This is one direction we can go to solve some issues above.

# Other ideas

In our next post, we will explore a different idea altogether - tweaking the API interface
and using CSS variables.
