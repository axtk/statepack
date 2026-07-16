# statepod

Vanilla TS/JS state management for sharing data across decoupled parts of the code and routing. Routing is essentially shared state management, too, with the shared data being the URL.

This package exposes the following classes:

```
EventEmitter ──► State ──► PersistentState
                    │
                    └────► URLState ──► Route
```

Roughly, their purpose boils down to the following:

- `EventEmitter` is for triggering actions without tightly coupling the interacting components
- `State` is `EventEmitter` that stores data and emits an event when the data gets updated, it's for dynamic data sharing without tight coupling
- `PersistentState` is `State` that syncs its data to the browser storage and restores it on page reload
- `URLState` is `State` that stores the URL + syncs with the browser's URL in a SPA fashion
- `Route` is `URLState` + native-like APIs for SPA navigation and an API for URL matching

Contents: [State](#state) · [PersistentState](#persistentstate) · [Route](#route) · [Annotated examples](#annotated-examples) · [Integrations](#integrations)

## `State`

`State` is a thin container for dynamic data. It enables data sharing across multiple parts of code without making these parts directly dependent on each other.

```js
import { State } from "statepod";

const counterState = new State(42);

document.querySelector("button").addEventListener("click", () => {
  counterState.setValue((value) => value + 1);
});

counterState.on("set", ({ current }) => {
  document.querySelector("output").textContent = String(current);
});
```

In this example, a button changes a counter value and an `<output>` element shows the updating value. Both elements are decoupled from each other: they are only aware of the shared counter state, but not of each other.

A `"set"` event callback is called each time the state value changes and immediately when the callback is added. Subscribe to the `"update"` event to have the callback respond only to the subsequent state changes without the immediate invocation.

## `PersistentState`

`PersistentState` is a variety of `State` that syncs its data to the browser storage and restores it on page reload. The way it's used is almost identical to `State`.

```diff
- import { State } from "statepod";
+ import { PersistentState } from "statepod";

- const counterState = new State(42);
+ const counterState = new PersistentState(42, { key: "counter" });

  document.querySelector("button").addEventListener("click", () => {
    counterState.setValue((value) => value + 1);
  });

  counterState.on("set", ({ current }) => {
    document.querySelector("output").textContent = String(current);
  });
```

By default, `PersistentState` stores its data at the specified `key` in `localStorage` and transforms the data with `JSON.stringify()` and `JSON.parse()`. Switch to `sessionStorage` by setting `options.session` to `true` in `new PersistentState(value, options)`. Set custom `serialize()` and `deserialize()` in `options` to override the default data transforms used with the browser storage. Alternatively, use custom `{ read(), write()? }` as `options` to set up custom interaction with an external storage.

Instances of `PersistentState` automatically sync their values with the browser storage when created and updated. At other times, call `.emit("sync")` on a `PersistentState` instance to sync its value from the browser storage when needed.

## `Route`

`Route` stores the URL and exposes a `window.location`-like API for SPA navigation with a URL matching API.

```js
import { Route } from "statepod";

const route = new Route();
```

Navigate to other URLs in a SPA fashion similarly to the browser APIs:

```js
route.href = "/intro";
route.assign("/intro");
route.replace("/intro");
```

Or in a more fine-grained manner:

```js
route.navigate({ href: "/intro", history: "replace", scroll: "off" });
```

Check the current URL value like a regular `string` with `route.href`:

```js
route.href === "/intro";
route.href.startsWith("/sections/");
/^\/sections\/\d+\/?/.test(route.href);
```

Or, alternatively, with `route.at(url)` returning `true` if the current URL matches `url`, and `false` otherwise:

```js
route.at("/intro"); // `true` at "/intro", `false` otherwise
route.at(/^\/sections\//); // `true` at "/sections/*"
```

Or with the extended form `route.at(url, x, y?)`, which is similar to the ternary conditional operator `atURL ? x : y`:

```js
document.querySelector("header").className = route.at("/", "full", "compact");
// at "/" ? then "full" : otherwise "compact"
```

Use `route.at(url, x, y?)` with dynamic values that require values from the URL pattern's capturing groups:

```js
document.querySelector("h1").textContent = route.at(
  /^\/sections\/(?<id>\d+)\/?/,
  ({ params }) => `Section ${params.id}`,
);
// at "/sections/:id", "Section <id>" (otherwise `undefined`)
```

Or, alternatively, with a string URL pattern:

```js
import { url } from "url-shape";

document.querySelector("h1").textContent = route.at(
  url("/sections/:id"),
  ({ params }) => `Section ${params.id}`,
);
// at "/sections/:id", "Section <id>" (otherwise `undefined`)
```

Use a URL builder like the one from `url-shape` also in conjunction with a URL schema to set up type-safe routes ([example](https://codesandbox.io/p/sandbox/qg7qg3?file=%2Fsrc%2Findex.ts)).

Enable SPA navigation with HTML links inside the specified container (or the entire `document`) without any changes to the HTML:

```js
route.observe(document);
```

Tweak the links' navigation behavior by adding a relevant combination of the optional `data-` attributes (corresponding to the `route.navigate()` options):

```html
<a href="/intro">Intro</a>
<a href="/intro" data-history="replace">Intro</a>
<a href="/intro" data-scroll="off">Intro</a>
<a href="/intro" data-spa="off">Intro</a>
```

Links covered by `route.observe(container, selector?)` also automatically receive the `data-active="true"` attribute whenever their `href` attribute matches the current URL, which can be used for additional styling. If links are added to the DOM asynchronously, call `route.emit("ready")` when the rendering is complete to mark active links.

Define what should be done when the URL changes:

```js
route.on("navigationcomplete", ({ href }) => {
  renderContent();
});
```

Define what should be done before the URL changes (in a way effectively similar to routing middleware):

```js
route.on("navigationstart", ({ href }) => {
  if (hasUnsavedInput)
    return false; // Quit the navigation, prevent the current URL change

  if (href === "/") {
    route.href = "/intro"; // SPA redirection
    return false;
  }
});
```

## Annotated examples

- [Shared state](https://codesandbox.io/p/sandbox/lqt3z2?file=%252Fsrc%252Findex.ts)
- [Shared form input state](https://codesandbox.io/p/sandbox/4q7f99?file=%252Fsrc%252Findex.ts)
- [Persistent shared state](https://codesandbox.io/p/sandbox/c9gt3r?file=%252Fsrc%252Findex.ts)
- [URL-based rendering](https://codesandbox.io/p/sandbox/kt6m5l?file=%252Fsrc%252Findex.ts)
- [Type-safe URL-based rendering](https://codesandbox.io/p/sandbox/qg7qg3?file=%2Fsrc%2Findex.ts)
- [SPA redirection](https://codesandbox.io/p/sandbox/rpl3gh?file=%252Fsrc%252Findex.ts)

Find also the code of these examples in the repo's [`tests`](https://github.com/axtk/statepod/tree/main/tests) directory.

## Integrations

[`react-statepod`](https://www.npmjs.com/package/react-statepod)
