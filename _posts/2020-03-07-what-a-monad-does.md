---
layout: post
title: Ask Not What a Monad Is, Ask What a Monad Does
date: 2020-03-07 14:00:00 -0600
categories: functional-programming haskell monads
---
Time to add my voice to the ever-growing Internet cacophony that tries to explain the concept of monads. But rather than focus on what a monad *is*, like (it seems to me) most other articles, I'll attempt to explain what a monad *does*, at least from a programmer's point of view (though I may write a category theory-oriented explanation of monads in the future). I'll be working in Haskell for this article, but the principles should be at least somewhat generally applicable.

Before anything else in this article, I'll introduce my thesis statement: "A monad abstracts the rules of a given computational environment." Or, more specifically, "A monad lets a programmer define computations in a given computational environment (or 'universe') without worrying about maintaining continuity to stay within the rules of that environment." Keep this in mind as the article progresses, as I'll attempt to build up to it.

First, let's talk about the `Maybe` monad in Haskell. Firstly, whenever anyone talks about "the *x* monad", they're really talking about the set of behaviours of a type, as related to functioning as a monad, or perhaps the type's implementation of the `Monad` typeclass (more later). If, like me, you come from an object-oriented background, you can think about the type as implementing some sort of interface to be a proper monad, which is somewhat like how typeclasses in functional programming work, but instead of dealing with OOP classes, they deal with FP types. This means that `Maybe` is, first and foremost, a type defined as follows:

```Haskell
data Maybe a = Just a | Nothing
```

`Maybe` is commonly used to express possible failures or invalid results. Many a function is defined as an `a -> Maybe b` mapping which returns `Nothing` if the specific `a` given to it leads to some sort of failure, or some sort of `Just b` if the `a` leads to a success. As programs grow in complexity, it may be desirable, or even necessary, to chain these functions together as part of some larger computation, so we must have a way of defining these functions so that they can handle `Nothing`s as well as `Just`s. But it would be bothersome to redefine each and every function to map `Maybe a -> Maybe b`, especially if we can already see them all doing the same thing: if `Nothing`, return `Nothing`, otherwise do the rest of the work with the value inside the `Just`. So, we might want to define a function we can re-use to take care of all of this. We'll call it `>>=` (pronounced "bind"):

```Haskell
--(>>=) :: Maybe a -> (a -> Maybe b) -> Maybe b
Nothing >>= _ = Nothing
Just x >>= f = f x
```

Then, we can string together `a -> Maybe b` functions, like `doThis 5 >>= doThat`. Neat, huh?

Next, let's consider the List monad, but we'll think of it again as a type first and foremost. Lists are used to contain multiple values of some other type. This has many uses, but the one we'll be focusing on here, in the context of monads, is for representing non-deterministic computations. For example, one may need to define a function, which we'll call `sqrtn :: Floating a => a -> [a]`, for calculating *both* square roots of a number, not just the principal (positive) ones. For example, `sqrtn 4` is equal to `[2, -2]`, since squaring either number results in 4.

If we had multiple `a -> [b]` functions, we might want to chain them together like we did with `Maybe`, using another version of `>>=` that works on lists:

```Haskell
--(>>=) :: [a] -> (a -> [b]) -> [b]
a >>= f = concat $ map f a
```

Notice how we have a common task we want to carry out in two, and probably more of these computational contexts? Spoiler alert: this function, and two more that we'll get into later, are already provided for us in Haskell, and enforced on anything that wants to be a monad by the `Monad` typeclass. Of the three functions that each `Monad` must define, `>>=` is the most powerful one. Instead of having to make each monad-returning function take in a monad as well, and worry about the plumbing that's needed to handle all the possible values of that monad, we let `>>=` take care of it. Note that the type declarations have all been commented out here. This is because they aren't really the *true* types of the functions. The `Monad` typeclass already defines their type more generally (using polymorphism) in one fell swoop:

```
class Monad m where
	(>>=) :: m a -> (a -> m b) -> m b
```

Let's introduce the second function of the `Monad` typeclass while we're here: `return`. It simply takes a value and wraps it in a monadic type. Here's the definitions for `Maybe` and lists:

```Haskell
class Monad m where
	return :: a -> m a

--return :: a -> Maybe a
return a = Just a

--return :: a -> [a]
return a = [a]
```

For now, it's mainly useful if we needed to write a function that deals with monads (anything that implements the `Monad` typeclass) in general. We'll see another use for it, and where it gets its (strange-looking) name from later on.

