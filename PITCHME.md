Contract Programming
====================

Theory and Implementation
-------------------------

**Peter Varo** &copy; 2017

---

What is contract programming?
=============================

> *[...]* software designers should define **formal**, **precise** and
> **verifiable** interface specifications for software components, which extend
> the ordinary definition of *abstract data types* with **preconditions**,
> **postconditions** and **invariants**.
> <br/>
> &mdash; **Wikipedia:** *Design by Contract*

---

All right, but what does that mean?
===================================

- A program is rarely a single block
- Most blocks are building on top of each other => block chains aka **units**
- Units should be responsible for correctness
- To make that happen blocks have to make *promises* to each other
- These *promises* are the contracts

---

What does it look like in practice?
===================================

- Instead of abstract thoughts, let's see some examples!
- Python doesn't have native support for contracts
- I implemented a library, called `pcd` (Python Contract Decorators)
- FOSS under the Lesser General Public License version 3.0
- Pure Python implementation and supporting both Python 2 and 3
- Easy to maintain, clean, small source: ~210 LoC (+ comments)

![pcd][10]

---

Let's say we have a ticket...
=============================

> Process a list of sub-strings coming from an external source into a single
> string, where:
> - the sub-strings are delimeted by the `::` symbol,
> - empty sub-strings are filtered out,
> - the sub-strings containing only spaces are considered to be empty,
> - the sub-strings' leading and trailing whitespaces should be removed,
> - remaining spaces in sub-strings shall be replaced with the `_` character.

---

The naive approach...
=====================

```python
def get_data(external_data):
    return '::'.join(process_values(external_data))

def process_values(values):
    return filter(None, map(sanitise_value, values))

def sanitise_value(value):
    return value.strip().replace(' ', '_')
```
```text
>>> get_data(['  alpha beta', '  ', 'gamma', 'delta epsilon  '])
'alpha_beta::gamma::delta_epsilon'
```

---

Then comes the seasoned developer...
====================================

```python
def get_data(external_data):
    if (isinstance(external_data, str) or
        not all(isinstance(v, str) for v in external_data)):
            raise ValueError('Invalid input data')
    try:
        return '::'.join(process_values(external_data))
    except TypeError:
        raise TypeError('Internal error: cannot produce string')
    except Exception as error:
        raise TypeError('Internal error: {}'.format(error))

def process_values(values):
    if isinstance(values, str):
        raise TypeError('Expected an iterable of strings, but got: str')
    sanitised = []
    for value in values:
        if not isinstance(value, str):
            raise TypeError('Expected a string, '
                            'but got: {}'.format(value.__class__.__name__))
        sanitised.append(sanitised_value(value))
    return sanitised

def sanitise_value(value):
    try:
        return value.strip().replace(' ', '_')
    except AttributeError:
        raise TypeError(
            'Expected a string, but got: {}'.format(value.__class__.__name__))
```

---

> **Note:** The previous error handlings were mostly about type checking, which
> of course can be handled by a static type checker at compile time.  However
> 1. The currently available optional type checking in Python is pretty much in
>    its infancy
> 2. Even in the previous example not all cases can be solved by type checking
> 3. This is just a dummy example, try to think about schema checks as well!

---

What's wrong with this?
=======================

1. The code is completely cluttered, very hard to read and see what is going on
2. Most of the checks are redundant in a way, they are only there to support
   modularity
3. Redundant runtime checkings introduce significant overhead
4. Removing any of the checkings leads to undocumented and/or non-binding
   assumptions, which will introduce unexpected bugs

---

Contracts to save the day!
==========================

```python
@contract(post=[lambda r: isinstance(r, str),
                lambda r: ' ' not in r,
                lambda r: ':::' not in r])
def get_data(external_data):
    if (isinstance(external_data, str) or
        not all(isinstance(v, str) for v in external_data)):
            raise ValueError('Invalid input data')
    return '::'.join(process_values(external_data))

@contract(pre=[lambda: isinstance(values, str),
               lambda: not all(isinstance(v, str) for v in values)],
          post=[lambda r: isinstance(r, collections.Iterable),
                lambda r: all(isinstance(v, str) for v in r),
                lambda r: all(r)])
def process_values(values):
    return filter(None, map(sanitise_value, values))

@contract(pre=lambda: isinstance(value, str),
          post=lambda r: isinstance(value, str))
def sanitise_value(value):
    return value.strip().replace(' ', '_')
```

