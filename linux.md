# On Linux, lower level stuff


## Question: [How to unbuffer stdout of legacy running binary without stdbuf and similar tools](https://stackoverflow.com/q/45460998/404556)

I want to monitor the realtime output of a program that I will start. I am trying to do this by
redirecting the output of the program to a pipe and then reading the pipe from a monitoring script. 

    ./program >> apipe

then from the monitoring script

    cat apipe

However due to the buffer in >> there is no output. Anyway I can disable this buffer? I am running
on a barebones embedded system (petalinux) so I don't have access to unbuffer, script, or stdbuf to
help me out. 

I have tried the scripts on another platform where unbuffer is available it works as I expect.

Any way I can configure this buffer, or use another binary to redirect?

I do not have access to the source code of the command I am trying to run. It is a legacy binary.


## Answer

If you don't have access to `stdbuf`, you might as well simulate it and **unbuffer the `stdout`
manually** with `gdb` (assuming obviously you have access to `gdb`).

Let's take a look at how `stdbuf` actually operates. The
[`stdbuf`](https://github.com/coreutils/coreutils/blob/master/src/stdbuf.c) GNU coreutils command
basically only injects `libstdbuf` in the user program by setting `LD_PRELOAD` environment variable.
(Irrelevant, but for the record, options are passed via `_STDBUF_E`/`_STDBUF_I`/`_STDBUF_O` env
vars.)

Then, when the [`libstdbuf`](https://github.com/coreutils/coreutils/blob/master/src/libstdbuf.c) is
run, it calls [`setvbuf`](https://www.gnu.org/software/libc/manual/html_node/Controlling-Buffering.html) libc
function (which in turn executes the underlaying syscall) on appropriate file descriptors
(`stdin`/`stdout`/`stderr`), with the appropriate mode (fully buffered, line buffered, or
unbuffered).

Declaration for `setvbuf` is in `stdio.h`, available with `man 3 setvbuf`:

    #include <stdio.h>
    
    int setvbuf(FILE *stream, char *buf, int mode, size_t size);

Values for `mode` are: `_IONBF`, `_IOLBF`, `_IOFBF`, as defined in `stdio.h`. We are here only
interested in the unbuffered mode: `_IONBF`. It has a value of `2` (you can check your
`/usr/include/stdio.h`).

### The "unbuffer" script

So, to unbuffer a `stdout` for some process, we just need to call:

    setvbuf(stdout, NULL, _IONBF, 0)

We can easily do that with `gdb`. Let's make a script we can call, `unbuffer-stdout.sh`:

    #!/bin/bash
    # usage: unbuffer-stdout.sh PID

    gdb --pid "$1" -ex "call setvbuf(stdout, 0, 2, 0)" --batch

Then, we can call it like:

    $ ./unbuffer-stdout.sh "$(pgrep -f my-program-name)"

(You'll probably need `sudo` to run it as `root`.)

### Testing

We can use this simple Python program with buffered standard output (if not called with `-u`, and
with unset `PYTHONUNBUFFERED`), `writer.py`:

    #!/usr/bin/python
    import sys, time
    
    while True:
        sys.stdout.write("output")
        time.sleep(0.5)

Run it with:

    $ ./writer.py >/tmp/output &
    $ tailf /tmp/output

and observe no output appears until we run:

    $ sudo ./unbuffer-stdout.sh "$(pgrep -f writer.py)"

