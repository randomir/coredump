# On Python performace


## Question: [List comprehension vs filter vs remove](https://stackoverflow.com/q/45042801/404556)

How are the latter two better than the former solution?

    primes = (1, 2, 3, 5, 7)

    # Classic solution
    items = list(range(10))
    for prime in primes:
        items.remove(prime)
    items

    # List comprehension
    items = list(range(10))
    [item for item in items if item not in primes]

    # Filter
    items = list(range(10))
    list(filter(lambda item: item not in primes, items))

The three examples are something I came across in a book and it says that the first solutions takes
`O(n*m)` time (n=len(items), m=len(primes)) whereas the latter two take `O(n*1)` time... Resulting
in 50 comparisons for the first solution (slightly better actually - 40 comparisons) and just 10 for
the latter.

I do not understand this. I don't understand how could it be time or memory efficient.

Here is the paragraph in the book that explains this:
   
>    To remove or insert a single item from/into the list, Python needs to copy 
    the entire list, which is especially heavy with larger lists. When executing 
    this only once, it is of course not all that bad. But when executing a large 
    number of deletions, a filter or list comprehension is a much faster 
    solution because, if properly structured, it needs to copy the list only
    once.
    .... then the examples ...
    The latter two are much faster for large lists of items. This is because the 
    operations are much faster. To compare using n=len(items) and m=len(primes), 
    the first takes O(m*n)=5*10=50 operations, whereas the latter two take 
    O(n*1)=10*1=10 operations.

EDIT: The book is not wrong. `primes = set((1, 2, 3, 5, 7))` is the right declaration and not
`primes = (1, 2, 3, 5, 7)`.


## Answer

If the code in the book is exactly the same as you posted, then **the book is flat out wrong**.

The first example has time complexity `O(n*m)`, but so do the other two.

If `primes` were a `set` (or `dict`), then it would be true -- existence lookup with `in` operator
in a hashmap has time complexity `O(1)`, but in a `list` or `tuple` has `O(n)`! Therefore, the total
complexity of `O(n*m)`.

Let's check this with some measurements:

    t = tuple(range(10000))
    l = list(t)
    s = set(t)
    d = {i:1 for i in l}
    
    In [16]: %%timeit
    4738 in t
       ....: 
    10000 loops, best of 3: 45.5 µs per loop
    
    In [17]: %%timeit
    4738 in l
       ....: 
    10000 loops, best of 3: 45.4 µs per loop
    
    In [18]: %%timeit
    4738 in s
       ....: 
    10000000 loops, best of 3: 36.9 ns per loop

    In [19]: %%timeit
    4738 in d
       ....: 
    10000000 loops, best of 3: 38 ns per loop

Notice the lookup in `set` is `~37ns` (similar as in `dict`), 3 orders of magnitude faster than in
`list`/`tuple`, `~45us`.


---


## Question: [python: class vs tuple huge memory overhead](https://stackoverflow.com/q/45123238/404556)

I'm storing a lot of complex data in tuples/lists, but would prefer to use small wrapper classes to
make the data structures easier to understand, e.g.

    class Person:
        def __init__(self, first, last):
            self.first = first
            self.last = last
    
    p = Person('foo', 'bar')
    print(p.last)
    ...

would be preferable over

    p = ['foo', 'bar']
    print(p[1])
    ...

*however* there seems to be a horrible memory overhead:

    l = [Person('foo', 'bar') for i in range(10000000)]
    # ipython now taks 1.7 GB RAM

and 

    del l
    l = [('foo', 'bar') for i in range(10000000)]
    # now just 118 MB RAM

Why? is there any obvious alternative solution that I didn't think of?

Thanks!

(I know, in this example the 'wrapper' class looks silly. But when the data becomes more complex and
nested, it is more useful)


## Answer

As others have said in their answers, you'll have to generate different objects for the comparison
to make sense.

So, let's compare some approaches.

### `tuple`

    l = [(i, i) for i in range(10000000)]
    # memory taken by Python3: 1.0 GB

### `class Person`

    class Person:
        def __init__(self, first, last):
            self.first = first
            self.last = last

    l = [Person(i, i) for i in range(10000000)]
    # memory: 2.0 GB

### `namedtuple` (`tuple` + `__slots__`)

    from collections import namedtuple
    Person = namedtuple('Person', 'first last')

    l = [Person(i, i) for i in range(10000000)]
    # memory: 1.1 GB

`namedtuple` is basically a class that extends `tuple` and uses `__slots__` for all named fields,
but it adds fields getters and some other helper methods (you can see the exact code generated if
called with `verbose=True`).

### `class Person` + `__slots__`

    class Person:
        __slots__ = ['first', 'last']
        def __init__(self, first, last):
            self.first = first
            self.last = last

    l = [Person(i, i) for i in range(10000000)]
    # memory: 0.9 GB

This is a trimmed-down version of `namedtuple` above. A clear winner, even better than pure tuples.
