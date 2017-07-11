# How to analyse computing problems


## Question: [Generate Image using Text](https://stackoverflow.com/questions/44088393/generate-image-using-text/)

I visited this website,
[https://xcode.darkbyte.ru/][1]

[![enter image description here][2]][2]

Basically the website takes a text as Input and generates an Image.
It also takes an image as input and decodes it back to text.
I really wish to know what this is called and how it is done
I'd like to know the algorithm [preferably in Java]
Please help, Thanks in advance

  [1]: https://xcode.darkbyte.ru/
  [2]: https://i.stack.imgur.com/8AVmx.png

## Answer

There are many ways to encode a text (series of bytes) as an image, but the site you quoted does it in a pretty simple and straightforward way. And you can reverse-engineer it easily:

 - Up to 3 chars are coded as 1 pixel; 4 chars as 2 pixels -- we learn from this that only R(ed), G(reen) and B(lue) channels for each pixel are used (and not alpha/transparency channel).

 - We know PNG supports 8 bits per channel, and each ASCII char is 8 bits wide. Let's test if first char (first 8 bits) are stored in red channel.
   - Let's try `z..`. Since `z` is relatively high in ASCII table (`122`) and `.` is relatively low (`46`) -- we expect to get a redish 1x1 PNG. And we do.
   - Let's try `.z.`. Is should be greenesh.. And it is.
   - Similarly for `..z` we get a bluish pixel.

 - Now let's see what happens with a non-ASCII input. Try entering: `①` (unicode char `\u2460`). The site [html-encodes][1] the string into `&#9312;` and then encodes that ASCII text into the image as before.

 - Compression. When entering a larger amount of text, we notice the output is shorter then expected. It means the back-end is running some compression algorithm on raw input before (or after?) encoding it as image. By noticing the resolution of the image and maximum information content (HxWx3x8 bits) being smaller than input, we can conclude the compression is done before encoding to image, and not after (thus not relying to PNG compression). We could go further in detecting which compression algorithm is used by encoding the raw input with the common culprits like [Huffman coding](https://en.wikipedia.org/wiki/Huffman_coding), [Lempel-Ziv](https://en.wikipedia.org/wiki/LZ77_and_LZ78), [LZW](https://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Welch), [DEFLATE](https://en.wikipedia.org/wiki/DEFLATE), even [Brotli](https://en.wikipedia.org/wiki/Brotli), and comparing the output with bytes from image pixels. (Note we can't detect it directly by inspecting a magic prefix, chances being author stripped anything but the raw compressed data.)

  [1]: https://en.wikipedia.org/wiki/Unicode_and_HTML


---


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
