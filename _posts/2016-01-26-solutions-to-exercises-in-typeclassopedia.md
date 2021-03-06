---
layout: post
title: "类型宝典（Typeclassopedia）题解"
category: summary
tags: [Haskell, Typeclassopedia]
---

最近在重新学习Haskell，看完LYAHFGG之后，想加深一下对Applicative，Monad啥的理解，找了这个叫Typeclassopedia的文章来看。断断续续看了两个星期左右，把里面的习题也做得差不多了。下面列一下题解（不保证完全，也不保证正确）。

Functor 函子
----

要为一个类型`f`实现一个函子实例，这个类型必须具有`* -> *`这样的kind。也就是说`f`必须再接受一个类型`a`才能成为一个完整的类型。而函子的`fmap`就是一个将`a`映射到另外一个类型`b`的映射。所以`fmap`具有这样的类型签名：

```haskell
fmap :: (a -> b) -> f a -> f b
```

### 实现`Either e`与`((->) e)`的函子实例


首先我们知道`Either l r`是一个完整的类型（即`Either`的kind是`* -> * -> *`），那么要实现关于`Either`的函子实例，必须固定最左侧的`*`，也就是说我们只能实现关于`Either l`的函子实例。更具体来说，我们实现的函子实例可以将`Either l a`变换为`Either l b`。

```haskell
data MyEither = MyLeft e | MyRight a
instance Functor (MyEither e) where
  _ `fmap` MyLeft e = MyLeft e
  f `fmap` MyRight a = MyRight $ f a
```

同样的道理`((->) e)`的函子实例可以将`(e -> a)`映射为`(e -> b)`，那么它的实现就很直观了。

```haskell
instance ((->) e) where
  f `fmap` g = \e -> f (g e)
```

如果要写得吊一点：

```haskell
instance ((->) e) where
  fmap = (.)
```

### 实现`((,) e)`和`data Pair a = Pair a a`的函子实例，并解释他们的异同

与上面一样，`(,)`的kind是`* -> * -> *`，那么只能实现`((,) e)`的函子实例，也就是说只能实现`(e, a)`到`(e, b)`的映射，而Pair的kind已经是`* -> *`了，所以可以实现`Pair a a`到`Pair b b`的函子实例。

```haskell
newtype MyTuple e a = MyTuple { getMyTuple :: (e, a) }

instance Functor (MyTuple e) where
  f `fmap` MyTuple (e, a) = MyTuple (e, f a)


data Pair a = Pair a a

instance Functor Pair where
  f `fmap` Pair a b = Pair (f a) (f b)
```

### 实现以下类型的函子实例

```haskell
data ITree a = Leaf (Int -> a)
             | Node [ITree a]
```

Trivial:

```haskell
instance Functor ITree where
  f `fmap` Leaf t = Leaf $ f `fmap` t
  f `fmap` Node xs = Node $ map (fmap f) xs
```

*如果你知道`[]`也是一个函子，那么`map`实际上可以写成`fmap`。

### 说明两个函子的复合也是一个函子

首先定义出两个函子的复合这个类型，要注意`FCompose`这个构造器是一个一元函数，而不是三元函数：

```haskell
data FCompose f1 f2 x = FCompose (f1 (f2 x))
	deriving Show
```

当`f1`和`f2`是函子时，

```haskell
instance (Functor f1, Functor f2) => Functor (FCompose f1 f2) where
  f `fmap` FCompose t = FCompose $ fmap f `fmap` t
  -- = FCompose $ fmap (fmap f) t = FCompose $ (fmap . fmap) f t
```

对于`FCompose f1 f2`，如果，`fmap`方法的类型应该是

```haskell
fmap :: (Functor f1, Functor f2) => (a -> b) -> FCompose f1 f2 a -> FCompose f1 f2 b
```

也就是说如果我们能把`f :: a -> b`映射到`f1 (f2 a)`上并得到`f1 (f2 b)`，那么两个函子的复合就仍是一个函子。所以，

1. 首先利用`fmap :: (a -> b) -> f2 a -> f2 b`把`f :: a -> b`提升（lift）到`f2`上得到`f' :: f2 a -> f2 b`（柯里化）；
2. 这样就可以用`f1`的`fmap :: (f2 a -> f2 b) -> f1 (f2 a) -> f1 (f2 b)`将`f'`提升为`f'' :: f1 (f2 a) -> f1 (f2 b)`（仍然是柯里化）；
3. 接下来将`f''`应用到`f1 (f2 a)`上得到`f1 (f2 b)`。

这说明两个任意函子的复合确实仍然是一个函子。它告诉我们，我们可以利用各种函子复合出任意复杂的新函子。

实际上，函数式编程的一个重要的部分就是利用各种复杂结构间的复合，降低整个程序的复杂度，即尽量将程序的复杂度转移到结构之中，由于结构是通用的，并且充分研究过的，使得程序员可以更加关注于问题的业务上的解决。也就是说，Typeclassopedia这篇文章里介绍的各种数据类型其实就是函数式编程中的**设计模式**。

要得到关于多个函子的复合的`fmap`方法，只需要将多个`fmap`复合。例如两个函子复合的情况，可以是`(fmap . fmap) f t`。

Applicative 应用函子
----

### 证明以下等式

```haskell
pure f <*> x = pure (flip ($)) <*> x <*> pure f
```

从右边入手

```haskell
    pure (flip ($)) <*> x <*> pure f
  = pure (flip ($)) <*> pure x' <*> pure f where x = pure x'
  = pure (flip ($) x' f)  -- (Applicative的同态性)
  = pure (f x')  -- (f $ y = f y)
  = pure f <*> pure x'
  = pure f <*> x
```

其实仔细看了之后会发现这个等式其实很无聊。

### 实现`Maybe`的Applicative实例

Trivial:

