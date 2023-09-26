---
layout: post
title: "Transposing a Text File in Haskell"
date: 2019-07-14 17:25:00 -0500
categories: haskell
---
This post answers an exercise in [Chapter 3 of the updated version of Real World Haskell][rwh-ch3], reprinted here:

>Write a program that transposes the text in a file. For instance, it should convert `"hello\nworld\n"` to `”hw\neo\nlr\nll\nod\n”`.

Using the command-line framework described earlier in the chapter, we are left to define a function `myFunction` which transforms a string (from the input file) into another string (to be written to the output file). We can treat the overall function as follows:

```Haskell
myFunction :: String -> String
myFunction = unlines . transpose . lines
```

This leaves us with the hard part: defining `transpose`. One solution that fits the "hello world" example given by the exercise is the following, taken from [this Stack Overflow answer][stackoverflow]:

```Haskell
transpose :: [[a]] -> [[a]]
transpose ([]:_) = []
transpose x = (map head x) : transpose (map tail x)
```

However, this algorithm is best suited to lists of lists in which each sublist's length is lesser than or equal to the one before it. That means that for input such as this:

```
The quick
brown fox jumps
over the lazy dog.
```

...the resulting output is truncated in later lines:

```
Tbo
hrv
eoe
 wr
qn 
u t
ifh
coe
kx 
```

Instead, we can define this version of `transpose`. Though it loses out on the polymorphism of the above example, it adds spaces at the ends of columns where the rows in the original matrix end prematurely. It does this recursively by appending a space wherever it finds an empty sublist, until it reaches the base case of having a list made up entirely of empty sublists:

```Haskell
transpose :: [String] -> [String]
transpose xs
    | all null xs = []
    | otherwise = map safeHead xs : transpose (map safeTail xs)
    where
        safeHead x = if null x then ' ' else head x
        safeTail x = if null x then [] else tail x
```

Running this against the "quick brown fox" example yields the following output:

```
Tbo
hrv
eoe
 wr
qn 
u t
ifh
coe
kx 
  l
 ja
 uz
 my
 p 
 sd
  o
  g
  .
```

To make the function more polymorphic, we can instead let the caller specify a value to pad lists with:

```Haskell
transpose :: a -> [[a]] -> [[a]]
transpose padValue xs
    | all null xs = []
    | otherwise = map safeHead xs : transpose padValue (map safeTail xs)
    where
        safeHead x = if null x then padValue else head x
        safeTail x = if null x then [] else tail x
```

This way, we can specify a value to fill in the blanks with (most commonly spaces, when dealing with strings) that is of the type of the contents of the list of lists. Thanks to currying, we can redefine `myFunction` from before as:

```Haskell
myFunction :: String -> String
myFunction = unlines . transpose ' ' . lines
```

Within the scope of the entire program, however, `a` will need to implement `Show` to be output to a text file.

[rwh-ch3]: https://github.com/tssm/up-to-date-real-world-haskell/blob/master/4-functional-programming.org#exercises
[stackoverflow]: https://stackoverflow.com/questions/2578930/understanding-this-matrix-transposition-function-in-haskell