---
layout: post
title: Three Basic ReadP Exercises
date: 2020-11-30 20:10:00 -0600
categories: parsing haskell
---
Recently, I've been learning [parser combinators][wikipedia-parser-combinators] in Haskell, using the included `Text.ParserCombinators.ReadP` library. For validating small bits of text, such as checking something like a phone number for proper formatting, they're a departure from [regular expressions][wikipedia-regex] that I and many other programmers are already accustomed to. However, they're more easily readable maintainable than the keysmash that regular expressions tend to become when they get sufficiently complex. This post details a few small exercises for `ReadP`, which should (hopefully) be applicable to other Haskell parser combinator libraries such as Parsec, Attoparsec and Megaparsec as well.

This post won't attempt to teach parser combinators from scratch. For that, I recommend [Two Wrongs' introduction to parser combinators][two-wrongs-readp], which I followed when I first started. Past that, the [docs on Hackage][hackage-readp] are a good reference for what each `ReadP` operation/included parser does.

# Canadian postal codes

*Input: a [Canadian postal code][wikipedia-postal-codes], with or without a space in the middle, in uppercase or lowercase. (eg: "h0h0h0", "H0H 0H0")*

*Output: a `newtype`d `String` containing the postal code in uppercase and with a space. (eg: "H0H 0H0")*

Canadian postal codes are six-character strings that alternate between letters and digits, usually but not always written with a space after the first three. In addition, the first letter is limited to some, but not all, letters of the Latin alphabet (A through Z). This means we'll need to construct a parser out of four smaller parsers: one for a single starting letter, one for a single Latin letter in general, one for a single digit, and one for an optional space.

For a single digit, we can use `satisfy isDigit`. `satisfy` takes a `Char -> Bool`, and we can use `isDigit` (from `Data.Char`) to get a parser.

For a single Latin letter, however, `isLetter` returns `True` not just for Latin characters, but for anything defined in Unicode to be a letter. Therefore, we'll have to make our own function, which we can then give to `satisfy` to make a parser that returns a single uppercase or lowercase Latin chatacter:

```Haskell
isPostalCodeLetter :: Char -> Bool
isPostalCodeLetter = (flip elem ['A'..'Z']) . toUpper
```

The first letter in a postal code is limited to a certain subset of the Latin alphabet as well. In the same manner as the above, we can use `satisfy` to get a parser for that, too:

```Haskell
isFsaDistrictLetter :: Char -> Bool
isFsaDistrictLetter = (flip elem "ABCEGHJKLMNPRSTVXY") . toUpper
```

Finally, for our optional space, we'll need to again make a parser that takes a single space. `satisfy isSpace`, again leaning on `Data.Char`, should do. To make it take *zero* or one space, however, we'll need to put the `optional` parser in front of it:

```Haskell
optional $ satisfy isSpace
```

Finally, when put together, our postal code parser looks like this, including what we need to return a properly formatted postal code:

```Haskell
newtype PostalCode = PostalCode String

postalCode :: ReadP PostalCode
postalCode = do
    fsaDistrict <- satisfy isFsaDistrictLetter
    fsaDigit <- satisfy isDigit
    fsaLetter <- satisfy isPostalCodeLetter
    optional $ satisfy isSpace
    lduDigit1 <- satisfy isDigit
    lduLetter <- satisfy isPostalCodeLetter
    lduDigit2 <- satisfy isDigit
    eof
    return $ PostalCode
        [ toUpper fsaDistrict
        , fsaDigit
        , toUpper fsaLetter
        , lduDigit1
        , toUpper lduLetter
        , lduDigit2
        ]
```

We use `eof` at the end so that we fail to match on any strings that have more to then than what we've matched on so far. (eg: "H0H 0H0Hello, world!")

# North American phone numbers

*Input: a North American phone number with the following criteria:*
* *beginning either with "+1", "1", or nothing, with an optional hyphen or space to the right*
* *a three-digit area code, optionally surrounded by parentheses, optionally separated by a hyphen or space*
* *three digits, optionally separated by a hyphen or space*
* *four digits, optionally separated by a hyphen or space*

*Output: a newtyped `String` containing the phone number in [E.164 format][twilio-phone-numbers]*

First of all, there's a few places where we expect either a space, a hyphen, or nothing. We can use the `optional` parser to get part of the way there, but then we need a parser that reads either a single hyphen or space. There are a few different ways of doing this, but for the sake of using the `choice` parser which we haven't seen yet, here's what I used:

```Haskell
spaceOrHyphen :: ReadP Char
spaceOrHyphen = choice [char ' ', char '-']
```

The `char` parser reads one of the character that is given to it, and the `choice` parser goes through the list of parsers it is given until it finds one that succeeds. We can also use `char ' ' <|> char '-'`, with `<|>` coming from `Control.Applicative`. This is what `choice` uses behind the scenes, since `ReadP` is a `Monad`, and therefore, an `Applicative`. We could also use `satisfy` with `flip elem "- "` if we wanted, depending on what you find more readable.