But first, we have one more Haskell monad to get through to complete our explanation: the `IO` monad. The mechanics of this are more opaque (and much better explanied in some other article), but all we need to know here is that instead of dealing with a computational environment where something can fail, have multiple results, or any other myriad monads, the `IO` type deals with values coming from the outside world, beyond the strictness and pureness of Haskell. An `IO` value kinda-sorta has a pure value inside of it *somewhere*, but there's no way to get at it since we don't know what it is until we run the program, and it reaches out and grabs that value. However, it can't just grab that *something* like in an imperative language, as that would violate the purity of our code. Instead, we chain together functions, like before. This is why I said `IO`s only "kinda-sorta" have values inside them: instead of actually extracting a value from outside, we write functions that take our yet-to-be-determined outside values as parameters. So, we'll need a `>>=` function. We'll also throw in a `return` function for good measure. These are their types as far as `IO` is concerned, but what they actually do comes from within Haskell, and aren't really definable:

```Haskell
--(>>=) :: IO a -> (a -> IO b) -> IO b
--return :: a -> IO a
```

Now that we have a `>>=` for `IO`, we can do the same `doThis >>= doThat` chaining of functions as before, but for the outside world. For example, let's define a function that prompts the user for a number, typed in via the command line:

```Haskell
getNumber :: IO Int
getNumber = putStrLn "Enter a number" >>=
	(\_ -> getLine >>=
	(\n -> return (read n)))
```

Note that we can return different types from each of our functions, so long as they all start with `IO`. Here, putStrLn is a function `String -> IO ()` that prints out whatever gets passed to it. Since it can't really come back with any meaningful result, it instead returns the empty tuple `()`, equivalent to `void` in many imperative languages, which we can safely throw out as we do in the next function. However, `getLine` is an `IO String` value, which demonstrates that as long as we stay within the realm of `IO`, we can have functions that return anything within that. And speaking of throwing out values, let's introduce another function , `>>`, that makes it cleaner to do just that:

```Haskell
class Monad m where
	(>>) :: m a -> m b
	a >> b = a >>= \_ -> b
```

Note that this one function can work with any `Monad`, so it's defined as part of the typeclass. Then, we can rewrite `getNumber` as:

```Haskell
getNumber = putStrLn "Enter a number" >>
	getLine >>=
	(\n -> return (read n))
```

`IO` is a big deal to Haskell, since it lets us interact with the impure outside world from within a pure language, and also helps us keep it roped off from the rest of our (pure) code. In fact, there's one more trick we can pull off to make our functional code look almost indistinguishable from imperative code, and that is `do`-notation:

```Haskell
getNumber = do
	putStrLn "Enter a number"
	n <- getLine
	return (read n)
```

Where did our `Monad` functions, `>>=` and `>>`, go? They've been done away with as part of `do`-notation. If we would connect one function ("line") to another with `>>`, that becomes writing that first function on a line by itself. If we would use `>>=` instead, and we would be concerned with what value comes out of that first function, instead of making it a parameter a lambda function, we assign it to a variable name using a left arrow (`<-`), and use it later on. This is mere syntactic sugar for the chain of `>>`s and `>>=`s we were using before, but it makes our code look a lot cleaner.

In addition, we get to see why `return` gets its name. If we want to end a monadic function by returning something some pure computation, we need to wrap it in an `IO` first. However, in an imperative language, we would just use the `return` keyword to end our function. See the parallels? Though it's a misnomer, Haskell's `return` function mimics, in `do`-notation, what would be going on in similar imperative code.

In conclusion, monads in Haskell let us, as programmers, work in different computational environments (or "universes"), such as those that consider the potential of failure (`Maybe`), multiple results (lists), and influence to/from the outside world (`IO`), and keep ourselves within their rules without having to continually account for everything that goes on within those environments. There are many more monads in Haskell's standard library, and even more so in other people's code, and perhaps code you may have yet to write, but even `Maybe`, lists and `IO` on their own bring a great deal of power to Haskell's abilities.

This is a high-level overview, and the finer details, most importantly the monad laws, are best covered by one of many other sources out there. If you read this article to try and grasp monads for yourself, don't despair if you're still confused after reading this! Find some other articles that look at them from a different angle, and find an explanation that works for you. And most importantly, practice for yourself and write code that uses various monads. Any simplified analogy/explanation, this one included, will break down over time as you get further into the details, but they *are* useful for getting you started.