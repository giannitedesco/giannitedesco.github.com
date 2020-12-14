---
title: A Faster Partition Function in Python
layout: post
comments: true
mathjax: true
date: 2020-12-14 23:22:00 +0900
tags: performance python
summary: In which we demonstrate how to optimize loops in cpython
---

Some pythonistas wonder what is the fastest way to write a function to
partition a sequence of items in to two lists based on some predicate. One
could, to be sure, `filter` the list and then `filterfalse` the list. But in
this case the predicate will evaluated twice for each item - an inefficiency
that some would balk at. That approach also doesn't seem very parsimonious.

This problem has inspired several broad classes of solutions based on different
needs and priorities and some very minor controversy about which is the most
efficient. I hope to clear up the question of efficiency by introducing a new
variant here today.

# An Itch You Just Can't Scratch
Have you ever found yourself doing something like this:

```python
foos = [x for x in input if x.type in known_foos]
bars = [x for x in input if x.type not in known_foos]
```

It's a DRY itch. Quite irritating. It calls for a salve. Surely Python would
have a solution for this? It has an expansive standard library...

# Could I Even Give Two Forks?
The suggestion in the `itertools` documentation is to use `tee` in an elegant
way, along with `filter` and `filterfalse`, which are also both in the standard
libraries. This approach also has the benefit of working with arbitrary
generators. It can even, in principle, handle an infinitely long input
sequence.

It's quite clever, so it's worth mentioning.

```python
from itertools import tee, filterfalse

def stdlib(pred, it):
    a, b = tee(it)
    return filterfalse(pred, a), filter(pred, b)
```

But the problem is that it's also quite slow when you're just going to be
materialising the results immediately. To see why, imagine calling it:

```python
hay, needles = partition(lambda item: item in needle_set, haystack)
return Results(list(hay), list(needles))
```

As soon as we materialise one of the returned generators, tee will internally
materialise haystack in to a temporary list. And the predicates are still being evaluated twice, once for each fork of the `tee`.

But I can see this sort of thing being useful in a graph of `asyncio` tasks or
something. Maybe one day I will find myself in a situation where I would need
to use a cool technique like this.

# The Clever Trevors
But let's forget about possibly-infinite generators and lazy evaluation and
just consider the case of a list that we want to partition in to two new lists.

For that case case, I've seen a couple of clever approaches.

The first one is quite quick to type and nice and readable. It involves using
the result of the predicate function to index in to the results tuple. It takes
advantage of the fact that `True` and `False` cast to `1` and `0` when used to
index a tuple.

```python
def trevor(pred, it):
    ret = ([], [])
    for item in it:
        ret[pred(item)].append(item)
    return ret
```

Honestly, this probably would have been my go-to approach, but it actually
turns out to be quite slow.

The second is a very functional-style solution, straight out of some sort of
Haskellers PhD thesis, proposed by stackoverflow member Mariy. This one uses
`functools.reduce`, which is pythons version of `foldl`:

```python
def mariy(pred, it):
    return reduce(lambda x, y: x[pred(y)].append(y) or x, it, ([], []))
```

In princple, this is the same as the above but it takes advantage fo the fact
that reduce performs some of the boilerplate for you.

I expect it to be a bit less efficient since it creates as many result tuples
as there are elements in the input list. Repetetive object creation and
destruction is a common source of overhead in python.

