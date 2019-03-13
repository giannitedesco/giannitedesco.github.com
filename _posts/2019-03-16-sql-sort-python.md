---
title: SQL-Like Sorting in Python
layout: post
comments: true
date: 2019-03-16 23:20:00 +0900
tags: python sql
summary: Sort lists and iterables like it's 1974
---

Sorting things in python can often be a pain point. You need to be familiar
with the _decorate-sort-undecorate_ paradigm. Also known as the [Schwartzian
transform](https://en.wikipedia.org/wiki/Schwartzian_transform). In python3
this technique replaced the old system of comparator functions. Lots of words
have been expended discussing this in the quite excellent [python
documentation](https://docs.python.org/3/howto/sorting.html) so I won't try and
duplicate any of that here.

For what it's worth, I actually agree with the python implementors that this is
the better, less error-prone approach to the problem. And it can have some
performance benefits too.

But what if you crave simpler times? Flash back to when Eric Clapton's cover of
"I shot the sheriff" was blaring out of everyone's 8-tracks. SQL gives us a
pretty nice and intuitive formalisation for defining orderings over bags of
tuples. And iterable sequences of objects in python can be considered exactly
that.

Let's get the preliminaries out of the way. This is what the use-case looks
like:

```python
from types import SimpleNamespace
from enum import Enum
class Order(Enum):
    ASC = 0
    DESC = 1

keyfunc = sqlordering(('x', Order.ASC), ('y', Order.DESC))

input = [
    SimpleNamespace(x=999, y='aardvark'),
    SimpleNamespace(x=999, y='xylophone'),
    SimpleNamespace(x=111, y='xylophone'),
    SimpleNamespace(x=111, y='aardvark'),
    # etc..
]

for item in sorted(input, key=keyfunc):
    print(item)
# SimpleNamespace(x=111, y='xylophone')
# SimpleNamespace(x=111, y='aardvark')
# SimpleNamespace(x=999, y='xylophone')
# SimpleNamespace(x=999, y='aardvark')
```

## The key function
Let's work backwards from the key function required by `list.sort()`.  The
simplest way to produce a total ordering is to define a class whose constructor
takes one argument, the item that we want to compare/sort. That class should
keep a reference to the original object and then define the six [rich
comparison](https://www.python.org/dev/peps/pep-0207/) methods.

We'll use the
[functools.total\_ordering](https://docs.python.org/2/library/functools.html#functools.total_ordering)
decorator to fill in a lot of boilerplate for us. This way we'll only need to
define an `__lt__` and an `__eq__` method in our class and the decorator will
fill in the rest for us.  All those other methods can be defined in terms of
just those two (e.g. `__ne__` is equivalent to `not __eq__`).

Regardless, it looks like `sorted` and `list.sort` use only the `__lt__`
comparison so those methods ought never be called and there should be no
performance hit. But we may as well define them just because there's no such
thing as a guarantee in life. But that's another topic, and you can read about
my failures and disappointments in life in my upcoming memoir.

```python
def sqlordering(*spec):
    @total_ordering
    class K:
        __slots__ = ['_obj']
        __hash__ = None
        def __init__(self, obj):
            self._obj = obj
        def __lt__(self, other):
	    # Implement this
	    pass
        def __eq__(self, other):
	    # Implement this
	    pass
    return K
```

## How to define \_\_lt\_\_ and \_\_eq\_\_
The first thing we need to do is iterate over the sort specification. For any
field sorted `ASC`ending we'll use `operator.lt` (less-than) and for any field sorted
`DESC`ending we'll use the opposite: `operator.ge` (greater-than-or-equal).

Then we'll need to use [partial
application](https://en.wikipedia.org/wiki/Partial_application) to combine the
attribute lookups with the operator.

For example, to create a function f which takes two objects and returns `True`
if attribute `x` on the first argument is less than attribute `x` on the second
argument, we can do:

```python
from functools import partial
# a is attribute name, a string
# c is comparison operator, a callable
# x,y will remain free paramaters and can be any type
cmp_func = lambda a,c,x,y:c(getattr(x,a),getattr(y,a))
f = partial(cmp_func, 'x', operator.lt)
```

Now, with that, we need to create a higher order function which combines this
sequence of functions to create the two operators for our total ordering.

For the equals operator it's straightforward, two items are equal if and only
if all attributes are equal. So we just iterate over the sequence of equality
functions applying them as we go. If any of them fail, the comparison fails.

For the less-than operator it's a little bit more involved. Recall the semantics of SQL
sorts. You can find that in section 8.2 "\<comparison predicate\>" of ISO/IEC
9075-2:2016, general rules, paragraph 1, clause h:

> The relative position of row P is before row Q if PV<sub>n</sub> precedes
> QV<sub>n</sub> for some n, 1 (one) ≤ n ≤ N, and PV<sub>i</sub> is not
> distinct from QV<sub>i</sub> for all i < n.

In other words, if two rows are equal in the first comparison key, then we
proceed to ordering by the second comparison key, and so on.

So with that we can go ahead and complete the implementation. Here it is:

```python
from typing import Iterable, Tuple
from functools import total_ordering, partial
from operator import lt, ge, eq

def _cmp_func(*args):
    spec : Iterable[Tuple[str, Order]] = args
    cmp_func = lambda a,c,x,y:c(getattr(x,a),getattr(y,a))
    for attr, sense in spec:
        c_eq = partial(cmp_func, attr, eq)
        if sense == Order.ASC:
            c_lt = partial(cmp_func, attr, lt)
        elif sense == Order.DESC:
            c_lt = partial(cmp_func, attr, ge)
        else:
            raise ValueError
        yield (c_lt, c_eq)

def sqlordering(*spec):
    funcs = tuple(_cmp_func(*spec))

    @total_ordering
    class K:
        __slots__ = ['_obj']
        __hash__ = None
        def __init__(self, obj):
            self._obj = obj
        def __lt__(self, other):
            for ltcmp, eqcmp in funcs:
                if ltcmp(self._obj, other._obj):
                    return True
                elif not eqcmp(self._obj, other._obj):
                    return False
            return False
        def __eq__(self, other):
            for ltcmp, eqcmp in funcs:
                if not eqcmp(self._obj, other._obj):
                    return False
            return True
    return K
```
