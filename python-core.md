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