```haskell
data MyMaybe a = MyJust a | MyNothing

instance Functor MyMaybe where
  f `fmap` MyJust x = MyJust $ f x
  f `fmap` MyNothing = MyNothing

instance Applicative MyMaybe where
  pure x = MyJust x
  MyJust f <*> x = fmap f x
  MyNothing <*> x = MyNothing
```

我们接下来来检查上面的实现是否满足Applicative的规定。

恒等律：

```haskell
    pure id <*> MyJust 42
  = MyJust id <*> MyJust 42
  = fmap id (MyJust 42)
  = MyJust (id 42)
  = MyJust 42
  
    pure id <*> MyNothing
  = MyJust id <*> MyNothing
  = fmap id MyNothing
  = MyNothing
```

同态律：

```haskell
    pure f <*> pure x
  = MyJust f <*> MyJust x
  = fmap f (MyJust x)
  = MyJust (f x)
  = pure (f x)
```

交换律：

```haskell
-- 若 u = MyJust u'，
    u <*> pure y
  = MyJust u' <*> pure y
  = MyJust u' <*> MyJust y
  = fmap u' (MyJust y)
  = MyJust (u' y)
  = pure (u' y)
  = pure (u' $ y)
  = pure ((\x -> x $ y) u')
  = pure (($ y) u')
  = pure ($ y) <*> pure u'  -- 由同态律得
  = pure ($ y) <*> u

-- 若 u = MyNothing
    u <*> pure y
  = MyNothing <*> pure y
  = MyNothing

    pure ($ y) <*> u
  = fmap ($ y) MyNothing
  = MyNothing
  
-- => u <*> pure y = pure ($ y) <*> u
```

复合律：

```haskell
    u <*> (v <*> w)
  = u <*> (Maybe v' <*> Maybe w')  -- 由(<*>)的定义得
  = u <*> Maybe (v' w')
  = Maybe u' <*> Maybe (v' w')
  = Maybe (u' (v' w'))
  = Maybe ((u' . v') w')  -- 由(.)的定义得
  = Maybe ((.) u' v' w')
  = Maybe ((.) u' v') <*> Maybe w'  -- 由同态律得
  = Maybe ((.) u') <*> Maybe v' <*> w  -- 仍然由同态律得
  = Maybe (.) <*> Maybe u' <*> v <*> w
  = Maybe (.) <*> u <*> v <*> w
  = pure (.) <*> u <*> v <*> w
```

### 确定`ZipList`的Applicative实例里`pure`的正确定义，能够满足Applicative性质的实现有且仅有一个

我们已经知道`(<*>)`是使用`zipWith`函数实现的，现在考虑第一条性质`pure id <*> v = v`，如果`v`是一个有3个元素的`ZipList`，那么`pure id`就应该返回一个有**至少**三个`id`的`ZipList`，使得`zipWith`函数能够将`v`中的元素全部利用起来。到这里我们可以很轻松的得出`pure`的定义了

```haskell
pure = ZipList . repeat
```

我们稍微检查一下这个实现的性质：

恒等律：

```haskell
    pure id <*> ZipList xs
  = ZipList (repeat id) <*> ZipList xs
  = ZipList (zipWith ($) (repeat id) xs)
  = ZipList xs  -- 由Haskell常识得
```

同态律：

```haskell
    pure f <*> pure x
  = ZipList (repeat f) <*> ZipList (repeat x)
  = ZipList (zipWith ($) (repeat f) (repeat x))
  = ZipList (repeat (f $ x))  -- 由Haskell常识得
  = ZipList (repeat (f x))
  = ZipList . repeat (f x)  -- 由(.)的定义得
  = pure (f x)
```

剩下的规则就不一一推导了。

### 利用`unit`和`(**)`实现`pure`与`(<*>)`，再反过来实现他们

考虑在这个情况下的`pure`、`unit`、`(**)`和`fmap`的类型签名（已知一个Monoidal是一个函子）：

```haskell
pure :: Monoidal m => a -> m a
unit :: Monoidal m => m ()
(**) :: Monoidal m => m a -> m b -> m (a, b)
fmap :: Monoidal m => (t -> a) -> m t -> m a
```

可以发现`fmap`的返回值类型就是我们关心的类型（`fmap`的返回值类型与`pure`的相同）。但是此时`fmap`中的`t`从哪来呢。这时我们可以注意到Monoidal的`unit`方法恒定返回一个`m ()`类型的值，那么如果将`t`特化成`()`，就有了

```haskell
fmap :: Monoidal m => (() -> a) -> m () -> m a
```

这样就清楚了，先构造出一个接受一个任意参数的函数，然后将这个函数`fmap`到`unit`上

```haskell
pure x = fmap (\_ -> x) unit
-- or
pure x = fmap (const x) unit
```

再考虑`(<*>)`与其它已知方法的类型签名：

```haskell
(<*>) :: Monoidal m => m (a -> b) -> m a -> m b
f <*> x = ???

unit :: Monoidal m => m ()
(**) :: Monoidal m => m a -> m b -> m (a, b)
fmap :: Monoidal m => (a -> b) -> m a -> m b
```

可以发现`(<*>)`与`fmap`除了第一个参数类型不是特别相同，其它的是一样的。所以思路仍然是构造出一个供`fmap`使用的函数。同时我们发现`(**)`是Monoidal中可以与两个类型交互的方法，说明我们在构造上面说的函数中会使用到它。现在，已知量是`f :: m (a -> b)`，`x :: m a`，我们首先可以尝试`f ** x`，这样我们会得到`f ** x :: m (a -> b, a)`，而这样的类型很容易让人联想到它的计算结果的类型就是`m b`！因此如果将`fmap`中的`m a`特化成`m (a -> b, a)`，那么整个`fmap`的类型签名会变为：

```haskell
fmap :: ((a -> b, a) -> b) -> m (a -> b, a) -> m b
```

问题也就变为，如何构造出一个函数使得其类型为`(a -> b, a) -> b`，这样就很简单了

