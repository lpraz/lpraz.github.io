---
layout: post
title: "An Action for Arbitrarily Setting Nested State Properties in Hyperapp V2"
date: 2019-10-05 10:50:00 -0500
categories: hyperapp javascript
---
A problem I ran into recently when working on [a Hyperapp project of mine][time-from] is that I had to keep writing more or less the same actions for setting properties in the state whenever an `<input>` element changed:

```javascript
const SetFooBar = (state, value) => ({
    ...state,
    foo: {
        ...state.foo,
        bar: value
    }
});

// ...

const View = state => (
    <input
        value={state.foo.bar}
        oninput={[SetFooBar, event => event.target.value]} />
);
```

Rinse and repeat, ad nauseam, for `state.foo.baz`, `state.foo.bax`, and maybe even `state.bar.foo`, `state.baz.foo`, and so on. This, of course, violates the ["Don't repeat yourself" (DRY)][wikipedia-dry] principle, and if you have enough `<input>`s in your Hyperapp application, your code can become littered with these redundant actions.

Let's try and genericize this to an action, which we'll call `SetFromOnInput`, that can work on any part of the state and be reused across `<input>`s. Firstly, we can do at least a little better by not having to repeatedly specify a payload creator, since we know the payload will always be from `event.target.value`. We could make `SetFromOnInput` an array containing the action and the payload creator:

```javascript
const SetFromOnInput = [
    (state, value) => {
        ...state,
        foo: {
            ...state.foo,
            bar: value
        }
    },
    event => event.target.value
];
```

We could also replace `value` with `event.target.value` and still only have a function, since Hyperapp defaults to passing the entire event as the payload if no payload creator is specified. For this article, we'll choose this option.

```javascript
const SetFromOnInput = (state, event) => ({
    ...state,
    foo: {
        ...state.foo,
        bar: event.target.value
    }
});
```

Now, the tricky part: we want to be able to access any property inside `state`, not just `state.foo.bar`. Thankfully, JavaScript's bracket notation gets us part of the way there: just as we can access `state.foo.bar`, we can also access `state["foo"]["bar"]`, and both will refer to the same thing: the property named `bar` of the property named `foo` of `state`. We can make SetFromOnInput a function of one parameter which returns a function of two parameters (a sort of [currying][wikipedia-currying]). This first parameter will be the name of the property we want to be changed when we return the new state, and the other two parameters will be the same as before. Though we break away from strictly functional-looking JavaScript here, we'll at least maintain a lack of side effects in our action.

```javascript
const SetFromOnInput = prop => (state, event) => {
    let newState = { ...state };
    newState[prop] = event.target.value;
    return newState;
};

// ...

const View = state => (
    <input
        value={state.foo.bar}
        oninput={SetFromOnInput("foo")} />
);
```

There's a problem with this, though: we can't get to any of the nested properties this way, but we want to be able to by calling `SetFromOnInput("foo.bar")`. This means our `SetFromOnInput` action will need to do some string manipulation and recursion. We'll define an inner recursive function that works on an array of property names (eg: `["foo", "bar"]`), copies everything over to a new state object as before at each level, then stops at the last level and sets the property there to its new value:

```javascript
const SetFromOnInput = prop => (state, event) => {
    const inner = (obj, prop, value) => {
        let newObj = { ...obj };
        if (prop.length == 1)
            newObj[prop[0]] = value;
        else
            newObj[prop[0]] = inner(obj[prop[0]], prop.slice(1), value);

        return newObj;
    };
    return inner(state, prop.split("."), event.target.value);
};
```

This is where I stopped, but there are still ways in which this can be improved and iron-cladded. For example, arrays aren't accessible using square brackets (eg: `"state.baz[0]"`), and using dot notation (eg: `"state.baz.0"`) doesn't look nearly as idiomatic. We should, however, be able to do some more string replacing to make this work by passing the following on as our list of properties when we first call the inner function instead:

```javascript
prop.replace("[". ".")
    .replace("]", "")
    .split(".");
```

If we want to handle undefined values, we can, for example, return prematurely from `inner` if we ever get to one:

```javascript
if (typeof obj === undefined)
    return undefined;
```

Finally, this solution may appear shaky enough that you'd rather find some other way of accomplishing the same. From my (very preliminary) research, there are at least a few JavaScript libraries out there, such as [Ramda][ramda-set] and [Lodash][lodash-set], that should be able to do this for you, if you don't mind the additional overhead adding a new package would bring. In addition, look into the concept of lenses from functional programming.

A big thanks to Prusprus on StackOverflow, whose [post][stackoverflow-prusprus] served as the inspiration for this solution.

[time-from]: https://github.com/lpraz/time-from
[wikipedia-dry]: https://en.wikipedia.org/wiki/Single_responsibility_principle
[mdn-brackets]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Property_Accessors#Bracket_notation
[wikipedia-currying]: https://en.wikipedia.org/wiki/Currying
[stackoverflow-prusprus]: https://stackoverflow.com/a/26407251
[ramda-set]: https://ramdajs.com/docs/#set
[lodash-set]: https://lodash.com/docs/4.17.15#set