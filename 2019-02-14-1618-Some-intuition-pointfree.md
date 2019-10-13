# Some Basic Intuitions about Pointfree Style and Composition

I've been playing [Paiza's new game (in Japansese)](https://paiza.jp/botchi) with Haskell recently. 
I'd always use [Pointfree.io](http://pointfree.io/) to see what a pointfree version of my functions would look like,
and today I took the initiative to dissect some more complex pointfree definitions.
In the process I feel like I've gained some more intuition about how to approach pointfree style.

My motivation of learning pointfree style is my preference for Elm's pipeline style.
In Elm, a function `a -> b` may be written in such a way that it looks like 
a datum passes through a pipeline of processing components.
For example, a function that parses a string of a binary integer into a Elm Int can be written so (ignoring error handling):

```elm
parseBinary : String -> Int
parseBinary s = 
  let 
    l = s |> String.length |> List.range 0 |> List.map ((^) 2) 
  in 
    s 
    |> String.toList 
    |> List.map2 Tuple.pair l 
    |> List.filter (\x -> Tuple.second x == '1') 
    |> List.map Tuple.first 
    |> List.sum
```

Accidentally sank too much time into the above code segment. 
Anyway, the feeling is to get a better handle at thinking about process.
Elm `(|>)` operator does it quite nicely by making me thinking about how a datum would "flow" through the functions.

If you are not familiar with Haskell notation yet, here is a quick glossary:

```haskell
expression :: TypeOfExpression
-- Comment containing value or output of the expression above

-- anonymous function (closure)
\parameter1 -> \parameter2 -> expression
  :: TypeOfParameter1 -> TypeOfParameter2 -> TypeOfExpression
  
-- named function
f parameter1 parameter2 = expresstion
f :: TypeOfParameter1 -> TypeOfParameter2 -> TypeOfExpression
```

So today I came across a problem in which I need to layout a grid of 0's.
In procedural languages, this would probably be achieved by a nested `for` loop.
In declarative languages like Haskell, a first thing I find I need to adapt to is its vast vocabulary.
Just like structural programming was adding vocabulary to describe the structure implicit in `jmp`s,
declarative programming is adding vocabulary to describe the intentions implicit in the control flows.
In Haskell, I decide to use nested `repeat`:

```haskell
take 5 (repeat (take 5 (repeat '0'))) :: [[Char]]
-- ["00000","00000","00000","00000","00000"]
```

`repeat` takes an element and makes an infinite list of it, and `take 5` takes the first 5 elements from that list.

Pointfree style means the functions are composed together first, and only finally apply to the "point",
that is the parameter, or the datum. To do so, the composition operator `(.)` is used:

```haskell
(take 5 . repeat . take 5 . repeat) '0' :: [[Char]]
-- ["00000","00000","00000","00000","00000"]
```

Hopefully the equivalent two code blocks above demonstrates what I mean by the datum flowing from right to left.

In the code above, `'0'` is the datum. I can generalize it into a function so that 
I can fill a 5-by-5 grid with any `Char` I want:

```haskell
f c = (take 5 . repeat . take 5 . repeat) c
f :: Char -> [[Char]]
f '1' :: [[Char]]
-- ["11111","11111","11111","11111","11111"]
```

Here is where the namesake of pointfree style should be introduced.
Pointfree means the parameter ("point") is absent ("free") in the definition.
In the definition of function `f` above, notice parameter `c` appears last
both on the left hand side of the `=` sign and the right hand side.
Therefore it can be elided on both sides and the definition becomes:

```haskell
f = take 5 . repeat . take 5 . repeat
f :: Char -> [[Char]]
```

And it can be used the same way.

But I need to take the dimensions from input, not the character,
so I need to parameterize the initial expression differently:

```haskell
g w h = take h (repeat (take w (repeat '0')))
g :: Int -> Int -> [[Char]]
g 4 6 :: [[Char]]
["0000","0000","0000","0000","0000","0000"]
```

`w` is the width of the grid, `h` is the height.
I have no idea how to pointfree-ize this at a glance, so I enlist the help of Pointfree.io:

```haskell
g = flip take . repeat . flip take (repeat '0')
g :: Int -> Int -> [[Char]]
```

(\* If `g w h` is `g h w` instead, Pointfree.io gives a more complicated result. )

What's new here is `flip`. `flip` flips the order of the parameters of a function.
For example:

```haskell
(++) "first" "second" :: [Char]
-- "firstsecond"
flip (++) "first" "second" :: [Char]
-- "secondfirst"
```

Let's dissect the component functions of `g` above from right to left:

```haskell
g = flip take . repeat . flip take (repeat '0')
g :: Int -> Int -> [[Char]]
```

The right-most function is:

```haskell
flip take (repeat '0') :: Int -> [Char]
```

Remember `repeat` generates an infinite list and `take` cuts that list short to the first few of elements.
`take` normally gets the number of elements first, then is applied to the list to be cut.
In this case, `flip` enables programmer to give `take` the list to be cut first, and then decide how many elements are needed.
`(repeat '0')` is for a row, so this component function receives the width, and generates a row.

The middle function is `repeat`. It repeats the row generated by the right-most function we just discussed.

The left-most function is `flip take` again, but this time instead of `(repeat '0')`,
the list to cut is the repeating rows generated by the middle `repeat`.

So far the result of the function is a `[[Char]]`. 
I need to print it out as a grid, so I use `intercalate "\n"` to concatenate the list with line breaks.

```haskell
g w h = intercalate "\n" (take h (repeat (take w (repeat '0'))))
g :: Int -> Int -> [Char]
```

The pointfree version is:

```haskell
g = (intercalate "\n" .) . flip take . repeat . flip take (repeat '0')
g :: Int -> Int -> [Char]
```

Compare this to the version without `intercalate`:

```haskell
g = flip take . repeat . flip take (repeat '0')
g :: Int -> Int -> [[Char]]
```

The only difference is the `intercalate` part composed to the left.
For me this is a new development. 
So far the functions I've composed are independent, partially applied, or `flip`-ed then partially applied.
It has never occured to me to partially apply `(.)` itself. Like this:

```haskell
(intercalate "\n" .) :: (a -> [[Char]]) -> a -> [Char]
```

This type signature reminds me immediately of the pattern of callback function,
with which I have become familiar in JavaScript.
For reference, a list of partial applications of `(.)`:

```haskell
intercalate "\n"
  :: [[Char]] -> [Char]
(intercalate "\n" .)
  :: (a -> [[Char]]) -> a -> [Char]
((intercalate "\n" .) .)
  :: (a1 -> a2 -> [[Char]]) -> a1 -> a2 -> [Char]
(((intercalate "\n" .) .) .)
  :: (a1 -> a2 -> a3 -> [[Char]]) -> a1 -> a2 -> a3 -> [Char]
```

The pattern here is that each `(.)` adds a parameter to the callback function.

To practice, the binary integer parser in Haskell, pointfree style.

```haskell
parseInt :: [Char] -> Int
parseInt = foldl' (flip ((+) . fst)) 0 . filter ((=='1') . snd) . zip (map (2^) [0..])
```
