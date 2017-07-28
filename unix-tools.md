# Usage examples and problems solved with *nix tools (including GNU)


## Question: [Sorting a file by multiple columns using bash sort](https://stackoverflow.com/q/45042480/404556)

Here is the file I am working with

    word01.2	10	25
    word01.2	30	50
    word01.1	10	30
    word01.1	40	50
    word01.2	40	50
    word01.1	10  20
    word01.1	5	8

When I try my sort command

    sort -k1,1 -k2,2 -k3,3 file.txt 

I receive the following; I don't understand why line 2 and line 1 are not sorted, they should be in reverse positions

    word01.1	10	30
    word01.1	10  20
    word01.1	40	50
    word01.1	5	8
    word01.2	10	25
    word01.2	30	50
    word01.2	40	50

What I would like as results is

    word01.1	5	8
    word01.1	10	20
    word01.1	10  30
    word01.1	40	50
    word01.2	10	25
    word01.2	30	50
    word01.2	40	50


## Answer

You're missing the
[`-n`/`--numeric-sort`](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html#index-numeric-sort)
option, to sort according to string numerical value, not lexicographically (at least for second and
third field):

    $ sort -k1,1 -k2,2n -k3,3n file.txt
    word01.1    5   8
    word01.1    10  20
    word01.1    10  30
    word01.1    40  50
    word01.2    10  25
    word01.2    30  50
    word01.2    40  50

Note that you can provide a global `-n` flag, to sort all fields as numerical values, or per key.
Format for key is `-k KEYDEF`, where `KEYDEF` is `F[.C][OPTS][,F[.C][OPTS]]` and `OPTS` is one or
more of *ordering options*, like `n` (numerical), `r` (reverse), `g` (general numeric), `h` (human
numeric), etc.


---


## Question: [Using `xargs` for parallel processing with multiple argument script](https://stackoverflow.com/q/45064054/404556)

I have a file with values of arguments:

    table1,db1
    table2,db2
    table3,db3
    ...

and I want to call script `test.sh` for each pair of those arguments, using `xargs`, with maximum
of 10 processes running in parallel. Something like this `bash` script, but in parallel:

    while IFS=',' read -r a b; do test.sh "$a" "$b"; done <  file2.txt

I have tried with:

    xargs --max-procs 10 -n 1 sh test.sh <  file2.txt

but the script isn't invoked properly.


## Answer

Your second script, `test.sh`, expects two arguments, but `xargs` is feeding it only one (one word,
in this case the complete line). You can fix it by first converting commas `,` to newlines (with a
simple `sed` script) and then passing two arguments (now two lines) per call to `test.sh` (with
`-n2`):

    sed s/,/\\n/g file2.txt | xargs --max-procs 10 -n2 sh test.sh

Note that `xargs` supports a custom delimiter via `-d` option, and you could use it in case each
line in `file2.txt` were ending with `,` (but then you should probably strip a newline prefixed to
each first field).


---


## Question: [Count pattern in curl output for a list of URLs](https://stackoverflow.com/q/45243043/404556)

How it is possible to count a number of times a pattern occurs in `curl` output for a list of URLs, in `bash`?

For example, if URLs in `url.txt` are:

    xy.com/test.php 
    xy.com/test2.php
    xy.com/test3.php

and both `test.php` and `test2.php` return:

    {"error":null,"result":true}

and I want to count how many times a response `{"error":null,"result":true}` occurs, when I execute a command, to get this:

    $ curl -i url.txt ....
    ...
    ...
    Matched result: 2


## Answer

If I'm understanding your question correctly, you wish to fetch all URLs from the `url.txt` file
(one per line) with `curl` and count the number of times the `{"error":null,"result":true}` string
appears.

You can do it like this:

    $ <url.txt xargs -n1 curl -s -i | grep -F '{"error":null,"result":true}' -c
    2

We pipe the `url.txt` to `xargs` (assuming URLs are properly quoted, without whitespaces) which
calls `curl` with one URL at a time (due to `-n1` option). Progress output from `curl` is silenced
with `-s`, and `-c` tells `grep` to output the count instead of matches, while `-F` looks only for
fixed string given (not pattern matches).


---


## Question: [Bash - Start a new instance of a command in another terminal seperate from your current terminal](https://stackoverflow.com/q/45380896/404556)

I have a simple bash script (test.sh) set up like this:

    #!/bin/bash
    args=("$@")
    if [[ ( ${args[0]} = "check_capture" ) ]]; then
      watch -n 1 'ls -lag /home/user/capture0'
      watch -n 1 'ls -lag /home/user/capture1'
      watch -n 1 'ls -lag /home/user/capture2' 
      exit
    fi

Files are continuously being written to these target locations capture 0, capture 1, and capture 3.
I want to be able to watch these directories using ls command continuously on 3 seperate terminals,
and once I run this script (test.sh) from the current terminal, I want it to exit. 

Right now it is blocked by each wait, which I know is a blocking bash command waiting for user input
control-c. Is there a way I can have the 3 watch commands be executed in seperate terminals then
reach the exit statement?


## Answer

You can start several instances of the terminal in background, each one running a command, like this:

    if [[ ... ]]; then
        xterm -e 'watch -n 1 "ls -lag /home/user/capture0"' &
        xterm -e 'watch -n 1 "ls -lag /home/user/capture1"' &
        ...
        exit
    fi

Check `man xterm`:

> **`-e program [ arguments ... ]`**
>
> This  option  specifies the program (and its command line arguments) to be run in the xterm
> window.  It also sets  the window title and icon name to be the basename of the program being
> executed if neither -T nor -n are given on the command line.  This must be the last option on the
> command line.

The same option works also for `xfce-terminal` and `gnome-terminal`.

In addition, `xterm` (and others) also support setting the title of the window, position, size
(called geometry), colors, fonts, and *many, many* other features.

