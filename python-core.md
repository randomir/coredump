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

