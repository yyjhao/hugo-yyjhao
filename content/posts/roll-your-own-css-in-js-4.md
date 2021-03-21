+++
title = "Roll Your Own CSS-in-JS Library (4) - Static Extraction"
date = 2021-03-15T01:00:00Z
+++

# Static extraction is HARD

Firstly, the set-up of static extraction is more complicated - there
needs to be a build process, and the static extractor needs to fit
into that build process.

More importantly, static extraction will require some form of static
evaluation. Take the following code for example:

```typescript
import getColor from "helpers";

const Block = styled({
  color: getColor("block"),
});
```

In order to statically generate the css styles, the extractor needs to look
at at least one other file called "helpers". In this case, it may also need to
understand what "helpers" is pointing to - is it to some top level file called
"helpers", or a module? Moreover, "helpers" may itself import `getColor` from
yet another file. Even worse, Due to the flexibility of JavaScript,
`getColor` may also be defined in many interesting ways.

A perfect extractor will need to somehow evaluate all of these code and generate
the class name as well as the CSS code.

At the same time, we should only extract the minimal amount of code necessary,
both due to performance concerns and the general desire to avoid side effects
during a build process.

This is really hard to do right. [Facebook prepack](https://prepack.io) may
offer a glimpse of hope, but it's still very early in the development stage
and the development seems to have stalled as of March 2021.

For now, most build-time CSS-in-JS libraries impose some kind of restrictions to
make things easier, and most non-build-time CSS-in-JS libraries avoid building
any static extraction at all to stay flexible (see arguments from
[styled-component](https://github.com/styled-components/styled-components/issues/1018)
and [emotion](https://github.com/emotion-js/emotion/blob/master/docs/extract-static.mdx)).

Nevertheless, let's give it a try and build a simple static extractor for our
little library.

# Building a simple static extractor

In our last post we have built a dynamic CSS-in-JS library powered by CSS variables.
We will create a script that generate CSS code and replaces the call to `styled` with
some static class names.

To keep things simple, we will eschew a lot of important features like source
mapping and integration with build tools. We will also use JavaScript this time
to make it easier to run in the terminal.

## Basic idea

We will impose some restrictions here. In particular:

1. All expressions in the function parameter of `styled` must be static.
   - This means that all static values needs to be inline constants.
2. We will replace all function calls with `styled()`, i.e. `styled` doesn't need
   to be imported, but developers also can't redefined `styled` to be something else

Then we will build a simple script that reads through only a _single_ js file,
extracts these definitions, replaces them with a definition with some constant
class name and generates a corresponding stylesheet.

Since we are dealing with a single JS file at a time, our implementation can be
encapsulated in the following function:

```javascript
const process = (fileContent) => {
  //...
  return {
    jsCode: "updated js code",
    cssCode: "generated css code",
  };
};
```

## Parsing JavaScript code

Our first step is to parse the source code into some
[AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree) so that
we can meaningfully analyze it. To do so, we will use
[babel](https://babeljs.io).

```javascript
const babelParser = require("@babel/parser");
const process = (fileContent) => {
  const ast = babelParser.parse(fileContent, {
    sourceType: "module",
    plugins: ["jsx"],
  });
};
```

That's pretty easy. We specify `sourceType: "module"`, because we want to support
generate `import` and `export` statement. `jsx` is also necessary to support the JSX
syntax.

At this point, it may be useful to print out the AST to get a sense of what it looks like.

<iframe style = "padding: 0; margin: 0; background: transparent; width: 100%;" src = "https://runkit.com/e/oembed-iframe?target=%2Fusers%2Fyyjhao%2Frepositories%2F604eef40c321c8001a58529f%2Fdefault&referrer=" height = "500" frameborder = "0" allowfullscreen></iframe>

Notice that the AST consists of 'nodes', each has a `type` attribute that describes what
type of node it is. Each node can nest other nodes. For example, the `VariableDeclaration`
node has a `declarations` list, where some elements can be `VariableDeclarator`s, that specify
the `id` (e.g. `x` in `const x =`), and the `init` part of the declarator (e.g. the expression
to call a function).

## Identify valid `styled` calls

Next, we want to traverse the AST, look for our `styled()` call, generate some CSS code
based on that, and replace the `styled()` call with something that only reference the
class name. To do so, we use the `traverse` function from `babel`. This function follows
the [visitor pattern](https://en.wikipedia.org/wiki/Visitor_pattern).

We will call `traverse` with the AST with a 'visitor', and the traverse function walks
through the AST and calls the proper functions in the visitor when it encounters
specific nodes. In this case, we only care about the 'call expression', since we
want to know when `styled` is called.

```javascript
const babelParser = require("@babel/parser");
const { default: traverse } = require("@babel/traverse");
const process = (fileContent) => {
  const ast = babelParser.parse(fileContent, {
    sourceType: "module",
    plugins: ["jsx"],
  });
  traverse(ast, {
      CallExpression(path) {
        if (
          path.node.callee.name === "styled" &&
          path.node.arguments.length === 1 &&
          checkArgument(path.node.arguments[0])
        ) {
          // We know that this is something like styled({}).
        }
    });
};
```

Notice the `checkArgument` call - remember when we said we want to impose
some restriction? This is where we check against the first restriction -
all expressions must be static.

```javascript
function checkArgument(node) {
  // Make sure that the node is an object
  if (node.type === "ObjectExpression") {
    return node.properties.every((node) => {
      // The object has to consists of only key: value pairs,
      // i.e. no spread syntax
      return (
        node.type === "ObjectProperty" &&
        // The value can be either string or function
        (node.value.type === "StringLiteral" ||
          node.value.type === "ArrowFunctionExpression") &&
        // The key can be either an identifier (e.g. a: "b"),
        // or a string literal (e.g. "a": "b")
        (node.key.type === "Identifier" || p.key.type === "StringLiteral")
      );
    });
  }
}
```

## Statically evaluate each `styled` call

Now that we know that the call is valid, let's statically evaluate it. Our
restriction has made things much simpler - we can pretty much evaluate the
object as it is, with functions represented by some 'tokens'.

We will need to pull out our `nestedDeclarationToRuleStrings` function
again. Notice that in `nestedDeclarationToRuleStrings` we don't ever try to
evaluate the functions, so we can update the implementation and replace
functions with our AST nodes instead. We annotate the changes with comments below:

```javascript
function nestedDeclarationToRuleStrings(rootClassName, declaration) {
  const resultRules = [];
  const resultMapping = {};

  let count = 0;
  function _makeNewVariable(valueDef) {
    const variableName = `--${rootClassName}-${count}`;
    count++;
    resultMapping[variableName] = valueDef;

    return variableName;
  }

  function _helper(selector, declaration) {
    const nestedNames = [];
    const cssProps = {};

    for (let prop in declaration) {
      const value = declaration[prop];

      // We use to check if `typeof value !== "Function"`, now we check
      // if value.type is defined. This is a bit hacky but should work
      // as long as <type> tag and a `type` css property don't exist.
      if (value instanceof Object && !value.type) {
        nestedNames.push(prop);
      } else {
        cssProps[prop] = value;
      }
    }

    const lines = [];
    lines.push(`${selector} {`);
    for (let prop in cssProps) {
      // Similarly We use to check if `typeof value !== "Function"`, now
      // we check if value.type is defined.
      if (cssProps[prop].type) {
        lines.push(`${prop}:var(${_makeNewVariable(cssProps[prop])});`);
      } else {
        lines.push(`${prop}:${cssProps[prop]};`);
      }
    }
    lines.push("}");

    resultRules.push(lines.join("\n"));

    nestedNames.forEach((nestedSelector) =>
      _helper(
        joinSelectors(selector, nestedSelector),
        declaration[nestedSelector]
      )
    );
  }
  _helper("." + rootClassName, declaration);
  return {
    rules: resultRules,
    // We use to return a function (props) => inline style, now
    // we just return a mapping from CSS variable names to the AST nodes
    // representing the original functions
    variablesValueMapping: resultMapping,
  };
}
```

Running this function should generate a list of CSS rules and a
map from the CSS variable to the AST nodes representing each function
in the original code.

```javascript
const process = (fileContent) => {
  const ast = babelParser.parse(fileContent, {
    sourceType: "module",
    plugins: ["jsx"],
  });
  let counter = 0;
  let cssCode = "";
  traverse(ast, {
    CallExpression(path) {
      if (
        path.node.callee.name === "styled" &&
        path.node.arguments.length === 1 &&
        checkArgument(path.node.arguments[0])
      ) {
        const className = "prefix-" + counter;
        counter++;
        const { rules, variablesValueMapping } = nestedDeclarationToRuleStrings(
          className,
          convertToDeclaration(path.node.arguments[0])
        );
        cssCode += rules + "\n";
        const code = `
React.forwardRef((props, ref) => {
  return React.createElement("div", {
    ...props,
    className: ["${className}", props.className ?? ""].join(" "),
    style: {
      ${
        // somehow generate the style code here
      }
    }
      ...props.style
    },
    ref
  });
})
`;
        // replace the call expression with our new code
        path.replaceWith(babelParser.parseExpression(code));
      }
    },
  });
};
```

Using the `nestedDeclarationToRuleStrings` function that we have just built,
coupled with our counter-base class name function and a simple string to collect
css code, we were able to statically generate the class name and the css rules
without actually running the JavaScript code in the string.

We then plug the statically evaluated class name into to the code string, which
is the same the implementation for `styled`, parse it into an AST and replace
the node representing the `styled` call with it.

## Generating the transformed JavaScript code

First we need to fill in the `style` attribute in our generated React code.
We will use `@babel/generator` to generate code from AST - think of it
as the inverse of `@babel/parser`.

The `variablesValueMapping` maps a variable name, which is a string, to
a node representing a function. We will first convert this node
to some code as a string, then insert the string to our generated React
code. We will do that for every key-value pair.

This is what it looks like:

```javascript
const { default: generate } = require("@babel/generator");
`
    style: {
${Object.keys(variablesValueMapping)
  .map((k) => {
    return `"${k}": (${generate(variablesValueMapping[k]).code})(props),`;
  })
  .join("\n")}
      ...props.style
    },
`;
```

Finally, the output JavaScript code can be generated by `generate(ast).code`.

## Complete source code

Below is the complete source code with an example JavaScript code input, copied
from our previous example. Click run to check out the output CSS and JS strings.

<iframe style="padding: 0; margin: 0; background: transparent; width: 100%" src="https://runkit.com/e/oembed-iframe?target=%2Fusers%2Fyyjhao%2Frepositories%2F604ef007f39763001a52835d" height="400" frameborder="0" allowfullscreen></iframe>

# Last words

While we simplified our extractor to make it almost not useful, our code generation
is probably on the more complicated side. Most tools will try to generate only
the class name and the list of variables to avoid being too specific to a framework.

Also, depending on where we specify the dynamic logics, we may not need CSS variables at all.
Facebook's StyleX is such an example. The API allows for only static styles with
'variants', and only generates class names. This makes static codegen simpler and
removes the potential overhead of CSS variables.
