---
layout: post
title: "Effects in Hyperapp V2"
date: 2019-07-19 14:15:00 -0500
categories: javascript hyperapp
---
Web apps written in [Hyperapp V2][hyperapp] are based primarily on a single, global state, and a set of actions that transform that state (or return a new state based on the old one). This is good for keeping code simple, but what if you need to do something that produces a side effect, like calling a web API, generating a random number, or so on? This is where effects come in.

Before I continue, [Jorge Bucaran's GitHub issue regarding effects in Hyperapp V2][bucaran] was massively helpful to me in understanding this. Go read it if this post doesn't help you. There already exists a sizeable library of Hyperapp effects in [`hyperapp-fx`][hyperapp-fx] for doing common things that involve side effects, so it's not strictly necessary that you implement your own. However, if you run into something that requires a side effect and doesn't have a Hyperapp effect written for it, or you just want to learn how effects work "under the hood" regardless, read on.

Effects are made up of two parts, the second of which is optional (more on this later). The first is the *effect function*. It is a function, usually defined in ES6 arrow syntax (like almost everything in Hyperapp), which takes two parameters: a set of properties, and a `dispatch` function which the effect should call when it's finished doing what it's doing:

```JavaScript
const somethingEffect = (props, dispatch) => {
    // do something
    dispatch( /* action */, /* parameters to action */ )
};
```

`dispatch` is used to call other actions, which can be defined in `props`. It comes from Hyperapp itself, and we call our actions with it so that Hyperapp knows how to update the application state to whatever our callback action transforms it into.

The second part of an effect is the *effect constructor*. It is another ES6 arrow function, but all it does is return an array containing the effect function from before, and a set of properties which gets passed as a parameter:

```JavaScript
const something = props => [somethingEffect, props];
```

This isn't strictly necessary, but it makes our code look a bit cleaner. Instead of having to call an effect like this:

```JavaScript
const someAction = () => [
    { /* state before effect */ },
    [somethingEffect, { /* props */ }]
]
```

...we can call it like this:

```JavaScript
const someAction = () => [
    { /* state before effect */ },
    something({ /* props */ })
]
```

Let's dive into an example: Say we want to make a set of effects for working with the browser's local storage. This is already covered by [`hyperapp-fx`][hyperapp-fx], but we'll make our own simple implementation anyway, for the sake of learning. We'll want two effects: one for reading from local storage, and one for writing. Since local storage is a simple key-value store, we'll make no assumption about what actually lies in the value within the effects themselves, in case someone wants to write a raw string, use a different serialization method, and so on. I wrote these initially for a Hyperapp to-do list example I was working on, which can be viewed [here][hyperapp-todo] if you want to see the entire codebase for it.

Our effect for reading a value from local storage, with an effect function and an effect constructor, looks something like this:

```JavaScript
const readFromLocalStorageEffect = (dispatch, props) => {
    var value = localStorage.getItem(props.key);
    dispatch(props.action, value);
};

const readFromLocalStorage = props => [readFromLocalStorageEffect, props];
```

For writing a value to local storage, we'll assume that we don't want to do anything to the application state after we write to local storage. If we did, it would be as simple as adding a call to `dispatch` after we set the item, with whatever action we specified earlier in `props`:

```JavaScript
const writeToLocalStorageEffect = (dispatch, props) => {
    localStorage.setItem(props.key, props.value);
};

const writeToLocalStorage = props => [writeToLocalStorageEffect, props];
```

Being the simpler of the two, we'll outline an action that uses the `writeToLocalStorage` effect first. We'll assume that `state.someValue` can be serialized as JSON:

```JavaScript
const writeAction = state => {
    ...state,
    writeToLocalStorage({
        key: "someKey",
        value: JSON.stringify(state.someValue)
    })
};
```

Meanwhile, an action that uses the `readToLocalStorage` effect to populate the same value in the application state might look like this:

```JavaScript
const readAction = state => {
    ...state,
    readFromLocalStorage({
        key: "someKey",
        action: (state, value) => ({
            ...state,
            someValue: JSON.parse(value)
        })
    })
}
```

[hyperapp]: https://github.com/jorgebucaran/hyperapp
[bucaran]: https://github.com/jorgebucaran/hyperapp/issues/750
[hyperapp-fx]: https://github.com/okwolf/hyperapp-fx
[hyperapp-todo]: https://github.com/lpraz/hyperapp-todo