```haskell
mf <*> mx = fmap (\(f, x) -> f x) (mf ** mx)
```

这里另外介绍一个技巧（或者说是idiom）：`\(f, x) -> f x`可以写成`uncurry id`。

很脏。首先来检查一下`uncurry`的类型：

```haskell
uncurry :: (a -> b -> c) -> (a, b) -> c
```

因为`(->)`是右结合的，所以上面的类型也可以写成

```haskell
uncurry :: (a -> d) -> (a, b) -> c
  where d :: b -> c
```

也就是说如果你给我一个返回另外一个一元函数的一元函数，并且给我两个参数，我能帮你计算出结果。如果要把`id :: a -> a`提供给它，那么`id`的参数必须是一个一元函数，也就是说它必须变形为`id :: (b -> c) -> (b -> c)`，那么就有了

```haskell
uncurry id :: (b -> c, b) -> c
```

这跟我们上面推理出的`(a -> b, a) -> b`的类型恰好一样。

用`pure`来实现`unit`很简单：

```haskell
unit :: Applicative f => f ()
unit = pure ()
```

用Applicative方法来实现`(**)`，首先检查类型

```haskell
pure :: Applicative f => a -> f a
(<*>) :: Applicative f => f (a -> b) -> f a -> f b
(**) :: Applicative f => f a -> f b -> f (a, b)
```

我们可以观察出`(<*>)`与`(**)`的类型很相似，而且要使用`(<*>)`来获得`(**)`的结果。那么不妨将`(<*>)`的返回值类型特化为`f (a, b)`，这样`(<*>)`的整个类型就是：

```haskell
(<*>) :: Applicative f => f (t -> (a, b)) -> f t -> f (a, b)
```

接下来就是如何构造这样一个函数`t -> (a, b)`使得我们可以利用已知量`f a`与`f b`。很自然的想法是构造一个这样的二元函数`\a b -> (a, b)`，这样我们就可以

```haskell
(**) :: f a -> f b -> f (a, b)
fa ** fb = pure (\a b -> (a, b)) <*> fa <*> fb
```

或者

```haskell
fa ** fb = pure (,) <*> fa <*> fb
```

看到这，你就会觉得上面的推导其实是复杂化了。我们可以这样理解：将元组构造器`(,)`通过`pure`提升到Applicative中，然后将两个已经在Applicative里的已知量通过`(<*>)`应用到`pure (,)`上。

Monad 单子
----

### 实现列表的单子实例

要注意到这里的列表指的并不是`ZipList`。

```haskell
instance Functor [] where
  f `fmap` [] = []
  f `fmap` (x:xs) = (f x):(fmap f xs)

instance Applicative [] where
  pure x = [x]
  fs <*> xs = concat $ map (\f -> fmap f xs) (fmap ($) fs)
  -- 首先通过(fmap ($) fs)取得[($) f]，然后将每一个(($) f)通过fmap应用于xs，最后
  -- concat一下来满足类型约束。

instance Monad [] where
  xs >>= f = concat $ pure f <*> xs
```

接下来检查一下单子所要满足的规律

左恒等律：

```haskell
    return x >>= f
  = pure x >>= f
  = [x] >>= f
  = concat $ [f] <*> [x]
  = concat (concat $ map (\f -> fmap f [x]) (fmap ($) [f]))
  = concat (concat $ map (\f -> fmap f [x]) [(f $)])
  = concat (concat $ [fmap (f $) [x]])
  = concat (concat [[f x]])
  = concat [f x]
  = f x
```

右恒等律：

```haskell
    xs >>= return
  = xs >>= pure
  = concat $ pure pure <*> xs
  = concat $ [pure] <*> xs
  = concat (concat $ map (\f -> fmap f xs) (fmap ($) [pure]))
  = concat . concat $ map (\f -> fmap f xs) [(pure $)]
  = concat . concat $ [fmap (pure $) xs]
  = concat . concat $ [fmap (pure $) [x1, x2, ..., xk, ...]]
  = concat . concat $ [[(pure $ x1), (pure $ x2), ..., (pure $ xk), ...]]
  = concat . concat $ [[(pure x1), (pure x2), ..., (pure xk), ...]]
  = concat . concat $ [[[x1], [x2], ..., [xk], ...]]
  = concat $ [[x1], [x2], ..., [xk], ...]
  = [x1, x2, ..., xk, ...]
  = xs
```

结合律：

