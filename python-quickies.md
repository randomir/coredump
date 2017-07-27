# Simple Python ops


## Question: [Combine multiple lists in python to single list](https://stackoverflow.com/questions/44599183/combine-multiple-lists-in-python-to-single-list/)

I have a function that combines two lists in a new list. What is the most pythonic way to extend
this from just 2 input parameters to an arbitrary number of lists as input. The output should still
be a list of lists.

The expected answer for `lt1 = [1, 2, 3]`, `lt2 = [4, 5, 6]` should be `[[1, 2, 3], [4, 5, 6]]`


## Answer

If I understand your question, you want something like this:

    >>> def combine(*args):
    ...     return list(args)
    ... 
    >>> print combine([1,2,3], ['a','b','c'], [True, False])
    [[1, 2, 3], ['a', 'b', 'c'], [True, False]]

So, we're using [positional arguments][1] from ``args`` (which are stored in a ``tuple``) and
converting them to a list. It works for any number of arguments, and it'll happily combine anything
(not just lists) into a list.


  [1]: https://docs.python.org/3/reference/expressions.html#calls "positional arguments"


---


## Question: [Split string with lookahead/lookbehind with empty string](https://stackoverflow.com/questions/44729923/split-string-with-lookahead-lookbehind-with-empty-string/)

I'm trying to split any number string such as `3.1415926535897932384626433832795028841971` right
after each `0` or group of `0`. However, I would like to keep the 0 after each group.

For example, the string `10203040506070809011` should be split into

    ['10', '20', '30', '40', '50', '60', '70', '80', '90', '11']

and the string `3.1415926535897932384626433832795028841971` should be split into

    ['3.14159265358979323846264338327950', '28841971']

Is there any way to split a string based on a lookahead or lookbehind on an empty string? I'm asking
about the general case, not just with numbers. For example, if I wanted to split apart
`3:18am5:19pm10:28am` into the separate times without losing the `am` or `pm`, and get an array
`['3:18am', '5:19pm', '10:28am']`, how would I go about doing this?


## Answer

