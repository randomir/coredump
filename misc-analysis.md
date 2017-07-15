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