```haskell
    xs >>= (\y -> f y >>= g)
  = xs >>= (\y -> concat $ pure g <*> f y)
  = xs >>= (\y -> concat $ [g] <*> f y)
  = xs >>= (\y -> concat (concat $ map (\h -> fmap h (f y)) (fmap ($) [g])))
  = xs >>= (\y -> concat . concat $ map (\h -> fmap h (f y)) [(g $)])
  = xs >>= (\y -> concat . concat $ [fmap (g $) (f y)])
  = xs >>= (\y -> concat . concat $ [[g fy1, g fy2, ..., g fyk, ...]])
  = xs >>= (\y -> concat [g fy1, g fy2, ..., g fyk, ...])
  = xs >>= (\y -> concat [[gfy11, gfy12, ..., gfy1k, ...],
                          [gfy21, gfy22, ..., gfy2k, ...],
						  ...,
						  [gfyk1, gfyk2, ..., gfykk, ...],
						  ...,])
  = xs >>= (\y -> [gfy11, gfy12, ..., gfy21, gfy22, ... gfyk1, gfyk2, ...])

let gfypq = [gfy11, gfy12, ..., gfy21, gfy22, ..., gfyk1, ...],

    xs >>= (\y -> f y >>= g)
  = xs >>= (\y -> gfypq)
  = concat $ [\y -> gfypq] <*> xs
  = concat . concat $ map (\h -> fmap h xs) (fmap ($) [\y -> gfypq])
  = concat . concat $ map (\h -> fmap h xs) [((\y -> gfypq) $)]
  = concat . concat $ [fmap ((\y -> gfypq) $) xs]
  = concat . concat $ [fmap ((\y -> gfypq) $) [x1, x2, ..., xk, ...]]
  = concat . concat $ [[gfx1pq, gfx2pq, ..., gfxkpq, ...]]
  = concat [[gfx111, gfx112, ..., gfx121, gfx122, ...],
            [gfx211, gfx212, ..., gfx221, gfx222, ...],
			...,
			[gfxk11, gfxk12, ..., gfxk21, gfxk22, ...],
			...]
  = [gfx111, gfx112, ..., gfx121, gfx122, ..., gfxk11, gfxk12, ...]
  
    (xs >>= f) >>= g
  = (concat $ [f] <*> xs) >>= g
  = (concat . concat $ map (\f -> fmap f xs) [(f $)]) >>= g
  = (concat . concat $ [fmap (f $) xs]) >>= g
  = (concat . concat $ [[f x1, f x2, ..., f xk, ...]]) >>= g
  = (concat [f x1, f x2, ..., f xk, ...]) >>= g
  = (concat [[fx11, fx12, ..., fx1k, ...],
             [fx21, fx22, ..., fx2k, ...],
			 ...,
			 [fxk1, fxk2, ..., fxkk, ...]]) >>= g
  = [fx11, fx12, ..., fx1k, ..., fx21, fx22, ..., fx2k, ...] >>= g

let fxpq = [fx11, fx12, ..., fx1k, ..., fx21, fx22, ..., fx2k, ...],

    (xs >>= f) >>= g
  = fxpq >>= g
  = concat $ [g] <*> fxpq
  = concat . concat $ map (\h -> fmap h fxpq) [(g $)]
  = concat . concat $ [fmap (g $) fxpq]
  = concat . concat $ [[g fx11, g fx12, ..., g fx1k, ..., g fx21, ...]]
  = concat [g fx11, g fx12, ..., g fx1k, ..., g fx21, g fx22, ...]
  = concat [[gfx111, gfx112, ...],
            [gfx121, gfx122, ...],
			...,
			[gfxpq1, gfxpq2, ...],
			...]
  = [gfx111, gfx112, ..., gfx121, gfx122, ..., gfxpq1, gfxpq2, ...]
  = xs >>= (\y -> f y >>= g)
```

如果用更简单的`(>>=)`实现（比如`base`包里给出的使用列表解析的实现），结合律的证明可能不会这么复杂。

### 实现`((->) e)`的单子实例

`((->) e)`的函子实例前面已经实现过，这里不再重复。

首先来实现`((->) e)`的Applicative实例，`pure`很trivial：

```haskell
pure x = \_ -> x
-- or
pure = const
```

接下来看`(<*>)`特化为`((->) e)`的类型：

```haskell
(<*>) :: ((->) e (a -> b)) -> ((->) e a) -> ((->) e b)
-- i.e.
(<*>) :: (e -> (a -> b)) -> (e -> a) -> (e -> b)
-- 由于(->)是右结合的
(<*>) :: (e -> a -> b) -> (e -> a) -> (e -> b)
```

沿着这个类型，它的具体实现就很清楚了，`f :: e -> a -> b`，`g :: e -> a`，我们要得到一个`e -> b`类型的返回值，首先要做的是创建一个接受一个`e`的函数，即`\e -> ???`。然后使用`g e`获得一个`a`类型的值。根据之前的推理，将这个`a`类型的值与`e`一起作用于`f`可以得到一个`b`，即

```haskell
f <*> g = \e -> f e (g e)
```

如果你看Haskell的源码，你会发现其实它是这么实现的：

```haskell
(<*>) :: (e -> a -> b) -> (e -> a) -> (e -> b)
-- 由于(->)右结合
(<*>) :: (e -> a -> b) -> (e -> a) -> e -> b
```

这也就告诉我们，上面的式子可以这么写：

```haskell
(<*>) f g e = f e (g e)
```

下面实现`((->) e)`的单子实例：它的`(>>=)`类型应该是

```haskell
(>>=) :: ((->) e a) -> (a -> ((->) e b)) -> ((->) e b)
-- =>
(>>=) :: (e -> a) -> (a -> (e -> b)) -> (e -> b)
-- =>
(>>=) :: (e -> a) -> (a -> e -> b) -> e -> b
```

那么

```haskell
instance Monad ((->) e) where
  (>>=) f g e = g (f e) e
  -- or f >>= g = \e -> g (f e) e
```

接下来检查这个实现的性质：

左恒等律：

```haskell
    return f >>= g
  = pure f >>= g
  = (\_ -> f) >>= g
  = \e -> g ((\_ -> f) e) e
  = \e -> g f e
  = g f
```

右恒等律：

```haskell
    f >>= return
  = f >>= pure
  = \e -> pure (f e) e
  = pure (f e)
  = \e -> (\_ -> f e) e
  = \e -> f e
  = f
```

结合律：

```haskell
    f >>= (\t -> g t >>= h)
  = f >>= (\t -> (\e -> h (g t e) e))
  = f >>= (\t e -> h (g t e) e)  -- uncurry
  = \e' -> (\t e -> h (g t e) e) (f e') e'
  = \e' -> (\e -> h (g (f e') e) e) e'
  = \e' -> h (g (f e') e') e'
  = \e' -> h ((\k -> g (f k) k) e') e'
  = (\k -> g (f k) k) >>= h  -- \e' -> h (K e') e' = K >>= h
  = (f >>= g) >>= h          -- 同理
```

### 实现以下数据类型的函子与单子

```haskell
-- Functor f =>
data Free f a = Var a
              | Node (f (Free f a))
```

既然我们在实现关于`Free f`的函子实例，我们就可以假设它是一个函子。那么对于`Free f a`，它的`fmap`的类型是

```haskell
fmap :: Functor f => (a -> b) -> Free f a -> Free f b
```