This simple regex in [``re.findall``](https://docs.python.org/3/library/re.html#re.findall) should suffice:

    l = re.findall(r'[.1-9]+(?:0+|$)', s)

Note:

- ``findall`` returns **all non-overlapping matches** of pattern in string, **as a list** of strings.

- for each match we want the longest string of digits (or a dot) ending with at least one zero, or the end of the string

- the zeros in the end should not be captured as another match (hence the ``(?:...``)

Similarly for you second example:

    >>> re.findall(r'[\d:]+(?:am|pm|$)', '3:18am5:19pm10:28am')
    ['3:18am', '5:19pm', '10:28am']

No need for lookahead/lookbehind magic, or non-greedy matching.


---


## Question: [Remove intersection from two lists in python](https://stackoverflow.com/questions/44741442/remove-intersection-from-two-lists-in-python/)

Given two lists, what is the best way to remove the intersection of the two?  For example, given:
   
    a = [2,2,2,3]
    b = [2,2,5]

I want to return:
    
    a = [2,3]
    b = [5]


## Answer

Let's assume you wish to handle the general case (same elements appear more than once in each list), the so called **multiset**.

You can use [``collections.Counter``](https://docs.python.org/3/library/collections.html#collections.Counter):

    from collections import Counter
    intersection = Counter(a) & Counter(b)
    multiset_a_without_common = Counter(a) - intersection
    multiset_b_without_common = Counter(b) - intersection
    new_a = list(multiset_a_without_common.elements())
    new_b = list(multiset_b_without_common.elements())

For your values of ``a``, ``b``, you'll get:

    a = [2,2,2,3]
    b = [2,2,5]
    new_a = [2, 3]
    new_b = [5]

Note that for a special case of each element appearing exactly once, you can use the standard [``set``](https://docs.python.org/2/library/stdtypes.html#set), as the other answers are suggesting.


---


## Question: [Sorting arrays in Python by a non-integer column](https://stackoverflow.com/q/44871397/404556)

So for example I have an array that I want to sort by a column in an ascending order, and it's easy
to do for integers using 'sorting()', 'np.arrange()', or 'np.argsort()'.

However, what if my column is consisting of floats?

I mean, I have something like:

    a = array([[1.7, 2, 3],
               [4.5, 5, 6],
               [0.1, 0, 1]])

and I want to get this:

    array([[0.1, 0, 1],
           [1.7, 2, 3],
           [4.5, 5, 6]])


## Answer

You can use a standard Python's [`sorted`](https://docs.python.org/2/library/functions.html#sorted)
(or `sort` for in-place sorting), no matter what is contained in the sequence. Just use a custom
`key`, or a custom compare function (`cmp`). For example, to sort a list of lists (2-d array)
ascending by 4th column:

    >>> a=[[1.0,2.0,3.0,4.0], [4.0,3.0,2.0,1.0], [0,0,0,0]]
    >>> from operator import itemgetter
    >>>> sorted(a, key=itemgetter(3))
    [[0, 0, 0, 0], [4.0, 3.0, 2.0, 1.0], [1.0, 2.0, 3.0, 4.0]]


---


## Question: [How to rebuild the object with repr?](https://stackoverflow.com/q/44894261/404556)

My code works perfectly except the last part. I want to recreate the object with repr function but it clearly doesn't work. I tried everything here and on the web but i'm still so confuse. Is there any way to do it and if so what is the syntax ?

    class Modulo(object):
    
        def __init__(self, grondtal, waarde = 0):
            self.grondtal = grondtal
            self.waarde = waarde % grondtal
    
        def __call__(self, m):
            return Modulo(self.grondtal, m)
    
        def __add__(self, other):
            return Modulo(self.grondtal, self.waarde + other.waarde)
    
        def __sub__(self, other):
            return Modulo(self.grondtal, self.waarde - other.waarde)
    
        def __mul__(self, other):
            return Modulo(self.grondtal, self.waarde * other.waarde)
    
        def __eq__(self, other):
            return self.waarde == other.waarde and self.grondtal == other.grondtal
    
        def __ne__(self, other):
            return not self.__eq__(other)
    
        def __str__(self):
            return  '[%s %% %s]' % (str(self.grondtal), str(self.waarde))
    
        def __repr__(self):
            return '%s' %Modulo(self.grondtal, self.waarde)


## Answer

You probably want this:

    def __repr__(self):
        return "Modulo(%d,%d)" % (self.grondtal, self.waarde)

Or, a little bit more generic:

    def __repr__(self):
        return "%s(%d,%d)" % (self.__class__.__name__, self.grondtal, self.waarde)

For example:

    >>> m = Modulo(3,2)
    >>> repr(m)
    'Modulo(3,2)'    


---


## Question: [Extract 8-digit numbers from a list of strings](https://stackoverflow.com/q/44958629/404556)

I have a list of strings which may contain letters, symbols, digits, etc, as below:

    list = ['\n', '', '0', '38059', '', '', '?_', '71229366', '', '1', '38059', '', '', '?_', '87640804', '', '2', '38059', '', '', '?_', '71758011', '', '', ':?', ';__', '71229366287640804271758011287169822']

How do I filter out all other strings, except numbers less than 10000000 and greater than 99999999?

Expected Output:

    list = ['71229366', '87640804', '71758011']


## Answer

Let me provide a **simple and efficient** answer, using regular expressions. There's no need to
`map` (duplicating the original list), or to convert everything to `int`s; you are basically asking
how **to keep all 8-digit integers** in your list:

    >>> filter(re.compile('^\d{8}$').match, data)
    ['71229366', '87640804', '71758011']

We [`compile`](https://docs.python.org/3/library/re.html#re.compile) a regular expression which
matches exactly 8 digits and then filter the list by providing a partial application of
[`regex.match`](https://docs.python.org/3/library/re.html#re.regex.match) to the standard
[`filter`](https://docs.python.org/3/library/functions.html#filter) function.


---


## Question: [How to generate string dynamically in Python](https://stackoverflow.com/q/45001530/404556)

Supposing we have the code below:

    var1 = "top"
    var2 = var1 + "bottom"

We want to change `var1` value if a condition is true:

    if COND:
      var1 = "changed"

Now I want to have `var2` **dynamically changed**. With the code above, `var2` will still have the value "`topbottom`".

How can I do that?


## Answer

You can elegantly achieve this with a callback proxy from [`ProxyTypes`](https://pypi.python.org/pypi/ProxyTypes/0.9) package:

    >>> from peak.util.proxies import CallbackProxy
    >>> var2 = CallbackProxy(lambda: var1+"bottom")
    >>> var1 = "top"
    >>> var2
    'topbottom'
    >>> var1 = "left"
    >>> var2
    'leftbottom'

Each time you access your `var2`, callback lambda will be executed and a dynamically generated value returned.


---


## Question: [How to split string at predefined indices?](https://stackoverflow.com/q/45104747/404556)

I have a string that I'd like to split in specific places into a list of strings. The split points
are stored in a separate split list. For example:

    test_string = "thequickbrownfoxjumpsoverthelazydog"
    split_points = [0, 3, 8, 13, 16, 21, 25, 28, 32]

...should return:

    >>> ['the', 'quick', 'brown', 'fox', 'jumps', 'over', 'the', 'lazy', 'dog']

So far I have this as the solution, but it looks incredibly convoluted (solution omitted).


## Answer

Like this?

    >>> map(lambda x: test_string[slice(*x)], zip(split_points, split_points[1:]+[None]))
    ['the', 'quick', 'brown', 'fox', 'jumps', 'over', 'the', 'lazy', 'dog']

We're `zip`ing `split_points` with a shifted self, to create a list of all consecutive pairs of
slice indexes, like `[(0,3), (3,8), ...]`. We need to add the last slice `(32,None)` manually, since
`zip` terminates when the shortest sequence is exhausted.

Then we `map` over that list a simple lambda slicer. Note the `slice(*x)` which creates a
[`slice`](https://docs.python.org/2/library/functions.html#slice) object, e.g. `slice(0, 3, None)`
which we can use to slice the sequence (string) with standard the
[item getter](https://docs.python.org/2/library/operator.html#operator.getitem) (`__getslice__` in Python 2).

A little bit more Pythonic implementation could use a list comprehension instead of `map`+`lambda`:

    >>> [test_string[i:j] for i,j in zip(split_points, split_points[1:] + [None])]
    ['the', 'quick', 'brown', 'fox', 'jumps', 'over', 'the', 'lazy', 'dog']


---


## Question: [Remove chars from string using Regular Expression](https://stackoverflow.com/q/45266640/404556)

Given an array of strings which contains alphanumeric characters but also punctuations that have to
be deleted. For instance the string `"0-001"` is converted into `"0001"`. (user's attempt omitted)


## Answer

If all you want to do is remove non-alphanumeric characters from a string, you can do it simply with
[`re.sub`](https://docs.python.org/3/library/re.html#re.sub):

    >>> re.sub('\W', '', '0-001')
    '0001'

Note, the `\W` will match any character which is not a Unicode word character. This is the opposite
of `\w`. For ASCII strings it's equivalent to `[^a-zA-Z0-9_]`.


---


## Question: [How to iterate over multiple slices of a list](https://stackoverflow.com/q/45280979/404556)

    for fi in files[0:10 ??? 605:615]:
        print fi

How can I add `[605:615]` to `[0:10]`?


## Answer

You could use [`itertools.chain`](https://docs.python.org/3/library/itertools.html#itertools.chain):

    from itertools import chain
    
    for fi in chain(files[0:10], files[605:615]):
        print fi

`itertools.chain` will make an iterator that will return all elements from the first iterable, then
from the second, third, etc.


---


## Question: How to split string on more than one delimiter?

How to split string `"10,0902\n13897,00641"` on both `\n` and `,`?

## Answer

With [`re.split`](https://docs.python.org/3/library/re.html#re.split) you can split by any regular
expression pattern. So you can use the alternation `\n|,`:

    >>> string = "10,0902\n13897,00641"
    >>> re.split(r'\n|,', string)
    ['10', '0902', '13897', '00641']

or character range, `[\n,]`:

    >>> re.split(r'[\n,]', string)
    ['10', '0902', '13897', '00641']

