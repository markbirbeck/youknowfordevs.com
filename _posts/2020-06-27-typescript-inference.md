---
layout: post
title:  "Let TypeScript Inference Take The Strain"
date:   2020-06-27 15:13:00 +0000
comments: true
tags: 
 - typescript
 - javascript
---

I was doing a code review recently and saw something like this:

```js
const mapEventName = (eventName: string): string => {
  switch (eventName) {
    case 'UPDATE_ABC':
      return 'ABC Updated'
    case 'UPDATE_DEF':
      return 'DEF Updated'
    default:
      return eventName
  }
}
```

Putting aside whether this should use a `Map` or some other technique instead of `switch`, I was interested specifically in the type information that was provided for the return value.

Something I see a lot when people adopt TypeScript is to type _everything_. I think it's a good idea to add types to structures and interfaces that are passing between layers of our code. But to add extra code to indicate that a function returns a string, _when it so obviously returns a string_ is overkill.

So what's the alternative?

Well in this example there is no need for the return type to be specified. TypeScript has impressive powers of inference and in this function it will have no trouble deducing that the type of the return value will be a string literal. If we look at the function there are three places where a return value is specified; two of them stipulate a string constant to return, and the third says to return the parameter that was passed in--which is itself defined as a string.

Together these conditions combine such that there is _no possibility_ that this function will return anything other than a string. TypeScript will therefore act accordingly. To verify this, take a look at the IntelliSense provided by your favourite editor. Here is the IntelliSense with the return type specified explicitly:

![Function intellisense with return type](/images/uploads/function-intellisense-with-return-type.png)

and here is exactly the same IntelliSense when the return type is omitted:

![Function intellisense without return type](/images/uploads/function-intellisense-without-return-type.png)

And if we want to reassure ourselves that TypeScript is still doing the same checking, let's try to use the function in a position where a number is required:

```js
const a = Math.max(mapEventName('UPDATE_ABC'), 4);
````

Put this in a TypeScript-enabled editor and you'll see there is a type error, regardless of the fact that the return type is not explicitly specified:

![Function intellisense incompatible type](/images/uploads/function-intellisense-incompatible-type.png)

You might ask whether missing off one declaration gains us much, but I would suggest it's a mindset. Rather than approaching TypeScript as this rigid straightjacket that must be wrapped around all of our code, we instead see it as a flexible combination of two things:

* a supercharged linter that can infer types from our code and then check that we're not doing anything dumb, and;
* a type language that we can add as we need to, to help to resolve ambiguities.

This gives us the option of a much 'lighter touch' approach to TypeScript.

For more on TypeScript's type inference see [Type Inference](https://www.typescriptlang.org/docs/handbook/type-inference.html). And for a particularly clever scenario where types can be inferred from `assert()` statements, see ['asserts condition'](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#assertion-functions).