对于`Var a`，`fmap`很简单

```haskell
g `fmap` Var x = Var $ g x
```

对于`Node t`，我们可以观察出来，`a`实际上是裹在两层函子里的，即首先它裹在`Free f a`这一个函子里，同时这个函子又裹在`f`这个函子里，那么根据之前的结论，去访问裹在两层函子里的值，我们用`fmap . fmap`，即

```haskell
g `fmap` Node t = Node $ (fmap . fmap) g t
```

接下来实现`Free f a`的Applicative实例：

```haskell
pure :: Functor f => a -> Free f a
-- =>
pure = Var

(<*>) :: Functor f => Free f (a -> b) -> Free f a -> Free f b
-- =>
Var f <*> free = fmap f free
Node t <*> free = ???
```

接下来考虑`Node t`的情况，这里的`t`实际上是一个`f (Free f (a -> b))`（观察它的`data`定义的第二句`Node (f (Free f a))`），即

```haskell
t :: Functor => f t'
  where t' :: Free f (a -> b)
```

而`free`则是一个`Free f a`，我们要解决的问题就是将`a -> b`应用到`a`上。我们最自然的想法，是递归地利用`(<*>)`使得`t`中的`t' :: Free f (a -> b)`能够与`Free f a`作用，即我们想要得到`t' <*> free`。

然而`t'`我们是不能直接得到的，它还裹在一个`f`中。回想起`fmap`的另一种理解方式：它将一个函数提升至对应的函子上下文中，即：

```haskell
fmap :: (a -> b) -> f a -> f b
fmap f :: f a -> f b
```

那么我们可以利用函子`f`的`fmap`将`(<*> free)`提升至函子`f`中，得到一个提升过的函数，再将`t`应用于这个函数上。更具体地，我们可以特化这其中的`a`为`Free f c`，`b`为`Free f d`，得到下面的类型

```haskell
fmap :: (Free f c -> Free f d) -> f (Free f c) -> f (Free f d)
-- 将c换为a -> b，d换为b
fmap :: (Free f (a -> b) -> Free f b) -> f (Free f (a -> b)) -> f (Free f b)

-- 而
(<*>) :: Functor f => Free f (a -> b) -> Free f a -> Free f b
(<*> free) :: Functor f => Free f (a -> b) -> Free f b 

-- 则
fmap (<*> free) :: f (Free f (a -> b)) -> f (Free f b)
```

这样，我们就可以把`t`应用于上面得到的式子：

```haskell
Node t <*> free = Node $ fmap (<*> free) t
```

接下来实现`Free f a`的单子实例：

```haskell
instance Monad (Free f) where
  Var a >>= f = f a
  Node t >>= f = ...
```

关于`Node t`，同样的

```haskell
t :: Functor f => f t'
  where t' :: Free f a

f :: Functor f => a -> Free f b
```

而`(>>=)`的类型为：

```haskell
(>>=) :: Functor f => Free f a -> (a -> Free f b) -> Free f b
```

则我们自然地想到，可以递归地利用`(>>=)`，来让`t' >>= f`，然而`t'`仍然裹在一层函子`f`中，那么简单地利用函子`f`的`fmap`将`(>>= f)`提升至函子`f`中即可，即

```haskell
Node t >>= f = Node $ fmap (>>= f) t
```

至此，`Free f a`的单子实例的推导结束。

可以看到，在上面的推导中，我们打了大量的类型运算的草稿，并且通过这些类型，我们能够对具体的实现有一种直观上的感受。通过类型运算来指引具体的实现在Haskell中是很重要的一个技巧。

接下来检查该实现是否符合各个单子律：

左恒等律：

```haskell
    return (Var a) >>= k
  = pure (Var a) >>= k
  = Var (Var a) >>= k
  = k (Var a)

	return (Node t) >>= k
  = Var (Node t) >>= k
  = k (Node t)
```

右恒等律：

```haskell
    Var a >>= return
  = Var a >>= Var
  = Var a

    Node t >>= return
  = Node t >>= Var
  = Node $ fmap (>>= Var) t

-- 若 t = f (Var a)，则
	Node t >>= return
  = Node $ fmap (>>= Var) t
  = Node $ fmap (\_ -> Var a >>= Var) t
  = Node $ t  -- 请通过类型运算让自己相信这一步是正确的
  = Node t

-- 若 t = f (Node u)，则
    Node t >>= return
  = Node $ fmap (\_ -> Node u >>= Var) t

-- 递归至第一步，如果t有穷（其递归结构终止于Var a）,则
    Node t >>= return
  = Node $ t  -- 请通过类型运算让自己相信这一步是正确的
  = Node t
```

结合律：太复杂，在简单的方法里我证明不出来。

### 利用`fmap`和`join`实现`(>>=)`

首先检查他们的类型：

```haskell
fmap :: Functor m => (x -> y) -> m x -> m y
join :: Monad m => m (m a) -> m a
(>>=) :: Monad m => m a -> (a -> m b) -> m b
```

然后检查已知量：

```haskell
m >>= f :: Monad m => m b
  where m :: m a
        f :: a -> m b
```

看到`f :: a -> m b`和`fmap`的第一个参数，不妨将`y`特化为`m b`，那么就有

```haskell
fmap :: Functor m => (x -> m b) -> m x -> m (m b)
```

看到这样的返回值类型就能很自然地将`join`联想起来了：

```haskell
m >>= f = join (fmap f m)
```

### 利用`(>>=)`和`return`实现`join`和`fmap`

类型就不再赘述。已知量如下：

```haskell
join x :: m a
  where x :: m (m a)
  
fmap f m :: m b
  where f :: a -> b
        m :: m a
```

`join`很简单：

```haskell
join x = x >>= \x' -> x'
-- i.e.
join x = x >>= id
```

观察到`f`与`(>>=)`中的第二个参数的类型相似，其可以用`return . f`来表示，那么`fmap`也很简单：