# The Plain Jane
According to gboffi, it turns out the fastest solution that
[stackoverflow](https://stackoverflow.com/questions/4578590/python-equivalent-of-filter-getting-two-output-lists-i-e-partition-of-a-list/52822805#52822805)
came up with is just the straight forward version, credited to Mark Byers:

```python
def byers(pred, it):
    ts = []
    fs = []
    for item in it:
        if pred(item):
            ts.append(item)
        else:
            fs.append(item)
    return fs, ts
```

But, knowing something about optimizing python code, I can see a clear
opportunity to speed this up.

# Don't dis My Code, Man
We can disassemble this to python bytecode with `dis`. Generally speaking, the
main determinant of performance in these kind of loops is the number of
bytecode instructions which need to be dispatched per iteration.

So here is what the Byers version compiles to in cpython 3.9:


```
  2           0 BUILD_LIST               0
              2 STORE_FAST               2 (ts)

  3           4 BUILD_LIST               0
              6 STORE_FAST               3 (fs)

  4           8 LOAD_FAST                1 (it)
             10 GET_ITER
        >>   12 FOR_ITER                34 (to 48)
             14 STORE_FAST               4 (item)

  5          16 LOAD_FAST                0 (pred)
             18 LOAD_FAST                4 (item)
             20 CALL_FUNCTION            1
             22 POP_JUMP_IF_FALSE       36

  6          24 LOAD_FAST                2 (ts)
             26 LOAD_METHOD              0 (append)
             28 LOAD_FAST                4 (item)
             30 CALL_METHOD              1
             32 POP_TOP
             34 JUMP_ABSOLUTE           12

  8     >>   36 LOAD_FAST                1 (fs)
             38 LOAD_METHOD              0 (append)
             40 LOAD_FAST                4 (item)
             42 CALL_METHOD              1
             44 POP_TOP
             46 JUMP_ABSOLUTE           12

  9     >>   48 LOAD_FAST                3 (fs)
             50 LOAD_FAST                2 (ts)
             52 BUILD_TUPLE              2
             54 RETURN_VALUE
```

The inner loop is between locations 12 and 46. It's 18 instructions long.

We can see that the append code has some duplicated instructions in each branch
(24-34, and 36-46), but since only one branch is taken at a time, that
shouldn't really be counted.

So if we measure the actual instruction path-length per iteration, it comes out
to 12 instructions.

More importantly, we can see that inside that inner loop we're doing a lookup
of the `append` method, which is actually looking up a string in a dictionary.
For this reason, the `LOAD_GLOBAL` and `LOAD_METHOD` calls are among the
slowest of the basic instruction types in the python machine (not including the
instructions which call out to python subroutines, of course).

# Python is Dynamic, and There is no Escape(-Analysis)
We have probably all heard that python is a dynamic language. But we may not
know what all of the consequences of that fact are. For one thing, it means
that attribute lookups are often expensive dictionary lookups. But it also
means that even some very obvious optimisations _simply cannot be made_. For
example, the python compiler cannot, in general, optimise:

```python
# Program A
x = []
for a in b:
    x.append(a)
```

in to:

```python
# Program B
x = []
f = x.append
for a in b:
    f(a)
```

The reason for this is that the method `list.append` is absolutely free to
assign a different value to `self.append`. This means that programs A and B are
(or at least, may be) semantically different. Therefore B is not a valid
optimization of A.

Now you might think that, well, the bytecode compiler can know about the
implementation of `list.append` because it's built-in. Which is true. But it
doesn't stop any other code, for example in `b.__iter__.next()` from obtaining
a reference to `x` via any number of mechanisms and then altering its attribute
dict.

You wouldn't even have to resort to reflection - the local variable `x` could
just be a reference to an object which is also reachable in a global scope. And
what variables there are in the global scope can be modified on the fly by any
code so it wouldn't be enough to do an
[escape-analysis](https://en.wikipedia.org/wiki/Escape_analysis) at compile
time.

Frankly there are just too many avenues in python for this sort of
jiggery-pokery to take place. And these avenues are kind of the point of
python.

# Locals are Fast In Python
OK, that's interesting, but what if we just explicitly apply the above
optimisation?

```python
def scara(pred, it):
    ts = []
    fs = []
    t = ts.append
    f = fs.append
    for item in it:
        if pred(item):
            t(item)
        else:
            f(item)
    return fs, ts
```

You can see from the bytecode that this has reduced the instruction path-length
as we expected:
```
 28           0 BUILD_LIST               0
              2 STORE_FAST               2 (ts)

 29           4 BUILD_LIST               0
              6 STORE_FAST               3 (fs)

 30           8 LOAD_FAST                2 (ts)
             10 LOAD_ATTR                0 (append)
             12 STORE_FAST               4 (t)

 31          14 LOAD_FAST                3 (fs)
             16 LOAD_ATTR                0 (append)
             18 STORE_FAST               5 (f)

 32          20 LOAD_FAST                1 (it)
             22 GET_ITER
        >>   24 FOR_ITER                30 (to 56)
             26 STORE_FAST               6 (item)

 33          28 LOAD_FAST                0 (pred)
             30 LOAD_FAST                6 (item)
             32 CALL_FUNCTION            1
             34 POP_JUMP_IF_FALSE       46

 34          36 LOAD_FAST                4 (t)
             38 LOAD_FAST                6 (item)
             40 CALL_FUNCTION            1
             42 POP_TOP
             44 JUMP_ABSOLUTE           24

 36     >>   46 LOAD_FAST                5 (f)
             48 LOAD_FAST                6 (item)
             50 CALL_FUNCTION            1
             52 POP_TOP
             54 JUMP_ABSOLUTE           24

 37     >>   56 LOAD_FAST                3 (fs)
             58 LOAD_FAST                2 (ts)
             60 BUILD_TUPLE              2
             62 RETURN_VALUE
```

The inner loop takes place between locations 24 and 48. It's 16 instructions
long. And if we look at the instruction path length for the loop, it's now 11
instructions long. One shorter than before. Of course, it's that missing
`LOAD_ATTR` instruction which has been hoisted out of the loop in to the
function preamble.


# A Bit More Squeezing
I couldn't figure out a way to get the inner loop any tighter but I did find a
way to make the code more compact by removing duplicated code on both sides of
the branch. It led to a tiny improvement in performance. Here it is:


```python
def scara2(pred, it):
    ts = []
    fs = []
    t = ts.append
    f = fs.append
    for item in it:
        (t if pred(item) else f)(item)
    return fs, ts
```

```
 40           0 BUILD_LIST               0
              2 STORE_FAST               2 (ts)

 41           4 BUILD_LIST               0
              6 STORE_FAST               3 (fs)

 42           8 LOAD_FAST                2 (ts)
             10 LOAD_ATTR                0 (append)
             12 STORE_FAST               4 (t)

 43          14 LOAD_FAST                3 (fs)
             16 LOAD_ATTR                0 (append)
             18 STORE_FAST               5 (f)

 44          20 LOAD_FAST                1 (it)
             22 GET_ITER
        >>   24 FOR_ITER                24 (to 50)
             26 STORE_FAST               6 (item)

 45          28 LOAD_FAST                0 (pred)
             30 LOAD_FAST                6 (item)
             32 CALL_FUNCTION            1
             34 POP_JUMP_IF_FALSE       40
             36 LOAD_FAST                4 (t)
             38 JUMP_FORWARD             2 (to 42)
        >>   40 LOAD_FAST                5 (f)
        >>   42 LOAD_FAST                6 (item)
             44 CALL_FUNCTION            1
             46 POP_TOP
             48 JUMP_ABSOLUTE           24

 46     >>   50 LOAD_FAST                3 (fs)
             52 LOAD_FAST                2 (ts)
             54 BUILD_TUPLE              2
             56 RETURN_VALUE
```

Now we're down to a 13 instruction inner loop with an 11 instruction
path-length.

# Benchmarks
For benchmarking I loaded a dictionary of some half a million words, created a
set of all the words that begin with "s", and then partitioned the dictionary
based on membership of the "s" set.

Tests were performed on an i7-6600U laptop with python 3.9.0 and
`PYTHONHASHSEED=0`

```python
dic = tuple(Path('/usr/share/dict/words').read_text().splitlines())
needles = frozenset((word for word in dic if word.startswith('s')))
for func_name in ('stdlib', 'mariy', 'trevor', 'byers', 'scara', 'scara2'):
    print(func_name, timeit(
        f'{func_name}(pred, dic)',
        setup=f'from __main__ import dic, pred, {func_name}',
        number=32,
    ))
```

|----------|------------------------|
| Variant  | Time for 32 iterations (seconds) |
|----------|------------------------|
|stdlib    | 3.5664224450010806     |
|mariy     | 3.532839963911101      |
|trevor    | 2.5254638858605176     |
|byers     | 2.356995061971247      |
|scara     | 2.1279552809428424     |
|scara2    | 2.087127824081108      |

# A Parting Shot
Here it is with all the `mypy --strict` typing goodness added:

```python
from typing import List, Any, Callable, Iterable, TypeVar, Tuple

T = TypeVar('T')

def partition(pred: Callable[[T], bool], it: Iterable[T]) \
        -> Tuple[List[T], List[T]]:
    ts: List[T] = []
    fs: List[T] = []
    t = ts.append
    f = fs.append
    for item in it:
        (t if pred(item) else f)(item)
    return fs, ts
```
