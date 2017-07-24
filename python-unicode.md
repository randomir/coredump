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


---


## Question: [Python's UTF-8 encoding yields odd results even though explicit utf-8 encoding is used](https://stackoverflow.com/q/45286324/404556)

I am parsing some JSON (specifically Amazon reviews file, which Amazon publicly provides). I am doing a line by line parsing with conversion to Pandas DataFrame and insert to SQL on the fly. I found something really odd. I use UTF-8 to open the json file. In the file itself when I open it with notepad I don't see any strange symbols or whatever. For example, substring of review:

    The temperature control doesn’t hold to as tight a temperature as some of the others reported.

But when I parse it and check the contents of string:

    The temperature control doesn\xe2\x80\x99t hold to as tight a temperature as some of the others reported. 

Why is that so? How I can't properly read it?

My current code is below:

    def parseJSON(path):
      g = io.open(path,'r',encoding='utf8')
      for l in g:
          yield eval(l)
    
    for l in parseJSON(r"reviews.json"):
        for review in l["reviews"]:
            df = {}
            df[l["url"]] = review["review"]
            dfInsert = pd.DataFrame( list(df.items()), columns = ["url", "Review"])


## Answer

First of all, you should never parse a text from an unsafe (online) source with [`eval`](https://docs.python.org/2/library/functions.html#eval). If the data is in [JSON](http://json.org), you should use a JSON parser. That's why JSON was invented - to provide a safe serialization and deserialization.

In your case, use [`json.load()`](https://docs.python.org/2/library/json.html#json.load) from the standard [`json`](https://docs.python.org/2/library/json.html) module:

    import json

    def parseJSON(path):
        return json.load(io.open(path, 'r', encoding='utf-8-sig'))

Since your JSON file contains a [BOM](https://en.wikipedia.org/wiki/Byte_order_mark), you should use the codec that knows how to strip it, i.e. the [`utf-8-sig`](https://docs.python.org/2/library/codecs.html#encodings-and-unicode).

If your file contains **one JSON Object per line**, you can read it like this:

    def parseJSON(path):
        with io.open(path, 'r', encoding='utf-8-sig') as f:
            for line in f:
                yield json.loads(line)


---

Now to answer why are you seeing `doesn\xe2\x80\x99t` instead of `doesn’t`. If you decode the bytes `\xe2\x80\x99` as UTF-8, you get:

    >>> '\xe2\x80\x99'.decode('utf8')`
    u'\u2019'

and what Unicode codepoint is that?

    >>> unicodedata.name(u'\u2019')
    'RIGHT SINGLE QUOTATION MARK'

Ok, now what happens when you `eval()` it in Python 2? Well, first, note that Unicode is not really a first-class citizen in the land of Python 2 strings (Python 3 fixed that).

So, `eval` tries to parse the string (series of bytes in Python 2) as a Python expression:

    >>> eval('"’"')
    '\xe2\x80\x99'

Note that (in my console that uses UTF-8) even when I type `’`, that's represented as a sequence of 3 bytes. 

It doesn't even help to say it's supposed to be a `unicode`:

    >>> eval('u"’"')
    u'\xe2\x80\x99'

What will help is to tell Python how to interpret the series of bytes that follow in the source/string, i.e. what's the encoding (see [PEP-263](https://www.python.org/dev/peps/pep-0263/)):

    >>> eval('# encoding: utf-8\nu"’"')
    u'\u2019'
