---
layout: post_page
title:  "Reimplementing the State Monad"
date:   2015-06-21 22:04:06
categories: haskell
comments: true
---

One of the easiest ways to check if have understood something is to re implement it from scratch without looking at the original source and that is exactly what I will be trying to do in this post with the State monad. This will hopefully serve as an easy from the scratch tutorial for understanding the State Monad.

The aim of the State monad is to allow us to pretend as if there was some mutable state which we can read and modify while still having the code be pure code. So to do this, we first have to define what a state is. The most simple way of thinking about a stateful computation is something that takes in an initial state, does the computation and gives us a result and the state after the computation is done. In our definition, we call `s` to be the type of the state and `a` to be the result of the stateful computation.

```haskell
data State s a = State { runState :: s -> (a, s) }
```

This gives us a type constructor of type `State :: (s -> (a,s)) -> State s a` and the inverse function `runState :: State s a -> s -> (a, s)`. Using these 2 functions, we can start defining the instances for Functor, Applicative and Monad.

For the functor instance, we take the given stateful action and run it to get the result to which we apply `f` and wrap the whole thing inside another stateful action.

```haskell
instance Functor (State s) where
	-- fmap :: (a -> b) -> State s a -> State s b
	fmap f x = State (\s -> let (a, s') = runState x s in (f a, s'))
```

For the pure function in the applicative instance, we don't need to worry about actually doing anything with the state but rather just pass it out unmodifed in the result and yield the value given to us. As for the `<*>` function, we will need to unwrap both the given arguments to get the `f` and the `a` and finally wrap it back up using `State`. In this function, we remember to thread the new state from unwrapping the first computation and use that as the initial state for unwrapping the second computation.

```haskell
instance Applicative (State s) where
	-- pure :: a -> State s a
	pure x = State (\s -> (x, s))
	-- (<*>) :: State s (a -> b) -> State s a -> State s b
	x <*> y = State (\s -> let
								(f, s')  = runState x s
								(a, s'') = runState y s'
						   in
						   		(f a, s''))
```

Finally for the Monad instance, we can just alias pure as the definition for return. The bind operator in this case would refer to the idea of running a stateful computation, using the result to create another stateful computation while implicitly threading the state inbetween these two functions so that we don't have to do it manually.

```haskell
instance Monad (State s) where
	-- return :: a -> State s a
	return = pure
	-- (>>=) :: State s a -> (a -> State s b) -> State s b
	x >>= y = State (\s -> let (a, s') = runState x s in runState (y a) s')
```

Finally, we can define the 3 helper functions that are defined in `Control.Monad.State` for us, which are `get`, which returns the current state, `set`, which changes the current state while yielding `()` as the result, and `modify`, which takes in a function that can modify the current state while yielding the `()` as well.

```haskell
get :: State s s
get =  State (\s -> (s, s))

put :: s -> State s ()
put x = State (\s -> ((), x))

modify :: (s -> s) -> State s ()
modify f = State (\s -> ((), f s))
```

To get an idea of how useful doing things in the state monad might be, we can try and solve the nested set problem using the tradition pure function approach and try it again using State monad to see how much easier it will be. We can first define a tree and a labelled tree types.

```haskell
data Tree a = EmptyTree | Node a [Tree a]
data Label a = Label Int Int a

normalAnnotateTree :: Tree a -> Tree (Label a)
normalAnnotateTree x = let (a, _) = go x 1 in a
	where
	go :: Tree a -> Int -> (Tree (Label a), Int)
	go EmptyTree count = (EmptyTree, count)
	go (Node a as) count = let
								lCounter = count
								(bs, count') = foldr go' ([], count+1) as
								rCounter = count'
							in (Node (Label lCounter rCounter a) bs, count' + 1)
		where
		go' :: Tree a -> ([Tree (Label a)], Int) -> ([Tree (Label a)], Int)
		go' tree (acc, count) = let (tree', count') = go tree count in (tree':acc, count')
```

Doing the same thing using the State monad would be as simple as:

```haskell
stateAnnotateTree :: Tree a -> Tree (Label a)
stateAnnotateTree x = let (a, _) = runState (go x) 1 in a
	where
	go :: Tree a -> State Int (Tree (Label a))
	go EmptyTree   = return EmptyTree
	go (Node a as) = do
		lCounter <- get
		modify (+1)
		bs <- mapM go as
		rCounter <- get
		modify (+1)
		return $ Node (Label lCounter rCounter a) bs
```

Hopefully this has given you a actionable understanding of the State Monad and some ideas on how to use it!
