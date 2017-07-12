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

