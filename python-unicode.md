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
