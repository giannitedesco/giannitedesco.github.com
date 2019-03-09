---
title: Item-at-a-time Reduce Functions in Python
layout: post
comments: true
date: 2019-03-09 23:28:47 +0900
tags: python mypy
summary: A nice way to implement reduce functions avoiding a conditional
---

If you need to write a
[Reduce](https://en.wikipedia.org/wiki/Fold_(higher-order_function)) function
in python, there's a number of ways of doing it. I'm going to assume you
already know what that is and leap right in. Suffice to say, you can think of
that as something like an aggregate function in an SQL query.

Perhaps the most obvious would be to use
[functools.reduce](https://docs.python.org/3/library/functools.html#functools.reduce).

```python
from functools import reduce
from operator import add

reduce(add, range(20))
```

But in this case you would need the entire iterable to be ready at the time of
calling `reduce`. What if you want to call the `reduce` one step at a time?
This would be particularly important, for example, if you were receiving the
items over a network connection or reading them from a huge file.

## With Generators
Another way might be to use a generator, but that's not without it's problems either:
```python
from operator import add

def genreduce(init_val, func):
    val = init_val
    yield init_val
    while True:
        nxt = yield val
        val = func(val, nxt)

def online_reduce(func, init_val, gen):
    state = genreduce(init_val, func)
    init_val = next(state)
    for item in gen:
        init_val = state.send(item)
    return init_val
```

So now things have got pretty ponderous. And the two things we want to specify
here are quite cumbersome to carry around - the function defining the reduction
and it's initial value.

## With Function Attributes
What I'm really after is a concise way to specify the _callees_ - the reduce
functions themselves. And an intuitive pythonic protocol for a _caller_ to use
them.

Luckily, in Python, we can just add attributes to functions.

```python
def foldsum(prev_val, val):
    return prev_val + val
foldsum.init_val = 0

def online_reduce(func, gen):
    val = func.init_val
    for item in gen:
        val = func(val, item)
    return val
```

This is nice, it lets us combine the function and the initial-value without
resorting to defining some new class, or just using a tuple like `(add, 0)`
which is essentially untyped and can get confusing.

It does make type checking blow up though.

## With a Decorator
Lets try again but make this a function decorator. We want to be able to
decorate our own functions as well as things like `operator.add`, so we'll need
an extra level of indirection so we don't start adding attributes on to members
of the `operator` module!

When it comes to typing, we want the caller to be able to access
`func.init_val` without causing a type error. We can define a custom protocol
for that using the
[typing\_extensions](https://mypy.readthedocs.io/en/latest/protocols.html#simple-user-defined-protocols)
module.

But also, our actual decorator needs to not blow up `mypy` and that gets a
little bit tricky. I couldn't figure out how to avoid using a
[cast](https://docs.python.org/3/library/typing.html#typing.cast).

With all that said, here's all the tedious boilerplate for the decorator:

```python
from typing import Callable,Any,cast
from typing_extensions import Protocol
from functools import wraps

class ReduceFunc(Protocol):
    init_val : Any
    def __call__(prev_val, val):
        raise NotImplementedError

def reducer(parm : Any):
    def decorator(func) -> ReduceFunc:
        @wraps(func)
        def wrapper(a, b):
            return func(a, b)
        rf = cast(ReduceFunc, wrapper)
        rf.init_val = parm
        return rf
    return decorator

__all__ = (
    'reducer'
)
```

And here's two ways of defining a `reduce` function. First for a user-defined
function, secondly when wrapping a function from somewhere else like the
`operator` module.

```python
@reducer(0)
def foldsum(prev_val, val):
    return prev_val + val
```

```python
from operator import add
foldsum = reducer(0)(add)
```
