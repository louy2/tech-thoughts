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

First, the boilerplate code.

```haskell
{-# LANGUAGE InstanceSigs #-}

newtype Reader e a = Reader (e -> a)

instance Monad (Reader e) where
    (>>=) :: Reader e a -> (a -> Reader e b) -> Reader e b
    (>>=) = undefined
```

The `InstanceSigs` pragma lets me annotate the unified type signature in the instance,
so that I am sure I have the types right before I start implementing.

First error:

> • No instance for (Applicative (Reader e))
    arising from the superclasses of an instance declaration
> • In the instance declaration for ‘Monad (Reader e)’

That's straightforward, I just need to add implementation for `Applicative` too:

```haskell
instance Applicative (Reader e) where
    pure :: a -> Reader e a
    (<*>) :: Reader e (a -> b) -> Reader e a -> Reader e b
```

This error again:

> • No instance for (Functor (Reader e))
    arising from the superclasses of an instance declaration
> • In the instance declaration for ‘Applicative (Reader e)’
