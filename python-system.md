# System-interactions from Python


## Question: [Python script in linux shell that checks if a command line writes into file](https://stackoverflow.com/q/44907875/404556)

I'm writing a python script that print output into the screen (linux shell), and I'm printing it with colors.
Is there a way to know if the output goes into a file or not?

Example:

    script.py parms
this gives me good colored output in the shell

Now if I do this:

    script.py parms > output.txt

when I open the file I see weird ASCII characters (the colors values), I've tried to open it in a few text editors (kate, gedit).

I want to do something like:

    if goesIntoFile:
      print in black
    else:
      print in color

How can I do it?


## Answer

You could use [`isatty()`](https://docs.python.org/2/library/os.html#os.isatty) on your `stdout` to
check if the standard output is a `tty` (terminal) device, or a file.

Check this `script.py`:

    #!/usr/bin/env python
    import sys
    print sys.stdout.isatty()

When run:

    $ python script.py
    True
    $ python script.py | cat
    False

Also, you might want to check some of the color-output libraries which handle this for you, for
example [crayons](https://pypi.python.org/pypi/crayons) uses the `isatty` approach above.


---


## Question: [Fastest way to read a binary file with a defined format?](https://stackoverflow.com/q/44933639/404556)

I have large binary data files that have a predefined format, originally written by a Fortran program as little endians. I would like to read these files in the fastest, most efficient manner, so using the [`array`][1] package seemed right up my alley as suggested [here][2].

The problem is the pre-defined format is non-homogeneous. It looks something like this:
`['<2i','<5d','<2i','<d','<i','<3d','<2i','<3d','<i','<d','<i','<3d']`

with each integer `i` taking up 4 bytes, and each double `d` taking 8 bytes.

Is there a way I can still use the super efficient `array` package (or another suggestion) but with the right format?

  [1]: https://docs.python.org/3/library/array.html
  [2]: https://stackoverflow.com/questions/5804052/improve-speed-of-reading-and-converting-from-binary-file-with-python


## Answer

It's not clear from your question whether you're concerned about the actual file **reading** speed
(and building data structure in memory), or about later data **processing** speed.

If you are reading only once, and doing heavy processing later, you can read the file record by
record (if your binary data is a recordset of repeated records with identical format), parse it with
`struct.unpack` and append it to a `[double]` array:

    from functools import partial

    data = array.array('d')
    record_size_in_bytes = 9*4 + 16*8   # 9 ints + 16 doubles

    with open('input', 'rb') as fin:
        for record in iter(partial(fin.read, record_size_in_bytes), b''):
            values = struct.unpack("<2i5d...", record)
            data.extend(values)

Under assumption you are allowed to cast all your `int`s to `double`s *and* willing to accept
increase in allocated memory size (22% increase for your record from the question).

If you are reading the data from file many times, it could be worthwhile to convert everything to
one large `array` of `double`s (like above) and write it back to another file from which you can
later read with
[`array.fromfile()`](https://docs.python.org/3/library/array.html#array.array.fromfile):

    data = array.array('d')
    with open('preprocessed', 'rb') as fin:
        n = os.fstat(fin.fileno()).st_size // 8
        data.fromfile(fin, n)

**Update**. Thanks to a nice [benchmark by @martineau](https://stackoverflow.com/a/45019271/404556),
now we know for a fact that preprocessing the data and turning it into an homogeneous array of
doubles ensures that loading such data from file (with `array.fromfile()`) is `~20x to ~40x` faster
than reading it record-per-record, unpacking and appending to `array` (as shown in the first code
listing above).

A faster (and a more standard) variation of record-by-record reading in @martineau's answer which
appends to `list` and doesn't upcast to `double` is only `~6x to ~10x` slower than
`array.fromfile()` method and seems like a better reference benchmark.

