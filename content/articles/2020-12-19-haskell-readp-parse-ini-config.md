---
layout: post
title: Parsing an INI File with Haskell's ReadP
date: 2020-12-19 17:25:00 -0600
categories: parsing haskell
---
Building on my [last post][lpraz-readp-exercises] about Haskell's `Text.ParserCombinators.ReadP`, wouldn't it be nice if we could see a larger-scale example of parser combinators in action? This post will show that, in the form of an exercise where we parse a complete INI configuration file. This example won't go on for as long as, say, parsing a JSON or XML file, or code for any decently complex programming language, but it'll give us a feel for a larger-scale parsing problem and how to solve it.

Note: if you actually want to be able to read and use the contents of INI files in a Haskell program, especially in production, there are [some][hackage-ini] [libraries][hackage-config-ini] on Hackage devoted to this already. I don't endorse any particular library in this post, but I encourage you to take a look at them and use whichever one suits your needs, as it will definitely be more built-up and robust than the one described here.

# Specification

First, let's see how INI files are structured. To keep things simple, we'll follow, more or less, the Windows implementation as given by the [Format][wikipedia-ini-format] section from the Wikipedia article, and leave any extras up to the reader (some examples later on in the "Exercises" section).

First of all, the metaphorical bread and butter of INI files is *keys*. Keys have both a name and a value, with the name to the left of an equals sign ("=") and the value to the right. The "=" is a reserved character for key names, so we can safely assume that the first "=" we come across is the end of the name. Semicolons (";") are also reserved in the key, for reasons that we'll see later. Keys are separated from each other (and other elements of an INI file) by newlines. As an example, here's two keys from a hypothetical INI file:

```ini
foo=bar
baz=quux
```

The other building block, at least in terms of what we'll actually be reading from an INI file, are *sections*. Sections are declared with a name of arbitrary characters, enclosed in square brackets ("[" and "]"). Any keys that appear after a section declaration belong to that section, and since there's no way to mark the end of a section, one section ends when another begins (or where the file itself ends). We'll also assume keys can appear before the first section declaration of a file, and that these keys don't belong to any section (so-called "global" properties). Here's an example of a section, using the two keys from above:

```ini
[values]
foo=bar
baz=quux
```

Finally, INI files can be marked with comments. INI files, for our purposes, have only line comments that begin with a semicolon at the start of a line, and end with the end of the line. These are ignored, and have no effect on the configuration values in the file. In addition, we'll accept any lines containing only whitespace (spaces and tabs) as lines to be ignored as well. Here's an example of a comment:

```ini
; comment (ignore me!)
```

The Windows implementation treats INI files as case-insensitive, but again, to keep things simple, we'll allow for case-sensitivity. At the very least, this keeps us from having to define/use a datatype that lets us access our keys this way.

# Implementation

We'll gradually build up a parser for an entire INI file by starting with parsers that work on smaller pieces of the file. To start, we'll define parsers that work on pieces that are smaller than a single line, but are still a little more complex than the ones included with `ReadP`:

```Haskell
restOfLine :: ReadP String
restOfLine = manyTill get eol

eol :: ReadP ()
eol = choice
    [ char '\n' >> return ()
    , eof
    ]
```

`eol` parses either the end of a line (`char \n`) or the file (`eof`, from `ReadP`), and throws it away. `restOfLine` takes this a bit further, and reads from its input until it gets to the end of a line, using `eol`. Both of these will show up in a few other, "bigger" parsers.

Next, let's start parsing entire lines of the file. We'll start with parsing a line that defines a key:

```Haskell
iniKey :: ReadP (String, String)
iniKey = do
    name <- manyTill (satisfy (flip notElem "=;\n")) (char '=')
    value <- restOfLine
    return (name, value)
```

Since a key name is made of any character that isn't an =, ; or `\n`, we use `satisfy` (from `ReadP`) to take all those characters in, then use it in `manyTill` with `char '='` to read up to the =. Then, we can safely read in the rest of the line using the `restOfLine` parser we defined earlier. We end by returning a tuple with the name and value of the key.

Next, let's parse a line that forms a section declaration:

```Haskell
notChar :: Char -> ReadP Char
notChar c = satisfy (/= c)

iniSectionName :: ReadP String
iniSectionName = do
    char '['
    sectionName <- manyTill (notChar '\n') (char ']')
    eol
    return sectionName
```

We start by parsing a [, then as many characters that aren't newlines as possible until we get to a ] at the end of the section declaration. We then parse the end of the line, and return the name of the section that was declared. `notChar` is included for convenience, and parses anything that *isn't* the character given to it.

You might have wondered, "why not use `between` instead of having `char` and `manyTill` on separate lines? Consider a line like this. This line is invalid, due to having characters other than a newline after the first "]":