Next, let's turn our attention to specific parts of the phone number. Firstly, it starts with either a "+1", "1" or nothing at all, which is the international calling code used for the US, Canada, and other countries that are part of the [North American Numbering Plan][wikipedia-nanp]. Again, there are a few different ways of reading this, but to make use of `choice` again, we'll can use this:

```Haskell
optional $ choice [string "`", string "+1"]
```

For groups of numbers, such as the groups of three and four digits after the area code, the simplest method is to employ the `count` parser. For brevity, we'll define a parser `digit` that parses a single digit:

```Haskell
digit :: ReadP Char
digit = satisfy isDigit
```

Then, to get a group of digits, we can use `count n digit`, where `n` is however many digits we want.

For the area code, we need to deal with the optional surrounding parentheses. To show off another built-in parser we haven't seen yet, we'll use `between`. For an area code, we can do this:

```Haskell
between
    (optional (char '('))
    (optional (char ')'))
    (count 3 digit)
```

The first argument is a parser for what's on the left of what we want to parse (the optional "("), the second for the right side (the optional ")"), and the third argument is for the part in the middle (the three digits of the area code), which `between` will pass on to us. `between` is more or less there for convenience, so you can just as easily make a parser that combines the first parser, then the third, then the second if desired.

Finally, we can put our complete phone number parser together like this:
```Haskell
newtype PhoneNumber = PhoneNumber String

nanpPhoneNumber :: ReadP PhoneNumber
nanpPhoneNumber = do
    optional $ choice [string "1", string "+1"]
    optional spaceOrHyphen

    nanpPhoneNumber = do
    optional $ choice [string "1", string "+1"]
    optional spaceOrHyphen
    
    areaCode <- between
        (optional (char '('))
        (optional (char ')'))
        (count 3 digit)
    optional spaceOrHyphen
    
    phoneNumber1 <- count 3 digit
    optional spaceOrHyphen
    
    phoneNumber2 <- count 4 digit
    eof
    let str = "+1" ++ areaCode ++ phoneNumber1 ++ phoneNumber2
    return $ PhoneNumber str
```

# Passwords

*Input: a password containing at least:*
* *Eight characters*
* *One uppercase letter*
* *One lowercase letter*
* *One digit*
* *One symbol (any character that isn't a letter, number or whitespace)*

*Output: a newtyped `String` containing the unchanged password*

Doing this as a parser combinator might be unnecessary and a little convoluted, but as an exercise, we get to make use of `ReadP`'s lookahead capabilities, which we'll see in a second. For now, though, we need functions that check if a single character is any of the four above things. `isUpper`, `isLower` and `isDigit` check all the boxes for the first three (unlike for postal codes before, we'll let non-Latin characters slide), but for the last, we'll need a new function:

```Haskell
isPwSymbol :: Char -> Bool
isPwSymbol c = not $ or $ map ($ c) [isUpper, isLower, isDigit, isSpace]
```

`isSpace` also comes from `Data.Char`. With that, we can put our complete password parser together like this (with some explanation below):

```Haskell
newtype Password = Password String

password :: ReadP String
password = do
    let criteria = [isUpper, isLower, isDigit, isPwSymbol]
    pw <- look
    if and (map (\f -> or (map f pw)) criteria)
        then return $ Password pw
        else pfail
```

We start out by putting our criteria functions from above in a list (`criteria`), then using `look` to get the rest of the password (since we haven't parsed anything yet, the full password) without parsing it.

This sets up the one-liner we use for our if-statement. `map f pw` applies a function (one of our criteria functions) to each character of the password. Wrapping that in `f -> or (...)` creates an anonymous function that returns `True` if `f` returns `True` for any one of the characters in the password. Wrapping *that* in `map (...) criteria` applies that function to `criteria`, which gives us a list of `Bool`s representing which criteria are met by the password. Finally, calling `and` on all of that tells us if our password matches all the criteria.

Finally, we use that one-liner in our if statement. If the password matches all the criteria, we return the password wrapped in a `newtype`. Else, we call `pfail`, which always fails and returns nothing.

[wikipedia-parser-combinators]: https://en.wikipedia.org/wiki/Parser_combinator
[wikipedia-regex]: https://en.wikipedia.org/wiki/Regular_expression
[two-wrongs-readp]: https://two-wrongs.com/parser-combinators-parsing-for-haskell-beginners
[hackage-readp]: https://hackage.haskell.org/package/base-4.14.0.0/docs/Text-ParserCombinators-ReadP.html
[wikipedia-postal-codes]: https://en.wikipedia.org/wiki/Postal_codes_in_Canada
[twilio-phone-numbers]: https://support.twilio.com/hc/en-us/articles/223183008-Formatting-International-Phone-Numbers
[wikipedia-nanp]: https://en.wikipedia.org/wiki/North_American_Numbering_Plan