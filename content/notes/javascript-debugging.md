---
title: "JavaScript Debugging"
description: "Tips for making the most the browser Console API for debugging JavaScript"
date: 2020-11-20T12:39:00-06:00
categories:
  - javascript
tags:
  - debugging
  - console-api
slug: javascript-debugging
---

## Use the console

There's more to the console than just `console.log()`.

References:

- [Console API on MDN](https://developer.mozilla.org/en-US/docs/Web/API/Console)
- [Chrome Console API Reference](https://developers.google.com/web/tools/chrome-devtools/console/api)

### `console.dir()`

Displays an interactive list of the properties of the provided object. I believe this was a bigger deal with Internet Explorer debugging. It's been a while, but I believe in IE, `console.log(object)` would call result in the highly useful `"[object Object]"` whereas `console.dir(object)` would list the properties. There's less of a distinction in Firefox and Chrome.

In Firefox, `console.dir(object)` will print the inspectable object expanded automatically to the first layer.

{{< figure src="/images/consolelogvsdir.png" title="console.log() vs console.dir()" alt="Comparing console.log and console.dir output in JavaScript Debugger">}}

### `console.group() / .groupEnd()`

Group a set of messages together, useful when looping to group each loop's messages together.

### `console.table()`

Print out a table, useful for comparing objects with the same properties.

### `console.time() / .timeEnd()`

Measure performance of a chunk of code.

## User Timing API

The User Timing API provides a way to mark points in the browser performance tools. It's a much better alternative then expaning a bunch of anonymous methods and drilling down to find a click event.

### `performance.mark()`

Adds an entry with a timestamp in the performance buffer.
