# Core Python (syntax, core libs, etc)


## Question: [Passing Default and optional Arguments in Python](https://stackoverflow.com/q/45078413/404556)

    def fun( key=False,*var):
      if key is False:
         print(var)
         return 0
      print("Key is true")
      return 0
    fun(1,2,3)

**I have not passed a value of key in fun(1,2,3)**. Hence the default value of key i.e. **key=False**. But the output is:

    Key is true

These Cases are contradicting. **Why is this happening?**


## Answer

But you did pass `1` for `key`. And `[2,3]` for `*var`.

Positional arguments you supply to a function are bound to argument variables *in order* they
appear. The exception are keyword arguments, but you have to explicitly name them when calling the
function (and you are not doing that).

So, you'll have to reverse the order and put keyword argument `key` at the end. But that would be an
invalid syntax in Python 2. The standard solution (that works *in both Python 2 and Python 3*) is to
switch to keyword arguments dictionary and to supply default values for arguments only when
[popping](https://docs.python.org/2/library/stdtypes.html#dict.pop) them:

    >>> def fun(*var, **kwargs):
    ...    key = kwargs.pop('key', False)
    ...    print(key)
    ... 
    >>> fun(1,2,3)
    False
    >>> fun(1,2,3, key=True)
    True

Since Python 3.0 (see [PEP 3102](https://www.python.org/dev/peps/pep-3102/) for the  new syntax),
you *can have* keyword arguments after a list of positional arguments, like this:

    >>> def fun(*var, key=False):
    ...    print(key)
    ... 
    >>> fun(1,2,3)
    False
    >>> fun(1,2,3, key=True)
    True


---


## Question: [Split list into lists based on a character occurring inside of an element](https://stackoverflow.com/q/45281189/404556)

In a list like the one below: 

    biglist = ['X', '1498393178', '1|Y', '15496686585007', '-82', '-80', '-80', '3', '3', '2', '|Y', '145292534176372', '-87', '-85', '-85', '3', '3', '2', '|Y', '11098646289856', '-91', '-88', '-89', '3', '3', '2', '|Y', '35521515162112', '-82', '-74', '-79', '3', '3', '2', '|Z', '0.0', '0.0', '0', '0', '0', '0', '0', '4', '0', '154']

There could be some numerical elements preceded by a character. I would like to break this into sub-lists like below:

    smallerlist = [
        ['X', '1498393', '1'],
        ['Y', '1549668', '-82', '-80', '-80', '3', '3', '2', ''],
        ['Y', '1452925', '-87', '-85', '-85', '3', '3', '2', ''],
        ['Y', '3552151', '-82', '-74', '-79', '3', '3', '2', ''],
        ['Z', '0.0', '0.0', '0', '0', '0', '0', '0', '4', '0', '154']
    ]

As you can tell, depending upon the character, the lists could look similar. Otherwise they could
have a different number of elements, or dissimilar elements altogether. The main separator is the
`"|"` character. I have tried to run the following code to split up the list, but all I get is the
same, larger, list within a list. I.e., list of `len(list) == 1`
    
    import itertools

    delim = '|'
    smallerlist = [list(y) for x, y in itertools.groupby(biglist, lambda z: z == delim) if not x]

Any ideas how to split it up successfully? 


## Answer

First, a quick **oneliner**, which is not an optimal solution in terms of space requirements, but
it's short and sweet:

    >>> smallerlist = [l.split(',') for l in ','.join(biglist).split('|')]
    >>> smallerlist
    [['X', '1498393178', '1'],
     ['Y', '15496686585007', '-82', '-80', '-80', '3', '3', '2', ''],
     ['Y', '145292534176372', '-87', '-85', '-85', '3', '3', '2', ''],
     ['Y', '11098646289856', '-91', '-88', '-89', '3', '3', '2', ''],
     ['Y', '35521515162112', '-82', '-74', '-79', '3', '3', '2', ''],
     ['Z', '0.0', '0.0', '0', '0', '0', '0', '0', '4', '0', '154']]

Here we join all elements of the big list by a unique non-appearing separator, for example `,`, then
split by `|`, and then split again each list into a sublist of the original elements.

But if you're looking for a bit more **efficient solution**, you can do it with
[`itertools.groupby`](https://docs.python.org/3/library/itertools.html#itertools.groupby) that will
operate on an intermediate list, generated on fly with the `breakby()` generator, in which elements
without `|` separator are returned as is, and those with separator are split into 3 elements: first
part, a list-delimiter (e.g. `None`), and the second part.

    from itertools import groupby

    def breakby(biglist, sep, delim=None):
        for item in biglist:
            p = item.split(sep)
            yield p[0]
            if len(p) > 1:
                yield delim
                yield p[1]
    
    smallerlist = [list(g) for k,g in groupby(breakby(biglist, '|', None),
                                              lambda x: x is not None) if k]


---


## Question: [Why adding multiple 'nan' in python dictionary giving multiple entries?](https://stackoverflow.com/q/45300367/404556)

Example problem:

    import numpy as np
    dc = dict()
    dc[np.float('nan')] = 100
    dc[np.float('nan')] = 200

It is creating multiple entries for `nan` like

`dc.keys()` will produce `{nan: 100, nan: 200}` but it should create `{nan: 200}`.


## Answer

The short answer to your question (of why adding `NaN` keys to a Python `dict` create multiple entries), is because **floating-point `NaN` values are unordered**, i.e. a `NaN` value is not equal to, greater than, or less than anything, including itself. This behavior is defined in the [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) standard for floating point arithmetic. The explanation why is that so is given by an IEEE 754 committee member [in this answer](https://stackoverflow.com/a/1573715/404556).

---

For a longer, Python-specific, answer, let's first have a look at how item insertion and key comparison work in CPython dictionaries.

When you say `d[key] = val`, [`PyDict_SetItem()`](https://github.com/python/cpython/blob/master/Objects/dictobject.c#L1537) for dictionary `d` is called, which in turn calls (internal) [`insertdict()`](https://github.com/python/cpython/blob/master/Objects/dictobject.c#L1095), which will either update the existing dictionary item, or insert a new item (maybe resizing the hash table consequentially).

The first step on insert is to lookup the `key` in the hash table of dictionary keys. A general-purpose lookup function that gets called in your case (of non-string keys) is [`lookdict()`](https://github.com/python/cpython/blob/master/Objects/dictobject.c#L679).

`lookdict` will use `key`'s hash value to locate the `key`, iterating over a list of possible keys with identical hash value, comparing first by address, then by calling `key`s' equivalence operator(s) (see excellent comments in [`Objects/dictobject.c`](https://github.com/python/cpython/blob/master/Objects/dictobject.c#L132) for more details on hash collision resolution in Python's implementation of [open addressing](https://en.wikipedia.org/wiki/Open_addressing)).

Since every `float('nan')` **has the same hash** value, yet each one is **a different object** (with a different "identity", i.e. memory address), and they're **not equal by their float-value**:

    >>> a, b = float('nan'), float('nan')
    >>> hash(a), hash(b)
    (0, 0)
    >>> id(a), id(b)
    (94753433907296, 94753433907272)
    >>> a == b
    False

when you say:

    d = dict()
    d[float('nan')] = 1
    d[float('nan')] = 2

`lookdict` will search for the second `NaN` by looking at its hash (`0`), then trying to resolve hash collision by iterating over keys with the same hash and comparing the keys by identity/address (they are different), then by invoking (the expensive) `PyObject_RichCompareBool`/[`do_richcompare`](https://github.com/python/cpython/blob/master/Objects/object.c#L655), which in turn calls [`float_richcompare`](https://github.com/python/cpython/blob/master/Objects/floatobject.c#L350) which compares floats just as C does:

    /* Comparison is pretty much a nightmare.  When comparing float to float,
     * we do it as straightforwardly (and long-windedly) as conceivable, so
     * that, e.g., Python x == y delivers the same result as the platform
     * C x == y when x and/or y is a NaN.

which behaves according to IEEE 754 standard (from [GNU C library docs](https://www.gnu.org/software/libc/manual/html_node/Infinity-and-NaN.html)):

> **20.5.2 Infinity and NaN**
>
> [...]
>
> The basic operations and math functions all accept infinity and NaN and produce sensible output. Infinities propagate through calculations as one would expect: for example, 2 + &infin; = &infin;, 4/&infin; = 0, atan (&infin;) = &pi;/2. NaN, on the other hand, infects any calculation that involves it. Unless the calculation would produce the same result no matter what real value replaced NaN, the result is NaN.
>
> In comparison operations, positive infinity is larger than all values except itself and NaN, and negative infinity is smaller than all values except itself and NaN. **NaN is unordered: it is not equal to, greater than, or less than anything, including itself. x == x is false if the value of x is NaN.** You can use this to test whether a value is NaN or not, but the recommended way to test for NaN is with the isnan function (see Floating Point Classes). In addition, <, >, <=, and >= will raise an exception when applied to NaNs. 

and which will return `false` for `NaN == NaN`.

That's why Python decides the second `NaN` object is worthy of a new dictionary entry. It may have the same hash, but its address and equivalence test say it is different from all the other `NaN` objects.

However, note that if you always use the same `NaN` object (with the same address) since the address is tested before float equivalence, **you'll get the expected behavior**:

    >>> nan = float('nan')
    >>> d = dict()
    >>> d[nan] = 1
    >>> d[nan] = 2
    >>> d
    {nan: 2}


---


## Question: [Update JSON value for a particular key, preserving JSON structure](https://stackoverflow.com/q/45336340/404556)

I need to update the value for key (`id`) in the JSON file. Value is stored in the variable `ids`. I
am able to update the key `id` with `ids` (updated value), but the structure of the JSON file gets
messed up. Can anyone suggest me a way to doing it without messing up the JSON structure?

    (user's code omitted for brevity)


## Answer

I'm guessing you're referring to the order of keys in you dictionary (later in serialized JSON) that
gets changed. That's because by default `json.load()` uses `dict` as an underlaying mapping type.

But you can change that to a dictionary type that preserves order, called `collections.OrderedDict`:

    from collections import OrderedDict
    
    ids = 10
    filename = 'update_test.json'

    with open(filename, 'r') as f:
        data = json.load(f, object_pairs_hook=OrderedDict)
        data['id'] = ids
    
    with open(filename, 'w') as f:
        json.dump(data, f, indent=4)

Note the use of `object_pairs_hook=OrderedDict` in [`json.load()`](https://docs.python.org/3/library/json.html#json.load). From the docs:

> `object_pairs_hook` is an optional function that will be called with the result of any object
> literal decoded with an ordered list of pairs. The return value of object_pairs_hook will be used
> instead of the `dict`. This feature can be used to implement custom decoders that rely on the
> order that the key and value pairs are decoded (for example,
> [collections.OrderedDict()](https://docs.python.org/3/library/collections.html#collections.OrderedDict)
> will remember the order of insertion).

