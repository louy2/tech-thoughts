While exploring intuitions about `Monad` with `List`s, 
I discovered this gem with [Pointfree.io](http://pointfree.io/):

```haskell
f l = elemIndex (maximum l) l
f = elemIndex =<< maximum
f :: Ord a => [a] -> Maybe Int
```

The function `f` above tries to find the index of the maximum element in the list `l`. 
To do so `l` needs to be passed to both `maximum` and `elemIndex` separately,
so composition doesn't really work here. 
Naturally I went to explore the type signatures of the components:

```haskell
maximum :: (Foldable t, Ord a) => t a -> a
elemIndex :: Eq a => a -> [a] -> Maybe Int
(=<<) :: Monad m => (a -> m b) -> m a -> m b
```

(\* For those who are not familiar with Haskell, `Foldable`, `Ord`, `Eq` and `Monad`
are called Typeclasses, while `t`, `a`, `Maybe`, `Int`, `m` are Types. 
Typeclasses classify Types with their behaviors, 
e.g. values of Types in the Typeclass `Eq` can be compared for equality with `(==)`.
For those who know Object-Oriented concepts, a Typeclass is similar to an Interface.)

There's a glaring `Monad` staring at me.
Knowing this would not be obvious, I try my best at unification.
Since `(=<<)` takes in `elemIndex` as `(a -> m b)`,
I thought maybe `a` is `(a -> [a])` and `m b` is `Maybe Int`.
But `maximum` has nothing to do with `Maybe`.

With no progress from direct observation, I opt for some partial application:

```haskell
(elemIndex =<<) :: Eq a => ([a] -> a) -> [a] -> Maybe Int
(=<< maximum) :: (Foldable t, Ord a) => (a -> t a -> b) -> t a -> b
```

This got me more confused. Where does the first `([a] -> a)` come from?
Where does the `(a -> t a -> b)` come from?
Why do they look so different?
Also, shouldn't `(=<<)` take 2 parameters and therefore the partial applications
take only 1 parameter?

After struggling with it for an hour I went to `#haskell-beginners` for help.
After bribing the channel with a cat Tweet I found on Twitter,
`@dminuoso` came to the rescue.
He crucially pointed out that `m` is not any of the obvious type constructors,
but the partially applied function constructor `([a] ->)`, 
or the actual syntax, `((->) [a])`, since I guess the special syntax for partially
applying infix operators is not yet supported on the type level.

Knowing that `m` is `([a] ->)`, the rest is not as difficult now.
To recap the type signatures:

```haskell
maximum :: (Foldable t, Ord a) => t a -> a
elemIndex :: Eq a => a -> [a] -> Maybe Int
(=<<) :: Monad m => (a -> m b) -> m a -> m b
elemIndex =<< maximum :: Ord a => [a] -> Maybe Int
```

After unification, the type signatures become:

```haskell
maximum :: Ord a => [a] -> a
elemIndex :: Eq a => a -> ([a] -> Maybe Int)
(=<<) :: Monad m => (a -> ([a] -> Maybe Int)) -> ([a] -> a) -> ([a] -> Maybe Int)
elemIndex =<< maximum :: Ord a => [a] -> Maybe Int
-- I made the ellidable parentheses explicit for readability
```

Now it's easy to see how each pigeon fits in a hole:

```haskell
                              maximum :: Ord a => [a] -> a
                                                 -- |
elemIndex :: Eq a => a -> ([a] -> Maybe Int)     -- |
                          -- |                      |
                          -- v                      v
(=<<) :: Monad m => (a -> ([a] -> Maybe Int)) -> ([a] -> a) -> ([a] -> Maybe Int)
                                                               -- |
                                                               -- v
                              elemIndex =<< maximum :: Ord a => [a] -> Maybe Int
```

`@dminuoso` told me that this Monad is often called a Reader Monad.
It is used to keep a shared environment between functions, 
making passing the environment implicit. 
He challenged me to implement the `Reader` myself.
So I started.

First, the template.

```haskell
{-# LANGUAGE InstanceSigs #-}

newtype Reader e a = Reader (e -> a)

instance Monad (Reader e) where
    (>>=) :: Reader e a -> (a -> Reader e b) -> Reader e b
    (>>=) = undefined
```

The `InstanceSigs` pragma lets me annotate the unified type signature in the instance,
so that I am sure I have the types right before I start implementing.
For this kind of basic stuff, usually the type signature dictates the implementation.
That is, if I have the type right, it would force me to implement it right,
and any mistake would be a compile time error. Like this one:

> - No instance for (Applicative (Reader e))
    arising from the superclasses of an instance declaration
> - In the instance declaration for ‘Monad (Reader e)’

That's straightforward, I just need to add implementation for `Applicative` too:

```haskell
instance Applicative (Reader e) where
    pure :: a -> Reader e a
    pure = undefined
    (<*>) :: Reader e (a -> b) -> Reader e a -> Reader e b
    (<*>) = undefined
```

This error again:

> - No instance for (Functor (Reader e))
    arising from the superclasses of an instance declaration
> - In the instance declaration for ‘Applicative (Reader e)’

OK, let's add `Functor` as well.  

```haskell
instance Functor (Reader e) where
    fmap :: (a -> b) -> Reader e a -> Reader e b
    fmap = undefined
```

There was a time I had problem understanding these names,
but I have come to accept that these things are too abstract to name them something mundane.
They are essentially mathematical constructs, so I should be thankful that
they are not named after some 19th-century discoverer.

(\* `superclass` is mentioned in the error message above, and it is
pretty much what the error messages say.
To implement a typeclass, all its superclasses must be implemented as well,
because those behaviors are necessary for the typeclass to work.
Not necessarily that those behaviors will be used in the implementation,
but they are needed to satisfy mathematical laws related to the typeclass.)

Let the implementation begin from `Functor`:

```haskell
fmap :: (a -> b) -> Reader e a -> Reader e b
```

So `fmap` takes a function of `a -> b` and a `Reader` of `Reader e a`
and returns a `Reader` of `Reader e b`.
Recall that the signature of `Reader` is:

```haskell
newtype Reader e a = Reader (e -> a)
```

So to construct a `Reader e b` instance as the return of `fmap`,
I need a function of `e -> b` to give the `Reader` constructor.
But conversely, I can get a function of `e -> a` out of the parameter `Reader e a`. 
The problem now becomes how I can construct a function of `e -> b` with 
a function of `a -> b` and a function of `e -> a`.
Time to use [Hoogle](https://hoogle.haskell.org/) to search for a function with
such a type signature: [`(a -> b) -> (e -> a) -> (e -> b)`][hs1].
(You may click to signature to see the Hoogle results.)

[hs1]: https://hoogle.haskell.org/?hoogle=(a%20-%3E%20b)%20-%3E%20(e%20-%3E%20a)%20-%3E%20(e%20-%3E%20b)

Turns out it's just the function composition operator `(.)`.

```haskell
(.) :: (b -> c) -> (a -> b) -> a -> c
-- or in our case
(.) :: (a -> b) -> (e -> a) -> (e -> b)
```

(\* If you feel you can't see the equivalence above, know that only the positions
of the types are important here, not the names.)

Therefore, `Functor` can be implemented as:

```haskell
instance Functor (Reader e) where
    fmap :: (a -> b) -> Reader e a -> Reader e b
    fmap a2b (Reader e2a) = Reader (a2b . e2a)
```

Next is `Applicative`. There are two functions needed for it:

```haskell
pure :: a -> Reader e a
pure = undefined
(<*>) :: Reader e (a -> b) -> Reader e a -> Reader e b
(<*>) = undefined
```

Let's start from `pure`.
Following the same steps, `pure` takes a value of type `a` and returns a `Reader e a`,
which needs a `e -> a` to construct. So I search Hoogle for [`a -> (e -> a)`][hs2].
`const` is the result:

[hs2]: https://hoogle.haskell.org/?hoogle=a%20-%3E%20(e%20-%3E%20a)

```haskell
const :: a -> b -> a
```

It's a function that takes a value of type 'a' and produces a new function that returns
that value but only after taking one parameter and ignores it. 
Or in a more functional way (mathematical way) of thinking, the produced function maps
anything to the same value. It is used like so in this case:

```haskell
pure :: a -> Reader e a
pure a = Reader (const a)
```

Or the pointfree version:

```haskell
pure :: a -> Reader e a
pure = Reader . const
```

(\* For more about pointfree, please read [my other post about it](https://github.com/louy2/tech-thoughts/blob/master/2019-02-14-1618-Some-intuition-pointfree.md).)

Next up, the `(<*>)` operator:

```haskell
(<*>) :: Reader e (a -> b) -> Reader e a -> Reader e b
```

Following the same steps to get rid of the `Reader` wrapper,
what we need is some function with the following signature:

```haskell
f :: (e -> a -> b) -> (e -> a) -> (e -> b)
```

There is no ready made function on Hoogle of this signature,
so I have to write it myself.

```haskell
f :: (e -> a -> b) -> (e -> a) -> (e -> b)
f e2a2b e2a = let e2b e = e2a2b e (e2a e) in e2b
```

To assist with my thinking I name the functions after their types.
`e2a2b` correspond to the `e -> a -> b` for example.
Since I need to return a function of `e -> b`, 
I embed a function definition of `e2b` with `let in`.
It takes `e` of type `e` as a parameter. 
In the body, `e2a e` gives a value of `a`.
`e2a2b` is applied to `e` then `e2a e` to yield a value of `b`.

Now to change the above to satisfy the newtype check noises:

```haskell
(<*>) :: Reader e (a -> b) -> Reader e a -> Reader e b
(<*>) (Reader e2a2b) (Reader e2a) = let e2b e = e2a2b e (e2a e) in Reader e2b
```

To consolidate everything into a complete implementation of `Applicative`:

```haskell
instance Applicative (Reader e) where
    pure :: a -> Reader e a
    pure = Reader . const
    (<*>) :: Reader e (a -> b) -> Reader e a -> Reader e b
    (<*>) (Reader e2a2b) (Reader e2a) = let e2b e = e2a2b e (e2a e) in Reader e2b
```

Finally, the actual `Monad`.

```haskell
instance Monad (Reader e) where
    (>>=) :: Reader e a -> (a -> Reader e b) -> Reader e b
    (>>=) = undefined
```

The signature without the newtype noise is this:

```haskell
f :: (e -> a) -> (a -> e -> b) -> (e -> b)
```

Again, nothing can be found on Hoogle by this signature.
So I went about to implement it myself:

```haskell
f :: (e -> a) -> (a -> e -> b) -> (e -> b)
f e2a a2e2b = let e2b e = a2e2b (e2a e) e in e2b
```

I then got stuck for 3 hours trying to rewrite that with the `Reader` newtype at here:

```haskell
(>>=) :: Reader e a -> (a -> Reader e b) -> Reader e b
(>>=) (Reader e2a) a2reb = let e2b e = a2reb (e2a e) e in Reader e2b
```

The `a2reb (e2a e) e` part doesn't work, because a `Reader e b` cannot be directly applied to `e`.
After 3 hours I finally realized I just need a helper to take the function out from the newtype `Reader`:

```haskell
(>>=) :: Reader e a -> (a -> Reader e b) -> Reader e b
(>>=) (Reader e2a) a2reb = let e2b e = runReader (a2reb (e2a e)) e in Reader e2b
    where runReader (Reader f) = f
```

I felt so dumb.

Anyway, the implementation of `Monad` on `Reader`:

```hakell
instance Monad (Reader e) where
    (>>=) :: Reader e a -> (a -> Reader e b) -> Reader e b
    (>>=) (Reader e2a) a2reb = let e2b e = runReader (a2reb (e2a e)) e in Reader e2b
        where runReader (Reader f) = f
```

Finally, to combine the whole thing:

```haskell
{-# LANGUAGE InstanceSigs #-}

newtype Reader e a = Reader (e -> a)

instance Functor (Reader e) where
    fmap :: (a -> b) -> Reader e a -> Reader e b
    fmap a2b (Reader e2a) = Reader (a2b . e2a)

instance Applicative (Reader e) where
    pure :: a -> Reader e a
    pure = Reader . const
    (<*>) :: Reader e (a -> b) -> Reader e a -> Reader e b
    (<*>) (Reader e2a2b) (Reader e2a) = let e2b e = e2a2b e (e2a e) in Reader e2b

instance Monad (Reader e) where
    (>>=) :: Reader e a -> (a -> Reader e b) -> Reader e b
    (>>=) (Reader e2a) a2reb = let e2b e = runReader (a2reb (e2a e)) e in Reader e2b
        where runReader (Reader f) = f
```

For reference, the implementation of `instance Monad ((->) a)` is at [https://hackage.haskell.org/package/base-4.12.0.0/docs/src/GHC.Base.html#line-828](https://hackage.haskell.org/package/base-4.12.0.0/docs/src/GHC.Base.html#line-828) .