```haskell
fmap f m = m >>= (return . f)
```

### 证明关于`(>=>)`的规则与一般的单子规则是等价的

其中`(>=>)`的定义如下：

```haskell
g >=> h = \x -> g x >>= h
```

左恒等律：

```haskell
    return >=> g
  = \x -> return x >>= g
  = \x -> g x  -- 通常的左恒等律
  = g
```

即证明了，当且仅当通常的左恒等律成立时，关于`(>=>)`的左恒等律成立。也就说明这条左恒等律与通常的左恒等律等价。

右恒等律：

```haskell
    g >=> return
  = \x -> g x >>= return
  = \x -> g x  -- 通常的右恒等律
  = g
```

结合律：

```haskell
    (g >=> h) >=> k
  = (\x -> g x >>= h) >=> k
  = \y -> (\x -> g x >>= h) y >>= k
  = \y -> g y >>= h >>= k
  = \y -> g y >>= (\x -> h x >>= k)  -- (>>=)的结合律
  = \y -> g y >>= (h >=> k)  -- (>=>)的定义
  = g >=> (h >=> k)  -- (>=>)的定义
```

### 给定`distrib :: N (M a) -> M (N a)`，实现以下函数

```haskell
join :: M (N (M (N a))) -> M (N a)
```

如果你熟悉do记法，这个就比较简单了：

```haskell
join mnmna = do nmna <- mnmna
                nna <- distrib nmna  -- 得到 M (N (N a))
				na <- join nna
				return na
```

Foldable 折子(?)
----

### `foldMap . foldMap`是什么类型？有什么用？

在GHCi里使用`:t <expr>`可以查看`<expr>`的类型。

`foldMap . foldMap`可以折叠两层嵌套的不同折子到一个幺半群（Monoid）上。

### 实现`toList :: Foldable f => f a -> [a]`

既然我们已经知道`[]`是一个幺半群，那么我们可以使用`foldMap`将`f`折叠为一个幺半群。已知`foldMap`的类型是：

```haskell
foldMap :: (Foldable f, Monoid m) => (a -> m) -> f a -> m

toList :: Foldable f => f a -> [a]
toList = foldMap pure
```

### 实现`concat`，`concatMap`，`and`，`or`，`any`，`all`，`sum`，`product`等函数，并弄明白如何联合使用折子与幺半群来优雅地实现它们

我们总是想优先使用`foldMap`，因为它需要的参数少（不需要初始值）。当发现`foldMap`对于给定的问题不合适之后才考虑`foldr`或者`foldl`以及他们的变体。

```haskell
concat :: Foldable f => f [a] => [a]
concat = foldMap id  -- 因为f [a]里的[a]已经是一个Monoid了，所以直接id就行了

concatMap :: Foldable f => (a -> [b]) -> f a -> [b]
concatMap = foldMap  -- concatMap实际上是foldMap里Monoid m特化为[]的一个特殊情况

and :: Foldable f => f Bool -> Bool
and = foldr (&&) True

or :: Foldable f => f Bool -> Bool
or = foldr (||) False
```

`any`, `all`, `sum`, `product`这几个函数的特点是，它们操作的对象都可以在多种运算符和空值下构成幺半群，比如整数可以在`(+)`与0下构成一个幺半群，也可以在`(*)`与1下构成一个幺半群。所以如果我们使用`newtype`来构建新的类型，并实现它们的幺半群实例。这样就可以使用`foldMap`来优雅地实现上面说的函数了。（当然如果直接用`foldr`或者`foldl`也不是不可以。但是`newtype`出来的类型不光在这里有用，在别的地方也有可能用得到，所以这里我们用使用`newtype`的写法。）

```haskell
newtype Any = Any { getAny :: Bool }

instance Monoid Any where
  mempty = Any False
  Any a `mappend` Any b = Any $ a || b

any :: Foldable f => (a -> Bool) -> f a -> Bool
any f = getAny . foldMap (Any . f)


newtype All = All { getAll :: Bool }

instance Monoid All where
  mempty = All True
  All a `mappend` All b = All $ a && b

all :: Foldable f => (a -> Bool) -> f a -> Bool
all f = getAll . foldMap (All . f)

newtype Sum a = Sum { getSum :: a }

instance Num a => Monoid (Sum a) where
  mempty = Sum 0
  Sum a `mappend` Sum b = Sum (a + b)

sum :: (Num a, Foldable f) => f a -> a
sum = getSum . foldMap Sum

-- product不再赘述
```

接下来看看`maximumBy`的类型：

```haskell
maximumBy :: Foldable f -> (a -> a -> Ordering) -> f a -> a
```

我们已经知道`Ordering`是一个幺半群了，看到幺半群就会自然地想到使用`foldMap`，然而`foldMap`中计算出来的幺半群就是它的结果，但是`maximumBy`的结果却不是一个幺半群。所以这里使用`foldMap`可能不太好。我们使用`foldr1`来实现它，因为`foldr1`是`foldr`的特化版本，它不需要使用者提供初始值。

```haskell
maximumBy f = foldr1 (\x acc -> case f x acc of
	                     LT -> acc
		                 EQ -> acc
						 GT -> x)
```

`elem x xs`返回一个`Bool`。当`xs`中至少有一个`x`时，它返回`True`，否则返回`False`。这让我们联想到构造一个这样的函数`\a -> a == x`并使用`or`。然而`or`需要的是`f Bool`，这里仅仅保证了`f`是一个折子，并不保证它是一个函子，所以我们无法将`f a`映射至`f Bool`。（你当然可以`or $ map (== x) (toList xs)`，但是我们有更好的方法。）

换个思路：当`xs`中至少有一个`x`时，这句话也就是`xs`中的任意一个元素等于`x`时，也就是说我们可以使用`any`！