```ini
[Sect]ion]
```

According to the [ReadP documentation][hackage-readp], "`manyTill p end` parses zero or more occurrences of `p`, until `end` succeeds." For us, this means `manyTill` stops reading after the first closing bracket ("]"). On the other hand, again from the ReadP documentation, "`between open close p` parses `open`, followed by `p` and finally `close`." This doesn't necessarily mean that the first time `close` succeeds marks the only valid parsing result for `between`. In fact, if we used `between (char '[') (char ']') (manyTill (notChar '\n'))` instead of what we have now, both `Sect` and `Section` would be returned, and in the whole parser, would give us two valid section names instead of none. Instead of trying to use something other than `notChar '\n'` to account for this, I opted to use `manyTill` to keep the code more simple.

For the last type of individual line our parser will read, we'll parse what I refer to as an "ignore line" in my code, which includes comment lines and lines with only whitespace. We'll make two separate parsers for these, then combine them into one that reads either:

```Haskell
iniIgnoreLine :: ReadP ()
iniIgnoreLine = choice [iniComment, iniBlankLine]

iniComment :: ReadP ()
iniComment = do
    char ';'
    restOfLine
    return ()

iniBlankLine :: ReadP ()
iniBlankLine = do
    manyTill (satisfy (flip elem " \t")) eol
    return ()
```

`iniComment` parses a single semicolon, reads the rest of the line, and returns nothing. `iniBlankLine` parses many spaces and/or tabs, followed by the end of the line. `iniIgnoreLine` stitches the two together.

Now that we have these parsers, we'll want to make one for groups of keys, ending on a new section declaration or the end of the file. This, however, will use an additional, smaller parser to achieve cleaner separation of concerns when we build on this to parse entire sections:

```Haskell
import Data.Maybe (catMaybes)
import qualified Data.Map as Map

iniKeys :: ReadP (Map.Map String String)
iniKeys = do
    maybeKeys <- manyTill (choice
        [ iniKey >>= (return . Just)
        , iniIgnoreLine >> return Nothing
        ])
        lookEndOfSection
    let keys = catMaybes maybeKeys
    return $ Map.fromList keys

lookEndOfSection :: ReadP ()
lookEndOfSection = do
    rest <- look
    if (rest == []) || (head rest == '[') then return () else pfail
```

`lookEndOfSection` checks for the end of the file, or the beginning of a new section (marked with "["), using lookahead with the `look` parser so as not to read anything that isn't outside the group of keys that `iniKeys` is meant to parse. Aside from that, we parse the lines within the group of keys to a list of `Maybe`s, use `catMaybes` to throw away the `Nothing`s and extract the values from the `Just`s, and return a map of the keys we just parsed.

Believe it or not, the hard parts are more or less over. To parse a section from this point is relatively simple:

```Haskell
iniSection :: ReadP (String, Map.Map String String)
iniSection = do
    sectionName <- iniSectionName
    keys <- iniKeys
    return (sectionName, keys)
```
And to parse the entire file, we use this parser. Note again that we take any keys before the first section declaration to be a valid part of the file:

```Haskell
data Config = Config
    { keys :: Map.Map String String
    , sections :: Map.Map String (Map.Map String String)
    }

ini :: ReadP Config
ini = do
    keys <- iniKeys
    sections <- manyTill iniSection eof
    return $ Config keys (Map.fromList sections)
```

# Exercises

There are many ways this parser can be improved on, to further exercise your knowledge of Haskell parser combinators. I'll list a few here, lifted from the [Wikipedia page][wikipedia-ini]:

- Support number signs as comments, except where they are used at the beginning of a key name. (eg: `#foo=bar` is a key, `# foo=bar` is a comment)
- Support escape sequences (with a backslash) in key and section declarations, including those for number signs (`\#`), equals signs (`\=`), semicolons (`\;`), and brackets (`\[`, `\]`).
- Handle duplicate names. Should the parser fail, in some way, if a duplicate key is found? If not, should the first definition be the true one? The last one? All of them (multi-valued properties)?
- Improve the safety of the `Config` type from before. Think of the different issues that could appear if it were being used in some other code.
- Make the results of reading the INI file case-insensitive. There are a few ways of doing this, but some may tie into the above task.

[lpraz-readp-exercises]: /parsing/haskell/2020/12/01/haskell-readp-parser-exercises.html
[wikipedia-ini]: https://en.wikipedia.org/wiki/INI_file
[wikipedia-ini-format]: https://en.wikipedia.org/wiki/INI_file#Format
[hackage-ini]: https://hackage.haskell.org/package/ini
[hackage-config-ini]: https://hackage.haskell.org/package/config-ini
[hackage-readp]: https://hackage.haskell.org/package/base-4.14.0.0/docs/Text-ParserCombinators-ReadP.html