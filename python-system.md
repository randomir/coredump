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