```haskell
elem :: (Eq a, Foldable f) => a -> f a -> Bool
elem x = any (== x)

-- 源码中的实现更简单 elem = any . (==)
```

`find`的返回值类型是`Maybe a`，当且仅当`a`是一个幺半群的时候`Maybe a`是一个幺半群。然而`find`的类型约束并不保证`a`是一个幺半群。所以这里我们尝试使用`foldr`，并且从`Nothing`开始折叠。

```haskell
find :: Foldable f => (a -> Bool) -> f a -> Maybe a
find pred = foldr (\x acc -> if pred x then Just x else acc) Nothing
```

这个实现会遍历所有可能的元素，返回最后一个遍历到的符合条件的元素。

源码中利用了一个叫`First a :: { getFirst :: Maybe a }`的幺半群结构。这个结构说在`mappend`的时候，如果左边的`First`的值是Nothing，那么它的结果是`mappend`右边的值，否则这个`mappend`结果是左边的值。也就是说，如果利用这个幺半群与`foldMap`，我们可以返回第一个符合条件的元素（虽然不一定会在那个时候就返回）：

```haskell
find pred = getFirst . foldMap $ \x -> First (if pred x then Just x else Nothing)
```

Traversable 遍子(?)
----

### 至少有两种自然的方式把一个由`[]`组成的树转换为一个由树组成的`[]`，它们是哪两种？

参照`[]`的两种意义：一种代表不确定的计算结果，一种代表并列的、有序的数据（即`ZipList`）。

如果树里的每一个节点上的`[]`代表的时候不确定的结算结果，那么我们可以将这个树展开，每个节点上的`[]`里的一个元素代表在这个节点的可能的元素，我们会得到**很多**树（类似集合的笛卡尔乘积）。

另一种则是，把每个节点的`[]`的第一个元素拿出来组成一个树，每个节点的第二个元素拿出来组成一个树，以此类推。我们可以得到`minimum . toList tree`个树。

### 给出一种将由树组成的`[]`转换成由`[]`组成的树的方法

可以将列表里的树按照坐标zip起来。

以下是另一种解法：

```haskell
data MT a = Nd a (MT a) (MT a) | MN
  deriving Show

instance Monoid a => Monoid (MT a) where
  mempty = MN
  MN `mappend` a = a
  a `mappend` MN = a
  Nd xs l r `mappend` Nd ys p q = Nd (xs `mappend` ys)
                                  (l `mappend` p)
                                  (r `mappend` q)
```

将一个包含具体元素的树lift成元素为在该节点可能出现的值：

```haskell
conformToMT :: [MT a] -> [MT [a]]
conformToMT = map lift_to_list
  where lift_to_list :: MT a -> MT [a]
        lift_to_list (Nd a l r) = Nd [a] (lift_to_list l) (lift_to_list r)
        lift_to_list MN = MN
```

然后就可以利用`foldMap`将两个`MT [a]`给`(<+>)`（`mappend`）起来：

```haskell
treesToLists :: [MT a] -> MT [a]
treesToLists = foldMap id . conformToMT
```

### 只使用`Traversable`方法实现`fmap`与`foldMap`

首先观察相关方法的类型：

```haskell
fmap :: Functor f => (a -> b) -> f a -> f b
traverse :: (Applicative f, Traversable t) => (a -> f b) -> t a -> f (t b)
```

可以观察到他们俩类型的结构是差不多的，都是两个参数：一个一元函数，一个带上下文的值，返回一个带上下文的结果。如果取`traverse`中的`f`为恒等应用函子（`Identity a`），那么`traverse`的类型会变为：

```haskell
traverse :: Traversable t => (a -> Id b) -> t a -> Id (t b)
```

也就是如果将`fmap`中的那个函数参数与`Id`这个构造器函数复合，就可以利用`traverse`了。只要最后用`getId`将结果从恒等应用函子中取出就行了。

```haskell
fmap f = getId . traverse (Id . f)
```

接下来是`foldMap`：

```haskell
foldMap :: (Monoid o, Foldable f) => (a -> o) -> f a -> o
traverse :: (Applicative f, Traversable t) => (a -> f b) -> t a -> f (t b)
```

同样，它们的结构很相似。但是`foldMap`比`traverse`多出一个幺半群的限制，即按照刚刚实现`fmap`的思路，上面`traverse`中`b`的类型应该有一个幺半群限制。同时，因为我们知道`foldMap`是将折子中所有的元素都`(<+>)`起来，而`traverse`是将遍子中所有的元素用`(<*>)`合起来，所以我们要构建的一个应用函子`f a`应该具有这样的性质：即当`a`为幺半群时，`F a1 <*> F a2 = F (a1 <+> a2)`。然而`traverse`要求其结果为`f (t a)`，与`(<*>)`的结果类型不同。但是这里我们可以构建这样一个类型`f a b`，当它与自己做`(<*>)`运算时，`a`之间做`(<+>)`运算获得另一个`a`，`b`之间做其它运算，获得`t b`。最简单的方法就是`b`留空。这里我们可以使用`Control.Applicative`模块里已经提供给我们的

```haskell
newtype Const a b = Const { getConst :: a }
```

`Const a b`满足，如果`a`是一个幺半群，那么`Const a b`是一个幺半群，同时满足当`a1`与`a2`是幺半群时，`Const a1 <*> Const a2 = Const (a1 <+> a2)`。如果将`f`换为`Const a b`应用函子，那么`traverse`的类型会变成：

```haskell
traverse :: Traversable t => (a -> Const b c) -> t a -> Const b (t c)
```

当`Const a b`中的`a`是一个幺半群时，`Const a b`是一个幺半群：

```haskell
traverse :: (Traversable t, Monoid b) =>
            (a -> Const b c) -> t a -> Const b (t c)
foldMap f = getConst . traverse (Const . f)
```

Arrow 箭子(?)
----

### 实现`Kleisli m a b`的`ArrowApply`实例，其中`Kleisli m a b`定义为

