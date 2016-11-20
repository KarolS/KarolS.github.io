---
layout: post
title: "From explicit nested monads to Kleisli arrows over monad transformers"
date: 2014-04-29 01:15
categories: [programming, haskell]
---

Recently at work I had a task of developing a small tool that could display charts with progress of our data extraction over time. 

After each iteration, results of data extraction were stored in several directories and compared against a handwritten reference set.

In short, this is the directory structure:

```plain
|--+ batch1
|  |--+ correct
|  |  `--- data.txt
|  |--+ guessed
|  |  |--- data.00000145a8ab68b9.txt
|  |  |--- data.00000145a92a530f.txt
|  |  `--- data.0000014594039f5b.txt
|  |--- input1.pdf
|  |--- input2.pdf
|  `--- input3.pdf
`--+ batch2
   |--+ correct
   |  `--- data.txt
   |--+ guessed
   |  |--- data.00000145a8ab68b9.txt
   |  |--- data.00000145a92a530f.txt
   |  `--- data.0000014594039f5b.txt
   |--- input4.pdf
   |--- input5.pdf
   `--- input6.pdf
```

The hexadecimal numbers are timestamps in milliseconds since Unix epoch.

Each file consisted of records in the format:

```plain
input1.pdf value from the first file
input2.pdf value from the second file
input3.pdf value from the third file
```

Long story short, there was no difference what programming language I would write the tool in, so I picked Haskell. Let's ignore most details of the implementation. What's important for this entry, is that I implemented the following functions:

```haskell

allCorrectFiles :: [FilePath]  -> IO [FilePath]
allGuessedFiles :: [FilePath]  -> IO [(LocalTime, FilePath)]
readDataFile :: FilePath -> IO [(Entry, String)]
getAllData :: [FilePath] -> IO ([(Entry, String)], [(LocalTime, Entry, String)])

```

The initial implementation of the `getAllData` function was straightforward, yet a bit clunky:

```haskell
getAllData subDirs = do
	correctFiles <- allCorrectFiles subDirs
	guessedFiles <- allGuessedFiles subDirs
	correctData <- mapM readDataFile correctFiles
	guessedData <- forM guessedFiles $ \(t,f) ->
		x <- readDataFile f
		return $ map (\(e,s) -> (t,e,s)) x
	return (concat correctData, concat guessedData)
```

This code is quite ugly. The especially jarring were the `return $ map` combination and `concat`s in the final line.

After a while, I noticed that all the functions I call from the `getData` function are of type `a -> IO [b]`. A double monad. So, I added `transformers` library to my project and rewritten that function as:

<!-- more -->

```haskell
getAllData subDirs = do
	correctData <- runListT $ do
		f <- ListT $ allCorrectFiles subDirs
		x <- ListT $ readDataFile f
		return x	
	guessedData <- runListT $ do
		(t,f) <- ListT $ allGuessedFiles subDirs
		x <- ListT $ readDataFile f
		return (t, fst x, snd x)
	return (correctData, guessedData)
```

Now it looks way more uniform.

This was my first piece of code using monad transformers, so I will explain what's going on to everyone who was in the same situation as me before understanding it.

`IO` is a monad, so is `[]`. For some monads, if you wrap them in another monad, you still get something that can be used like a monad. One of such monads is `[]`.

For every monad `m` and type `a`, `m [a]` is a monad over `a`. But in order for Haskell to treat it as one singular monad, we need to wrap it. And that's what `ListT` is for. 

So `allCorrectFiles subDirs` returns `IO [FilePath]`, and after wrapping it with `ListT` we get `ListT IO FilePath`.

This way, we stack our monadic effects: we both iterate over elements of a list, and track impure side effects. If the `readDataFile` or `allGuessedFiles` were pure, we would use `[]` monad. If they were impure, but returned only one element, we'd use `IO` monad. By wrapping `IO [a]` into `ListT`, we can use them both.

Since the results of inner `do` blocks are now of type `ListT IO a`, we unwrap them with `runListT`.

Note that since `ListT` allowed `<-` operator to unwrap not only the `IO` monad, but also the list monad, we didn't end up with lists of lists and we no longer needed to `concat` those lists together.

The code got less ugly, but it also got a bit longer. Could I get it shorter?

The answer was yes, but only a teeny tiny bit, and involved some more complicated machinery. Arrows.

The trivial arrow instance for functions is easy to understand: `>>>` behaves like flipped function composition, and there are several combinators, like `&&&`, `***`, `first` and `second` that make it easier to work with 2-element tuples. 

Since the return value from the `allGuessedFiles` was a list of 2-element tuples in an IO monad, and I only operated on the second element before fusing them together, I decided to use arrows. Luckily, the `base` library contains definitions for monadic arrows, which are to `>>=` as trivial arrows are to `.`.

The code had to involve even more wrapping and unwrapping. Here it is:

```haskell
getAllData subDirs = do
	correctData <- runKL subDirs $ 
		kl allCorrectFiles >>> kl readDataFile
	guessedData <- runKL subDirs $ 
		kl allGuessedFiles >>> second (kl readDataFile) >>^ \(a,(b,c)) -> (a,b,c)
	return (correctData, guessedData)
	where 
		kl f = Kleisli $ ListT . f
		runKL x ar = runListT $ runKleisli ar x
```

Now I do believe I owe you some explanations.

Let's start with the definition of `kl`. `f` is of type `a -> m [b]`, where `m` is a monad. After composing it with `ListT`, we get `ListT . f` of type `a -> ListT m b`, where `ListT m` is a monad now.

`Kleisli` wraps our function into a Kleisli arrow (of type `Kleisli (ListT m) a b`), so Haskell knows that when we join arrows together, we don't want to abstract `.` but `>>=`. This way, `kl f >>> kl g` means the same as `\x -> f x >>= g`, where `f >>> g` would mean  `\x -> (g . f) x`.

`runKL` applies our arrow and unwraps the `ListT` transformer. I reversed the argument order of `runKleisli` for convenience.

So how is the `guessedData` calculated? `allGuessedFiles` returns `IO [(LocalTime, FilePath)]`. `second` makes its argument to be applied only to the second argument of the input tuple, so `second readDataFile` would be of type `(a, FilePath) -> IO[(a,(Entry, String))]`\*, but since we used `kl`, it's actually `Kleisli (ListT IO) (a, FilePath) (a,(Entry, String))`.

*\* Note: I kinda simplified here a bit and technically this type is wrong.*

The last function, which is a lambda expression, was lifted into a Kleisli arrow with the `>>^` operator. In case of Kleisli arrows, such lifting is equivalent to `\f -> (return . f)`. It turns a function of type `a->b` into a function of type `a -> m b` wrapped into a Kleisli.

I also took a peek at the `arrow-list` package, which provides its own definitions of equivalents for `kl` and `runKL`, but I decided to not use it, since it would give little value.

**So, was it worth it?**

In some sense, yes. I have learnt what monad transformers and Kleisli arrows can be used for. The main problem is that the problem I had to solve was too small for such complex machinery. The transformers made the code more uniform, which allowed application of arrows, while arrows would have made code mode compact if it was more complex.