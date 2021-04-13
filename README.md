# Infixed

This is a golfed implementation of custom infix operators (I think about 5 lines of source code; on top of that, only one of those
lines contains the core functionality!). Here's an example:

```python

>>> from infixed import infix
>>> @infix
... def doubleadd(x, y):
...     return 2*x + y
...
>>> doubleadd(2, 4)
8
>>> 2 |doubleadd| 4
8

```

That example shows how to use the default infix function. This module contains many other infix functions
which use different delimiters, such as `and_infix` (`&doubleadd&`) and `mul_infix` (`\*doubleadd\*`). This
also contains the function `make_infix`, which allows you to define your own infix operators.

To use `make_infix`, you should first know the abbreviation for the operator; these can be found in the magic
method used to implement that operator (e.g. "truediv" for division because division is implemented using
`__truediv__`).  Afterwords, you can create the constructer function by calling `make_infix("truediv")`.
The returned result (it should be the class "TruedivInfix" or something along those lines) can be used to construct
infix operators delimited by the operator of your chooseing (in this example, you can now write `/doubleadd/`). Here's
that complete example:

```python

>>> from infixed import make_infix
>>> @make_infix('truediv')
... def doubleadd(x, y):
...     return 2*x + y
...
>>> 2 /doubleadd/ 4
8

```

Infixed already provides many common delimiters out of the box:

```python

>>> from infixed import add_infix, mul_infix, div_infix
>>> doubleadd = lambda x, y: 2*x + y
>>> add_delim = add_infix(doubleadd)
>>> mul_delim = mul_infix(doubleadd)
>>> div_delim = div_infix(doubleadd)
>>> 2 +add_delim+ 4
8
>>> 2 *mul_delim* 4
8
>>> 2 /div_delim/ 4
8

```

All of the functions above can be used as decorators as well (docstrings and other attributes won't fall through!)

## How it works

If you take away one thing from this section, it should be **don't use this code for anything important!**. If you
take a look at the source code (infixed.py), you will see that first big line of code, and your breath will be
stilled, for the monstrosity which I have created cannot be tamed.

With that disclaimer out of the way, I'm going to try to explain that first line of code: the implementation of
`make_infix`.

Here's the important part of that line:

```python
make_infix = lambda op: type(f'{op.title()}Infix', (), {'TYPE': op, '__new__': lambda cls, fn, args=[]: fn(*args) if len(args) > 1 else object.__new__(cls), '__init__': lambda self, fn, args=[]: [setattr(self, 'fn', fn), setattr(self, 'args', args)][0], f'__r{op}__': (w := lambda self, other: type(self)(self.fn, self.args + [other])), f'__{op}__': w, '__call__': lambda self, *args: self.fn(*args), '__doc__': 'DOCSTRING NOT SHOWN'})
```

To put it into a oneliner, I have taken great liberties with lambdas, setattr, python's default eager evaluation,
and the type function. Ungolfing this code would lead to something like this:

```python
def make_infix(op):
    class {op.title()}Infix:            # I know that this isn't proper syntax, but this is hard enough as it is
        "DOCSTRING NOT SHOWN"
        TYPE = op
        def __new__(cls, fn, args=[]):
            if len(args) > 1:
                return fn(*args)
            else:
                return object.__new__(cls)

        def __init__(self, fn, args=[]):
            self.fn = fn
            self.args = args

        def __r{op}__(self, other):
            return type(self)(self.fn, self.args + [other])

        def __{op}__(self, other):
            return type(self)(self.fn, self.args + [other])

        def __call__(self, *args):
            return self.fn(*args)
```

Note that I use f-string syntax to show how the method names and the class name depend on `op`.

Okay, an in-depth explanation.

First, a review of `__new__`. `__new__` is called before `__init__` with the exact same arguments. `__new__`
is expected to return an instance of the class; this is what `object.__new__(cls)` does. However, if `__new__` returns
an instance of any other class, `__init__` isn't called; the value of the expression is the value that `__new__` returns.
In my infix class, it only initializes the class when the number of arguments is less than one; that is, the class is only used
when the result can't be computed. If it can be computed, it calls the function with the provided arguments then and there and returns
the results.

The rest of the code is pretty simple; when you use this class with an operator, it returns a copy of the class where `args` has an extra
element: the object it was called with (if it was called as `5 |me`, `me.args` would cease to be `[]` and would now be `[5]`). But this
copy of the class could also be the result of the computation if `args`'s length is greater than or equal to 2.

I think the ungolfed code speaks for itself. Hopefully you can understand the golfed code now and understand why you shouldn't use it at
all.
