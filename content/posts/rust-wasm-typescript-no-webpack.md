---
title: "Set up rust + wasm + TypeScript without webpack"
date: 2021-04-25T01:00:00Z
tags: ["wasm", "typescript", "wasm-pack", "rust"]
---

# 1 Set up the rust code and wasm-pack

1. [install wasm-pack](https://rustwasm.github.io/wasm-pack/installer/)
2. Set up a new project with wasm-pack. Run

```bash
$ wasm-pack new hello-wasm
$ cd hello-wasm
```

3. Build with this command

```bash
$ wasm-pack build --target web
```

This should produce a `pkg/` directory which should contain some `d.ts` file,
a `.wasm` file and a `.js` file in ES6 module format.

# 2 Set up the TypeScript code

Now we should set up a JavaScript project.

1. Run `npm init` to create an npm project.
2. Run `npm i -D typescript` to install tsc
3. Run `npx tsc --init` to create a `tsconfig.json` file
4. For cleaner organization of our source code, let's say all ts files should be put in a specific `ts/` directory
5. Edit the `tsconfig.json` file. Here are the field values we should make sure to update

```json
{
  "module": "es2015", // must be es2015 or later
  "outDir": "./build" // other directory works too, but we need a outdir
}
```

For our little demo purpose, we will call the demo `greet()` function from the generated rust project.

Let's make a file `ts/entry.ts`, with the following content:

```typescript
import init, { greet } from "../pkg/hello_wasm.js";
init().then(function () {
  greet();
});
```

This mostly follows the [official guide](https://rustwasm.github.io/wasm-bindgen/examples/without-a-bundler.html).

It's a bit tricky here because TypeScript won't prevent you from calling `greet()` without calling `init()` first,
but doing so will lead to a crash. This is something to keep in mind, though it's not specific to our setup.

Next we verify that this builds:

```bash
$ npx tsc -p .
```

# 3 Set up a development iteration process

Now let's set up an actual html file.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Wasm Demo</title>
  </head>
  <body>
    <script type="module" src="/build/ts/entry.js"></script>
  </body>
</html>
```

Next, we can use any quick static server iterate on our code. I will use `live-server` here as an example:

Run

```bash
$ npm i -D live-server
$ npx live-server
```

On another terminal, run

```bash
$ npx tsc --watch -p .
```

Now making any changes in the typescript should trigger a rebuild and an autoreload.

Unfortunately there isn't a `watch` mode for `wasm-pack` yet as of April 2021 (see
[this issue](https://github.com/rustwasm/wasm-pack/issues/457) for some discussion).

# 4 Set up a production build process

Most of the important outputs should be in `/build` and `/pkg`. For deployment, it's
pretty easy to use any standard bundling tool to build a single bundle out of the
generated `entry.js` file. There are two other small details to take care of

1. The html file assumes the build layout - we can fix this by replacing the `src`
   attribute in the html file with `sed` or other fancy tools as part of our build process
2. The generated `.js` file from `wasm-pack` assumes that the `wasm` file will be adjacent.
   We can either ensure that this happens, or pass in the path to the `wasm` file when
   calling the `init` function.
