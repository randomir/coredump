# On Unicode in Python


## Question: [Replacing more than one character in unicode python](https://stackoverflow.com/questions/44701267/replacing-more-than-one-character-in-unicode-python/)

    import re
    test = unicode("شدَد", encoding='utf-8')
    test = test.replace(u"\u064e", "")

This is the code to remove one character. I would like to replace any of the following unicode characters: 0622, 0623, 0625 with 0627. This is for the Arabic language. I know how to do it in multiple lines but is there a way to do it in one?

## Answer

If you want multiple characters (unicode code points) to be replaced *in a oneliner*, you can use a
simple alternation [regex](https://docs.python.org/2/library/re.html#re.sub):

    import re
    test = unicode("شدَد", encoding='utf-8')
    test = re.sub(u"\u064e|\u0634", "", test,  flags=re.UNICODE)

Or, with a range regex:

    test = re.sub(u"[\u064e\u0634]", "", test,  flags=re.UNICODE)


---


## Question: [Print special characters in list in Python](https://stackoverflow.com/q/45001908/404556)

I have a list containing special characters (for example `é` or a white space) and when I print the
list these characters are printed with their Unicode code, while they are printed correctly if I
print the list elements separately:

    #!/usr/bin/env python
    # -*- coding: utf-8 -*-

    my_list = ['éléphant', 'Hello World']
    print(my_list)
    print(my_list[0])
    print(my_list[1])

The output of this code is

    ['\xc3\xa9l\xc3\xa9phant', 'Hello World']
    éléphant
    Hello World

And I would like to have `['éléphant', 'Hello World']` for the first output. What should I change?


## Answer

If possible, switch to Python 3 and you'll get the expected result.

If you have to make it work in Python 2, then use `unicode` strings:

    my_list = [u'éléphant', u'Hello World']

The way you have it right now, Python is interpreting the first string as a series of bytes with
values `'\xc3\xa9l\xc3\xa9phant'` which will only be converted to Unicode code points after properly
UTF-8 decoded: `'\xc3\xa9l\xc3\xa9phant'.decode('utf8') == u'\xe9l\xe9phant'`.

If you wish to print list `repr` and get "unicode" out, you'll have to manually encode it as UTF-8
(if that's what your terminal understands).

    >>> print repr(my_list).decode('unicode-escape').encode('utf8')
    [u'éléphant', u'Hello World']

But it's easier to format it manually:

    >>> print ", ".join(my_list)
    éléphant, Hello World


---


## Question: [How to convert python Unicode string to bytes](https://stackoverflow.com/q/45074655/404556)

I have a string `x` as below

    x = "\xe9\x94\x99\xe8\xaf\xaf"

This string should be Unicode string, but cannot be displayed (print) correctly.

And the string `y` is Unicode string/ bytes started with `b`, And `y` can be displayed correctly by `y.decode('utf-8')`

    y = b"\xe9\x94\x99\xe8\xaf\xaf"

My question is how to convert x to y ?


## Answer

Assuming we're talking about Python3, the Unicode string `x` is 6 code points long. It happens to be
that each of those code points is in range `0x00` to `0xff` (ASCII subset). We can get the exact
byte string with the [`raw_unicode_escape`](https://docs.python.org/3/library/codecs.html#text-encodings)
codec, like this:

    >>> x = "\xe9\x94\x99\xe8\xaf\xaf"
    >>> y = x.encode('raw_unicode_escape')
    >>> y
    b'\xe9\x94\x99\xe8\xaf\xaf'
    >>> y.decode('utf8')
    '错误'

Note that this will only work if the string `x` contains only ASCII subrange of Unicode; otherwise
you'll just get escaped Unicode code points (as the codec's name suggests):

    >>> "šž".encode('raw_unicode_escape')
    b'\\u0161\\u017e'
