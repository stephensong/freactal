# fr(e)actal

`freactal` is a composable state management library for React.

The library grew from the idea that state should be just as flexible as your React code; the state containers you build with `freactal` are just components, and you can compose them however you'd like.  In this way, it attempts to address the often exponential relationship between application size and complexity as projects grow.

Like Flux and React in general, `freactal` builds on the principle of unidirectional flow of data.  However, it does so in a way that feels idiomatic to ES2015+ and doesn't get in your way.

When building an application, it can replace [`redux`](redux.js.org), [`MobX`](https://mobx.js.org), [`reselect`](https://github.com/reactjs/reselect), [`redux-loop`](https://github.com/redux-loop/redux-loop), [`redux-thunk`](https://github.com/gaearon/redux-thunk), [`redux-saga`](https://github.com/redux-saga/redux-saga), `[fill-in-the-blank sub-app composition technique]`, and potentially [`recompose`](https://github.com/acdlite/recompose), depending on how you're using it.

Its design philosophy aligns closely with the [Zen of Python](https://www.python.org/dev/peps/pep-0020/):

```
Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
```

<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## Table of Contents

- [Guide](#guide)
  - [Containing state](#containing-state)
  - [Accessing state from a child component](#accessing-state-from-a-child-component)
  - [Transforming state](#transforming-state)
  - [Transforming state (cont.)](#transforming-state-cont)
  - [Intermediate state](#intermediate-state)
  - [Effect arguments](#effect-arguments)
  - [Computed state values](#computed-state-values)
  - [Composing multiple state containers](#composing-multiple-state-containers)
  - [Conclusion](#conclusion)
- [Architecture](#architecture)
- [API Documentation](#api-documentation)
  - [`provideState`](#providestate)
    - [`initialState`](#initialstate)
    - [`effects`](#effects)
      - [`initialize`](#initialize)
    - [`computed`](#computed)
    - [`middleware`](#middleware)
  - [`injectState`](#injectstate)
  - [`hydrate`](#hydrate)
- [Helper functions](#helper-functions)
  - [`hardUpdate`](#hardupdate)
  - [`softUpdate`](#softupdate)
  - [`spread`](#spread)
- [Server-side Rendering](#server-side-rendering)
  - [with `React#renderToString`](#with-reactrendertostring)
  - [with Rapscallion](#with-rapscallion)
- [FAQ](#faq)


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## Guide

This guide is intended to get you familiar with the `freactal` way of doing things.  If you're looking for something specific, take a look at the [API Documentation](#api-documentation).  If you're just starting out with `freactal`, read on!


### Containing state

Most state management solutions for React put all state in one place.  `freactal` doesn't suffer from that constraint, but it is a good place to start.  So let's see what that might look like.

```javascript
import { provideState } from "freactal";

const wrapComponentWithState = provideState({
  initialState: () => ({ counter: 0 })
});
```

In the above example, we define a new state container type using `provideState`, and provide it an argument.  You can think about the arguments passed to `provideState` as the schema for your state container; we'll get more familiar with the other possible arguments later in the guide.

The `initialState` function is invoked whenever the component that it is wrapping is instantiated.  But so far, our state container is not wrapping anything, so let's expand our example a bit.

```javascript
import React, { Component } from "react";
import { render } from "react-dom";

const Parent = ({ state }) => (
  <div>
    { `Our counter is at: ${state.counter}` }
  </div>
);

render(<Parent />, document.getElementById("root"));
```

That's a _very_ basic React app.  Let's see what it looks like to add some very basic state to that application.

```javascript
import React, { Component } from "react";
import { render } from "react-dom";
import { provideState } from "freactal";

const wrapComponentWithState = provideState({
  initialState: () => ({ counter: 0 })
});

const Parent = wrapComponentWithState(({ state }) => (
  <div>
    { `Our counter is at: ${state.counter}` }
  </div>
));

render(<Parent />, document.getElementById("root"));
```

Alright, we're getting close.  But we're missing one important piece: `injectState`.

Like `provideState`, `injectState` is a component wrapper.  It links your application component with the state that it has access to.

It may not be readily apparent to you why `injectState` is necessary, so let's make it clear what each `freactal` function is doing.  With `provideState`, you define a state template that can then be applied to any component.  Once applied, that component will act as a "headquarters" for a piece of state and the effects that transform it (more on that later).  If you had a reason, the same template could be applied to multiple components, and they'd each have their own state based on the template you defined.

But that only _tracks_ state.  It doesn't make that state accessible to the developer.  That's what `injectState` is for.

In early versions of `freactal`, the state was directly accessible to the component that `provideState` wrapped.  However, that meant that whenever a state change occurred, the entire tree would need to re-render.  `injectState` intelligently tracks which pieces of state that you actually access, and a re-render only occurs when _those_ pieces of state undergo a change.

Alright, so let's finalize our example with all the pieces in play.

```javascript
import React, { Component } from "react";
import { render } from "react-dom";
import { provideState, injectState } from "freactal";

const wrapComponentWithState = provideState({
  initialState: () => ({ counter: 0 })
});

const Parent = wrapComponentWithState(injectState(({ state }) => (
  <div>
    { `Our counter is at: ${state.counter}` }
  </div>
)));

render(<Parent />, document.getElementById("root"));
```

That'll work just fine!


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


### Accessing state from a child component

As was mentioned above, the `provideState`-wrapped component isn't really the one that provides access to state.  That's `injectState`'s job.  So what would stop you from injecting state into a child component, one that isn't containing state itself?  The answer is nothing!

Let's modify the example so that we're injecting state into a child component.

```javascript
import React, { Component } from "react";
import { render } from "react-dom";
import { provideState, injectState } from "freactal";


const Child = injectState(({ state }) => (
  <div>
    { `Our counter is at: ${state.counter}` }
  </div>
));

const wrapComponentWithState = provideState({
  initialState: () => ({ counter: 0 })
});

const Parent = wrapComponentWithState(({ state }) => (
  <Child />
));


render(<Parent />, document.getElementById("root"));
```

Let's review what's going on here.

1. Using `provideState`, we define a state-container template intended to store a single piece of state: the `counter`.
2. That template is applied to the `Parent` component.
3. When the `Parent` is rendered, we see that it references a `Child` component.
4. That `Child` component is wrapped with `injectState`.
5. Because `Child` is contained within the subtree where `Parent` is the root node, it has access to the `Parent` component's state.

We could insert another component at the end, and `injectState` into the `GrandChild` component, and it would work the same.


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


### Transforming state

Alright, so we know how to setup state containers, give them an initial state, and consume that state from child components.  But all of this is not very useful if state is never updated.  That's where effects come in.

Effects are the one and only way to change `freactal` state in your application.  These effects are defined as part of your state container template when calling `provideState`.  And they can be invoked from anywhere that state has been injected (with `injectState`).

Let's take a look at that first part.

```javascript
const wrapComponentWithState = provideState({
  initialState: () => ({ counter: 0 }),
  effects: {
    addOne: () => state => Object.assign({}, state, { counter: state.counter + 1 })
  }
});
```

You might be wondering why we have that extra `() =>` right before `state =>` in the `addOne` definition.  That'll be explained in the next section - for now, let's look at all the other pieces.

In the above example, we've defined an effect that, when invoked, will update the `counter` in our state container by adding `1`.

Since updating an element of state based on previous state (and potentially new information) is something you'll be doing often, `freactal` [provides a shorthand](#softupdate) to make this a bit more readable:

```javascript
const wrapComponentWithState = provideState({
  initialState: () => ({ counter: 0 }),
  effects: {
    addOne: softUpdate(state => ({ counter: state.counter + 1 }))
  }
});
```

Now let's look at how you might trigger this effect:

```javascript
const Child = injectState(({ state, effects }) => (
  <div>
    { `Our counter is at: ${state.counter}` }
    <button onClick={effects.addOne}>Add one</button>
  </div>
));
```

Wherever your `<Child />` is in your application, the state and effects it references will be accessible, so long as the state container is somewhere further up in the tree.

<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


### Transforming state (cont.)

If you've used Redux, effects are roughly comparable to an action-reducer pair, with a couple of important differences.

The first of those differences relates to asychronicity.  Under the hood, `freactal` relies heavily on `Promise`s to schedule state updates.  In fact, the following effects are all functionally equivalent:

```javascript
addOne: () => state => Object.assign({}, state, { counter: state.counter + 1 })
/* vs */
addOne: () => state => Promise.resolve(Object.assign({}, state, { counter: state.counter + 1 }))
/* vs */
addOne: () => state => new Promise(resolve => resolve(Object.assign({}, state, { counter: state.counter + 1 })))
```

To put it explicitly, the value you provide for each key in your `effects` object is:

1. A function that takes in some arguments (we'll cover those shortly) and returns...
2. A promise that resolves to...
3. A function that takes in state and returns...
4. The updated state.

Step 2 can optionally be omitted, since `freactal` wraps these values in `Promise.resolve`.

For most developers, this pattern is probably the least familiar of those that `freactal` relies upon.  But it allows for some powerful and expressive state transitions with basically no boilerplate.

For example, any number of things can occur between the time that an effect is invoked and the time that the state is updated.  These "things" might include doing calculations, or talking to an API, or integrating with some other JS library.

So, you might define the following effect:

```javascript
updatePosts: () => fetch("/api/posts")
  .then(result => result.json())
  .then(({ posts }) => state => Object.assign({}, state, { posts }))
```

In other words, any action that your application might take, that ultimately _could_ result in a state change can be simply expressed as an effect.  Not only that, but this pattern also allows for effects and UI components to be tested with clean separation.

And, perhaps most importantly, this pattern allows for intermediate state.


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


### Intermediate state

So far, we haven't see any arguments to the first, outer-most function in our effect definitions.  In simple scenarios, this outer-function may seem unnecessary, as in the illustration above.

But what about cases where you want state to be updated part-way through an operation?  You _could_ put all this logic in your UI code, and invoke effects from there multiple times.  But that's not ideal for a number of reasons:

1. A single effect might be invoked from multiple places in your application.
2. The code that influences how state might be transformed is now living in multiple places.
3. It is much harder to test.

Fundamentally, the problem is that this pattern violates the principle of separation of concerns.

So, what's the alternative?

Well, we've already defined an effect as a function that, when invoked, will resolve to another function that transforms state.  Why couldn't we re-use this pattern to represent this "part-way" (or intermediate) state?  The answer is: nothing is stopping us!

The first argument passed to an effect in the outer function is the same `effects` object that is exposed to components where state has been injected.  And these effects can be invoked in the same way.  Even more importantly, because effects always resolve to a `Promise`, we can wait for an intermediate state transition to complete before continuing with our original state transition.

That might be a lot to take in, so let's look at an example:

```javascript
const wrapComponentWithState = provideState({
  initialState: () => ({
    posts: null,
    postsPending: false
  }),
  effects: {
    setPostsPending: softUpdate((state, postsPending) => ({ postsPending })),
    getPosts: effects => effects.setPostsPending(true)
      .then(() => fetch("/api/posts"))
      .then(result => result.json())
      .then(({ posts }) => effects.setPostsPending(false).then(() => posts))
      .then(posts => state => Object.assign({}, state, { posts }))
  }
});
```

There's a lot going on there, so let's go through it piece by piece.

- The initial state is set with two keys, `posts` and `postsPending`.
  + `posts` will eventually contain an array of blog posts or something like that.
  + `postsPending` is a flag that, when `true`, indicates that we are currently fetching the `posts`.
- Two `effects` are defined.
  + `setPostsPending` sets the `postsPending` flag to either `true` or `false`.
  + `getPosts` does a number of things:
    * It invokes `setPostsPending`, setting the pending flag to `true`.
    * It waits for the `setPostsPending` effect to complete before continuing.
    * It fetches some data from an API.
    * It parses that data into JSON.
    * It invokes `setPostsPending` with a value of `false`, and waits for it to complete.
    * It resolves to a function that updates the `posts` state value.

In the above example, `setPostsPending` has a synchronous-like behavior - it immediately resolves to a state update function.  But it could just as easily do something asynchronous, like make an AJAX call or interact with the IndexedDB API.

And because all of this is just `Promise` composition, you can put together helper functions that give consistency to intermediate state updates.  Here's an example:

```javascript
const wrapWithPending = (pendingKey, cb) => effects  =>
  effects.setFlag(pendingKey, true)
    .then(cb)
    .then(value => effects.setFlag(pendingKey, false).then(() => value));
```

Which could be consumed like so:

```javascript
const wrapComponentWithState = provideState({
  initialState: () => ({
    posts: null,
    postsPending: false
  }),
  effects: {
    setFlag: softUpdate((state, key, value) => ({ [key]: value })),
    getPosts: wrapWithPending("postsPending", () => fetch("/api/posts")
      .then(result => result.json())
      .then(({ posts }) => state => Object.assign({}, state, { posts }))
    )
  }
});
```


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


### Effect arguments

But what if you want to update state with some value that you captured from the user?  In Redux parlance: what about action payloads?

If you were looking closely, you may have noticed we already did something like that when we invoked `setPostsPending`.

Whether you are invoking an effect from your UI code or from another effect, you can pass arguments directly with the invocation.  Those arguments will show up after the `effects` argument in your effect definition.

Here's an example:

```javascript
const wrapComponentWithState = provideState({
  initialState: () => ({ thing: "val" }),
  effects: {
    setThing: (effects, newVal) => state => Object.assign({}, state, { thing: newVal })
  }
});
```

And it could invoked from your component like so:

```javascript
const Child = injectState(({ state, effects }) => {
  const onClick = () => effects.setThing("new val");
  return (
    <div>
      { `Our "thing" value is: ${state.thing}` }
      <button onClick={onClick}>Click here to change the thing!</button>
    </div>
  );
});
```


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


### Computed state values

As an application grows, it becomes increasingly important to have effective organizational tools.  This is especially true for how you store and transform data.

Consider the following state container:

```javascript
const wrapComponentWithState = provideState({
  initialState: () => ({
    givenName: "Walter",
    familyName: "Harriman"
  }),
  effects: {
    setGivenName: softUpdate((state, val) => ({ givenName: val })),
    setFamilyName: softUpdate((state, val) => ({ familyName: val }))
  }
});
```

Let's say that we're implementing a component and we want to display the user's full name.  We might write that component like this:

```javascript
const WelcomeMessage = injectState(({ state }) => {
  const fullName = `${state.givenName} ${state.familyName}`;
  return (
    <div>
      {`Hi, ${fullName}, and welcome!`}
    </div>
  );
});
```

That seems like a pretty reasonable piece of code.  But, even for a small piece of data like a full name, things can get more complex as the application grows.

What if we're displaying that full name in multiple components?  Should we compute it in all those places, or maybe inject state further up the tree and pass it down as a prop?  That can get messy to the point where you're passing down dozens of props.

What if the user is in a non-English locale, where they may not place given names before family names?  We would have to remember to do that everywhere.

And what if we want to derive another value off of the generated `fullName` value?  What about multiple derived values, derived from other derived values?  What if we're not dealing with names, but more complex data structures instead?

`freactal`'s answer to this is computed values.

You've probably run into something like this before.  Vue.js has computed properties.  MobX has computed values.  Redux outsources this concern to libraries like `reselect`.  Ultimately, they all serve the same function: exposing compound values to the UI based on simple state values.

Here's how you define computed values in `freactal`, throwing in some of the added complexities we mentioned:

```javascript
const wrapComponentWithState = provideState({
  initialState: () => ({
    givenName: "Walter",
    familyName: "Harriman",
    locale: "en-us"
  }),
  effects: {
    setGivenName: softUpdate((state, val) => ({ givenName: val })),
    setFamilyName: softUpdate((state, val) => ({ familyName: val }))
  },
  computed: {
    fullName: ({ givenName, familyName, locale }) => startsWith(locale, "en") ?
      `${givenName} ${familyName}` :
      `${familyName} ${givenName}`,
    greeting: ({ fullName, locale }) => startsWith(locale, "en") ?
      `Hi, ${fullName}, and welcome!` :
      `Helló ${fullName}, és szívesen!`
  }
});
```

_**Note:** This is not a replacement for a proper internationalization solution like `react-intl`, and is for illustration purposes only._

Here we see two computed values, `fullName` and `greeting`.  They both rely on the `locale` state value, and `greeting` actually relies upon `fullName`, whereas `fullName` relies on the given and family names.

How might that be consumed?

```javascript
const WelcomeMessage = injectState(({ state }) => (
  <div>
    {state.greeting}
  </div>
));
```

In another component, we might want to just use the `fullName` value:

```javascript
const Elsewhere = injectState(({ state }) => (
  <div>
    {`Are you sure you want to do that, ${state.fullName}?`}
  </div>
));
```

Hopefully you can see that this can be a powerful tool to help you keep your code organized and readable.

Here are a handful of other things that will be nice for you to know.

- Computed values are generated _lazily_.  This means that if the `greeting` value above is never accessed, it will never be computed.
- Computed values are _cached_.  Once a computed value is calculated once, a second state retrieval will return the cached value.
- Cached values are _invalidated_ when dependencies change.  If you were to trigger the `setGivenName` effect with a new name, the `fullName` and `greeting` values would be recomputed as soon as React re-rendered the UI.

That's all you need to know to use computed values effectively!


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


### Composing multiple state containers

We started this guide by noting that, while most React state libraries contain state in a single place, `freactal` approaches things differently.

Before we dive into how that works, let's briefly consider some of the issues that arise with the centralized approach to state management:

- Oftentimes, it is hard to know how to organize state-related code.  Definitions for events or actions live separately from the UI that triggers them, which lives separately from functions that reduce those events into state, which also live separately from code that transforms state into more complex values.
- While React components are re-usable ([see](http://www.material-ui.com/) [component](http://elemental-ui.com/) [libraries](https://github.com/brillout/awesome-react-components)), complex stateful components are a hard nut to crack.  There's this fuzzy line when addressing complexity in your own code that, when crossed, means you should be using a state library vs React's own `setState`.  But how do you make that work DRY across applications and team boundaries?
- Sometimes you might want to compose full PWAs together in various ways.  But if they need to interact on the page or share state in some way, how do you go about accomplishing this?  The results here are almost universally ad-hoc.
- It is an often arduous process when it comes time to refactor your application and move state-dependant components into different parts of your application.  Wiring everything up can be tedious as hell.

These are constraints that `freactal` aims to address.  Let's take a look at a minimal example:

```javascript
const Child = injectState(({ state }) => (
  <div>
    This is the GrandChild.
    {state.fromParent}
    {state.fromGrandParent}
  </div>
));

const Parent = provideState({
  initialState: () => ({ fromParent: "ParentValue" })
})(() => (
  <div>
    This is the Child.
    <GrandChild />
  </div>
));

const GrandParent = provideState({
  initialState: () => ({ fromGrandParent: "GrandParentValue" })
})(() => (
  <div>
    This is the Parent.
    <Child />
  </div>
));
```

Its important to notice here that `Child` was able to access state values from both its `Parent` and its `GrandParent`.  All state keys will be accessible from the `Child`, unless there is a key conflict between `Parent` and `GrandParent` (in which case `Parent` "wins").

This pattern allows you to co-locate your code by feature, rather than by function.  In other words, if you're out a new feature for your application, all of that new code - UI, state, effects, etc - can go in one place, rather than scattered across your code-base.

Because of this, refactoring becomes easier.  Want to move a component to a different part of your application?  Just move the directory and update the import from the parents.  What if this component accesses parent state?  If that parent is still an anscestor, you don't have to change a thing.  If it's not, moving that state to a more appropriate place should be part of the refactor anyway.

But one word of warning: accessing parent state can be powerful, and very useful, but it also necessarily couples the child state to the parent state.  While the coupling is a "loose" coupling, it still may introduce complexity that should be carefully thought-out.

One more thing.

Child effects can also trigger parent effects.  Let's say your UX team has indicated that, whenever an API call is in flight, a global spinner should be shown.  But maybe the data is only needed in certain parts of the application.  In this scenario, you could define `beginApiCall` and `completeApiCall` effects that track how many API calls are active.  If above `0`, you show a spinner.  These effects can be accessed by call-specific effects further down in the state hierarchy, like so:

```javascript
const Child = injectState(({ state, effects }) => (
  <div>
    This is the GrandChild.
    {state.fromParent}
    {state.fromGrandParent}
    <button
      onClick={() => effects.changeBothStates("newValue")}
    >
      Click me!
    </button>
  </div>
));

const Parent = provideState({
  initialState: () => ({ fromParent: "ParentValue" }),
  effects: {
    changeParentState: (effects, fromParent) => state =>
      Object.assign({}, state, { fromParent }),
    changeBothStates: (effects, value) =>
      effects.changeGrandParentState(value).then(state =>
        Object.assign({}, state, { fromParent: value })
      )
  }
})(() => (
  <div>
    This is the Child.
    <GrandChild />
  </div>
));

const GrandParent = provideState({
  initialState: () => ({ fromGrandParent: "GrandParentValue" }),
  effects: {
    changeGrandParentState: (effects, fromGrandParent) => state =>
      Object.assign({}, state, { fromGrandParent })
  }
})(() => (
  <div>
    This is the Parent.
    <Child />
  </div>
));
```


### Conclusion

We hope that you found this guide to be helpful!

If you find that a piece is missing that would've helped you understand `freactal`, please feel free to [open an issue](https://github.com/FormidableLabs/freactal/issues/new).  For help working through a problem, [reach out on Twitter](http://twitter.com/divmain), open an issue, or ping us on [Gitter](https://gitter.im/FormidableLabs/rapscallion).

You can also read through the API docs below!


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## Architecture

**TODO**


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## API Documentation

### `provideState`

**TODO**

#### `initialState`

**TODO**

#### `effects`

**TODO**

##### `initialize`

**TODO**

#### `computed`

**TODO**

#### `middleware`

**TODO**

### `injectState`

**TODO**

### `hydrate`

**TODO**


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## Helper functions

**TODO**

### `hardUpdate`

This handy helper provides better ergonomics when defining an effect that updates state, regardless of the previous state.

It can be consumed like so:

```javascript
import { provideState, hardUpdate } from "freactal";
const wrapComponentWithState = provideState({
  // ...
  effects: {
    myEffect: hardUpdate({ setThisKey: "to this value..." })
  }
});
```

Which is equivalent to the following:

```javascript
import { provideState } from "freactal";
const wrapComponentWithState = provideState({
  // ...
  effects: {
    myEffect: () => state => Object.assign({}, state, { setThisKey: "to this value..." })
  }
});
```


### `softUpdate`

`softUpdate` is provides a shorthand for updating an element of state that _is_ dependant on the previous state.

It can be consumed like so:

```javascript
import { provideState, softUpdate } from "freactal";
const wrapComponentWithState = provideState({
  // ...
  effects: {
    myEffect: softUpdate(state => ({ counter: state.counter + 1 }))
  }
});
```

Which is equivalent to the following:

```javascript
import { provideState, softUpdate } from "freactal";
const wrapComponentWithState = provideState({
  // ...
  effects: {
    myEffect: () => state => Object.assign({}, state, { counter: state.counter + 1 })
  }
});
```

Any arguments that are passed to the invocation of your effect will also be passed to the function you provide to `softUpdate`.

I.e.

```javascript
effects: {
  updateCounterBy: (state, addVal) => Object.assign({}, state, { counter: state.counter + addVal })
}
```

is equivalent to:

```javascript
effects: {
  myEffect: softUpdate((state, addVal) => ({ counter: state.counter + addVal }))
}
```


### `spread`

**TODO**


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## Server-side Rendering

**TODO**

### with `React#renderToString`

**TODO**

### with Rapscallion

**TODO**


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## FAQ

**Do you support time-traveling?**

TODO

**What middleware is available?**

- [freactal-devtools](https://github.com/FormidableLabs/freactal-devtools)
- [freactal-logger](https://github.com/FormidableLabs/freactal-logger)

**TODO**