---

That's cool, but what is the advantage there?
=============================================

1. The in- and output error handling is separated from the logic of the function
2. The error reporting is automatic, all of them are *deadly* (`AssertionError`)
3. Not only the inputs are *constrained*, but the outputs as well
4. They can be removed without touching the code (`-O`)
5. (`mut` -- ability to monitor the states of mutable objects)

---

What about `class`es?
=====================

**Q:** Why are we abstracting data into objects in the first place?

**A:** To guarantee the integrity of the abstracted data

---

Let's say we have a ticket...
=============================

> Implement a triangle geometric object, which holds valid values and can
> provide an `area` member that returns the calculated are of the triangle.

---

The naive approach...
=====================

```python
class Triangle:

    def __init__(self, a, b, c):
        if a <= 0 or b <= 0 or c <=0:
            raise ValueError('All sides need to be greater than 0')
        elif a + b <= c or a + c <= b or b + c <= a:
            raise ValueError(
                'The sum of any two sides has to be greater than the third one')
        self.a = a
        self.b = b
        self.c = c

    @property
    def area(self):
        # Here goes the implementation of Heron's Formula
```

---

Then comes the seasoned developer...
====================================

```python
class Triangle:

    def __init__(self, a, b, c):
        self._validate(a, b, c)
        self._a = a
        self._b = b
        self._c = c

    @staticmethod
    def _validate(a, b, c):
        if a <= 0 or b <= 0 or c <=0:
            raise ValueError('All sides need to be greater than 0')
        elif a + b <= c or a + c <= b or b + c <= a:
            raise ValueError(
                'Sum of any two sides have to be greather than the third one')

    @property
    def a(self):
        return self._a
    @a.setter
    def a(self, value):
        self._validate(value, self._b, self._c)
        self._a = value

    @property
    def b(self):
        return self._b
    @b.setter
    def b(self, value):
        self._validate(self._a, value, self._c)
        self._b = value

    @property
    def c(self):
        return self._c
    @a.setter
    def c(self, value):
        self._validate(self._a, self._b, value)
        self._c = value

    @property
    def area(self):
        self._validate(self._a, self._b, self._c)
        # Here goes the implementation of Heron's Formula
```

---

What's wrong with this?
=======================

1. The code is dominated by the error handling
2. Even if the *user* of such object can guarantee the validity of the data
   provided to it the checks will always run
3. By not calling the validator explicitly can cause troubles later on, when the
   developer decides the internal method should become a public one

---

Invariants to save the day!
===========================

```python
class Triangle:

    __metaclass__ = Invariant
    __conditions  = (lambda: self.a > 0,
                     lambda: self.b > 0,
                     lambda: self.c > 0,
                     lambda: self.a + self.b > self.c,
                     lambda: self.a + self.c > self.b,
                     lambda: self.b + self.c > self.a)

    def __init__(self, a, b, c):
        self.a = a
        self.b = b
        self.c = c

    @property
    def area(self):
        # Here goes the implementation of Heron's Formula
```

---

That's cool, but what is the advantage there?
=============================================

0. Everything that we've learnt about `contract`s
1. All checks are running after the `__init__`, before the `__del__` and before
   and after of every public method automatically.
2. Optimised checks: the invariants are merged with the `contracts` defined for
   each methods.
3. The conditions are inherited.
4. The conditions can be extended during inheritance.
5. Works with `@property` callbacks.
6. They can be removed without touching the code (`-O`)

---

PCD
===

Get Involved
------------

- Play with it today! &rarr; `pip install pcd`
- Give it a &#9733; if you like it! &rarr; https://github.com/petervaro/pcd
- Fork it and become an active contributor!
- Or just look into the code: massive meta programming all over the place!

Future Plans
------------

1. Create super lightweight `Interface`es that can inherit `contract`s defined
   on their virtual methods.
2. Replace `uncompyle6` with `decompyle++` for more stable Python 3 support.
3. Auto-generated initialisers and finalisers for classes that are defining
   invariants.

---

Q&A
===

*...or comments perhaps?*

<!-- links -->
[10]: https://github.com/petervaro/pcd/raw/master/img/logo.png?raw=true
