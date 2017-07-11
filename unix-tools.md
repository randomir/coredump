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

