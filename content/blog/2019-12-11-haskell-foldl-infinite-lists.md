---
layout: post
title: "Why Doesn't Haskell's foldl Work on Infinite Lists?"
date: 2019-12-11 20:45:00 -0600
categories: haskell
---
It's often said that `foldr` can be used, in some cases, with infinite lists, and also that `foldl` (and its more recommended, strictly-evaluated cousin, `foldl'`) doesn't. What does this mean, however?

Let's implement two functions,`allLessThanThreeL` and `allLessThanThreeR`, to demonstrate. In both, we'll be taking in a list of integers and checking if they're all less than three. We could do this very quickly by using `all (<3)`, but so that we can walk through what each fold does, we'll have them use `foldl` and `foldr`, respectively:

```Haskell
allLessThanThreeL :: [Integer] -> Bool
allLessThanThreeL = foldl f True
    where f acc x = if x >= 3 then False else acc

allLessThanThreeR :: [Integer] -> Bool
allLessThanThreeR = foldr f True
    where f x acc = if x >= 3 then False else acc
```

Note that in both cases, the inner function `f` disregards the accumulator, `acc`, entirely if `x` is not less than `3`. This will come into play shortly.

To begin, what happens when we try to evaluate each with a finite list, `[1, 3, 5]`? Let's see what these functions would expand to behind the scenes, starting with the `foldl` version:

```Haskell
allLessThanThreeL [1, 3, 5]
    = f (f (f True 5) 3) 1
    = f (f False 3) 1
    = f False 1
    = False
```

Since `foldl` is left-associative (meaning it groups its recursive calls to the left), the first argument is another call to `foldl`, which repeats until we get an empty list (our base case) and we can substitute our initial accumulator, `True`. This means that our inner combining function `f` must take `acc` as its first argument. Therefore, we must know what `acc` is before we can evaluate `f`, even if `acc` might be ignored based on the value of `x`.

Now, let's try it with the `foldr` version:

```Haskell
allLessThanThreeR [1, 3, 5] = f 1 (f 3 (f 5 True))
    = f 1 False -- f 3 is already False, no matter what the second argument is!
    = False
```

Here, since `foldr` is right-associative, and thanks to Haskell's laziness, we can use `f`'s ability to short-circuit to our advantage! There's more levels of recursion inside the `f 3 ...` call, but since `3` is our first argument to `f`, and we already know that `f 3` is `False` no matter what the other parameter is, we can skip all of that other computation on the inside.

Now, imagine what would happen with an infinite list, `[1..]`, which would expand to `[1, 2, 3, 4, 5, ...]`. To evaluate `allLessThanThreeL [1..]`, since it is based on the left-associative `foldl`, we must keep going deeper and deeper into the resulting recursive computation. This inevitably ends in a stack overflow if you're a computer, or getting bored and doing something else if you're a human being stepping through the computation by hand. For `allLessThanThreeR [1..]`, meanwhile, we only need to go a few levels deep until we get to `f 3 allLessThanThreeR [4..]`, which we already know is `False` since `f 3 <anything> == False`. This is an example of a case in which `foldr` works on an infinite list, but `foldl` doesn't.

`foldr` isn't a silver bullet for infinite lists, though. Let's define two more functions that use a left and a right fold, this time for finding the sum of a list of integers. Of course, we could always use `sum` (and probably should in any real code), but for the sake of example, here's two functions `sumL` and `sumR`:

```Haskell
sumL :: [Integer] -> Integer
sumL = foldl (+) 0

sumR :: [Integer] -> Integer
sumR = foldr (+) 0
```

Now, what happens if we run them with our infinite list from before? When I tried it in `ghci`, `sumR [1..]` resulted in a stack overflow, while `sumL [1..]` crashed my entire computer. Neither is exactly an ideal result. This goes to show that simply using `foldr` isn't the be-all, end-all solution to working with infinite lists. If your inner combining function doesn't have a condition that allows the fold to stop considering any further elements of the list, no type of fold will save you from running out of stack space (or memory, period).