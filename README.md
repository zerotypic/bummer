# bummer
Python metaclass for creating generic lazy objects

## Requirements

* Python 3.x (not tested on Python 2.x, but maybe it works)

## Usage

```python
import bummer

class Foo(object, metaclass=bummer.LazyMeta):
    def __init__(self, index, value):
        print("Hi, __init__ is being called!")
        self.index = index
        self.value = self.expensive_computation(value)

    def __lazy_preinit__(self, index, value):
        print("__lazy_preinit__ is called first.")
        self.index = index

    def expensive_computation(self, value):
        print("Performing expensive computation.")
        return len(repr(value)) % 42
```

### Lazy instantiation

```python
In [4]: foo = Foo(10, "something")
__lazy_preinit__ is called first.

In [5]: foo.index
Out[5]: 10

In [6]: foo.value
Hi, __init__ is being called!
Performing expensive computation.
Out[6]: 11
```

### Preserving type hierarchy

```python
In [7]: foo2 = Foo(20, "other")
__lazy_preinit__ is called first.

In [8]: type(foo2)
Out[8]: bummer.Foo_Lazy

In [9]: isinstance(foo2, Foo)
Out[9]: True

In [10]: foo2.value
Hi, __init__ is being called!
Performing expensive computation.
Out[10]: 7

In [11]: type(foo2)
Out[11]: __main__.Foo
```

## How it works

`LazyMeta` is a metaclass that creates "half-initialized" objects.

Python objects are initialized in a two-step process, where memory for the
object is first allocated (see `__new__()`), and then the object's
initialization function is called (see `__init__()`).

When `LazyMeta` is used to turn some target class into a *lazified* class, it
first creates a stub class (`_Lazy`) associated with that class. Then,
whenever objects of the target class are to be created, the stub class is
returned instead.

The `_Lazy` stub class, in its implementation of `__new__()`, uses the target
class's `__new__()` to create a new half-initialized object of the target
class. The class of the half-initialized object is then changed to `_Lazy`,
which results in the target class's `__init__()` to not run. As a result, the
object returned when an instance of the lazified target class is requested is
a half-initialized target class object, masquerading as a `_Lazy` object.

By overriding `__getattr()__` and `__setattr__()`, the `_Lazy` class then
ensures that when any attribute on the object is accessed, *reification* takes
place. When the object is reified, the class of the object is switched back to
the target class, and the call to the target class's `__init__()` is made,
completing initialization of the object. The resultant object then behaves
exactly like it would if it had never been lazified.

The `__lazy_preinit__()` function is designed to allow certain attributes to
be set on the lazy object, which can be accessed without triggering
reification.

## Why is this so complex?

It is clearly a lot easier to do this by implementing some kind of a proxy
object, and overriding `__getattribute__()` on it to redirect to the target
object. However, this approach would result in a check on the status of the
object (whether it has been reified or not) every time an attribute was
accessed.

I wanted to find a way to create lazy objects where such a check did not
happen; that once a lazy object is reified, access to its attributes happen
with no additional overhead. So this is the solution I ended up with.

## Is this really a good idea?

I don't know, probably not. It involves magic (changing `__class__` for example)
that is likely bad practice. I've also not tested it on other kinds of
objects, especially more complex ones. So far though, it works for me. Use it
at your own risk, and file bug reports if you find any!

## TODO

* Document `LazyFactoryMeta` in README.

## License

MIT License

See [LICENSE](/LICENSE) for full text.