```haskell
data Kleisli m a b = Kleisli { runKleisli :: a -> m b }
```

可以看出`Kleisli`这个构造器要求一个`a -> m b`类型的参数。

现在我们首先看`app`要求的类型：

```haskell
app :: arr (arr b c, b) c
-- or
app :: (b `arr` c, b) `arr` c
-- or
app :: (b ~> c, b) ~> c
```

也就是说我们的`Kleisli m a b`的`ArrowApply`实例里`app`的类型应该是：

```haskell
app :: Monad m => Kleisli m (Kleisli m b c, b) c
app = Kleisli $ \ ...
```

这个`Kleisli m (Kleisli m b c, b) c`的第二个类型参数是一个二元元组，所以：

```haskell
app = Kleisli $ \(Kleisli f, b) -> ...
```

现在既然给了我们`f :: Monad m => b -> m c`和`b :: b`，很难不让人想把`b`应用到`f`上，于是有了：

```haskell
app = Kleisli $ \(Kleisli f, b) -> f b
```

### 实现`ArrowMonad m a = ArrowMonad (m () a)`的`Monad`实例

注：`ArrowMonad`的构造器是一个一元函数，接受一个`ArrowApply`类的值。

也就是让我们用箭子相关的函数实现`fmap`，`pure`，`(<*>)`和`(>>=)`。以下就不一一证明相关实例是否符合对应的法则了。

```haskell
instance ArrowApply m => Functor (ArrowMonad m) where
  -- fmap :: (a -> b) -> ArrowMonad m a -> ArrowMonad m b
  fmap f (ArrowMonad ma) = ArrowMonad $ ...
```

要实现这个，也就要想办法把`a`弄出来，才好应用到`f`上。观察`Control.Arrow`中的各种组合子，发现`(>>>)`具有我们感兴趣的类型：

```haskell
(>>>) :: Category cat => cat a b -> cat b c -> cat a c
-- i.e.
(>>>) :: ArrowMonad m a -> ArrowMonad a b -> ArrowMonad m b
-- i.e. (>>>) :: ArrowMonad (m () a) -> ArrowMonad (a () b) -> ArrowMonad (m () b)
```

刚好对应上我们的已知量，所以我们可以将`ma`右连接（`(>>>)`）至提升到箭子中的`f`来获得想要的结果：

```haskell
fmap f (ArrowMonad ma) = ArrowMonad $ ma >>> arr f
```

接下来是应用函子：

```haskell
instance ArrowApply m => Applicative (ArrowMonad m) where
  -- pure :: a -> ArrowMonad m a
  pure a = ArrowMonad $ ...
  -- (<*>) :: ArrowMonad m (a -> b) -> ArrowMonad m a -> ArrowMonad m b
  ArrowMonad mf <*> ArrowMonad ma = ArrowMonad $ ...
```

`pure`很简单，

```haskell
pure a = ArrowMonad $ arr $ \_ -> a
-- i.e. pure = ArrowMonad . arr . const
```

根据刚才的思路，我们可以使用`(>>>)`与适当构造的箭子一起获得`ArrowMonad`中包装起来的值。但是在`(<*>)`中，包装起来的值有两个。自然的想法是两次利用`(>>>)`（也就是要两次构造合适的箭子）：

```haskell
ArrowMonad mf <*> ArrowMonad ma =
  ArrowMonad $ mf >>> arr (\f ->
               ma >>> arr (\a ->
			   f a))
```

很像`Monad`的`(>>=)`应用，但是这样有类型错误：`ma >>> arr (\a -> f a)`很明显不应该是`\f -> ...`的返回值。


观察箭子的相关方法，发现`(&&&)`可以将两个箭子合并成一个，这个箭子接受一个元组为两个箭子的输入。那么我们就可以这样构造一个函数：

```haskell
ArrowMonad mf <*> ArrowMonad ma = ArrowMonad $ mf &&& ma >>> arr (uncurry id)
-- reminder: uncurry id === \(f, x) -> f x
```

最后是单子实例。还是根据之前的思路，利用`(>>>)`来我们构造将`mx`内的值应用至`f`的一个箭子，在构造的箭子里，我们会获得一个新的值`my`（因为`f`返回一个`ArrowMonad m b`）：

```haskell
instance ArrowApply m => Monad (ArrowMonad m) where
  ArrowMonad mx >>= f = ArrowMonad $
                        mx >>>
						arr (\x -> let ArrowMonad my = f x
								   in ...
```

这时我们来考察一下我们已知量的类型：

```haskell
(>>=) :: ArrowApply m => ArrowMonad m a -> (a -> ArrowMonad m b) -> ArrowMonad m b
mx :: m () a
f :: (a -> ArrowMonad m b)
my :: m () b

(>>>) :: m () a -> m a b -> m () b

-- 以及类型约束中的ArrowApply
app :: m (m a b, a) b
```

我需要后面未知的部分能返回`ArrowMonad m b`，也就是那个匿名函数需要返回`b`，这显然是做不到的，因为没有办法从`my :: m () b`得到`b`。那么就只能好好利用`app`了，可以观察到`app`最外层的`m`的中间那一坨刚好对应之前的定义`ArrowMonad (m () a)`，那么只要设法利用`app`，就可以使得`ArrowMonad $`后面那一坨变成`m _ b`，从而获得我们需要的返回值类型。

具体来说，`app`现在的类型是：

```haskell
app :: m (m () b, a) b
```

中间元组的第一部分刚好就是`my`！如果令`a`为`()`使得它满足`ArrowMonad (m () t)`的定义（即该`ArrowApply`的输入必须为一个`()`），那么省略号处的代码我们也清楚了：

```haskell
ArrowMonad mx >>= f = ArrowMonad $
                      mx >>>
					  arr (\x -> let ArrowMonad my = f x
					             in (my, ())) >>>
	                  app
```

Fin.

欢迎勘误与讨论！

