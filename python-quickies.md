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
