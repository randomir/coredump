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

