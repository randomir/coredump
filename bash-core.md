# Generic Bash problems


## Question: [Bash regex matching group](https://stackoverflow.com/q/44872421/404556)

So i have a regex matching particular package i.e org.package.util
I also need to catch packages which are like : 
**"org.package.util.ca# lc** , **org.package.util.di#v**, **org.package.util.http11** and so on" after it "." ,"#", "[0-9]" , [a-z] and spaces 
Currently this part of my regex is `\([0-9a-z.#\s]*\)*` but it only seems to catch alphabetical chars if there are no **# (' ') or digits**


## Answer

As `man bash` says:

> An additional binary operator, =~, is available, with the same precedence as == and !=.  When it is used, the  string  to the  right  of  the  operator is considered an **extended regular expression** and matched accordingly (as in regex(3)).

So, in Bash, you must use the extended regex ([POSIX ERE](http://pubs.opengroup.org/onlinepubs/009695399/basedefs/xbd_chap09.html)), and `\s` is a whitespace character class in [PCRE](http://www.pcre.org/original/doc/html/pcrepattern.html), with `[:space:]` being the POSIX equivalent.

Note that in PCRE, you can also use POSIX character classes (`[:digit:]`, `[:alpha:]`, etc.), along with PCRE classes (`\d`, `\w`, etc).

If you need a more advanced regex features (like back references, lookahead/lookbehind assertions, conditional subpatterns, recursive patterns, etc.) you can enable PCRE support in `grep` with the `-P` flag.

Also, from the question is not clear, but is seems you're escaping the grouping parenthesis and writing your regex directly inside the condition (like: `[[ "$text" =~ \([a-z]+\) ]]`). It's a good practice to store your regex in a variable (like `re='([a-z]+)'`) and use it like `[[ "$text" =~ $re ]]`. That way your expression will be much clearer and easier to maintain, since you only write ERE syntax and avoid the need for shell-escaping.


---


## Question: [How can I use bash's select on a newline-separated list?](https://stackoverflow.com/q/44958941/404556)

How can I use bash's `select` on a newline-separated list (so list items can contain spaces)?


## Answer

By setting `IFS=$'\n'`, your `list` will be split on newlines, for example:

    #!/bin/bash
    list=$'one\ntwo\nthree'

    IFS=$'\n'
    select item in $list; do
        case "$item" in
            one) echo "1";;
            two) echo "2";;
            *) break;;
        esac
    done

Be sure to either do this in a function with `local IFS`, or to restore `IFS` later manually.

If you expect your filenames (in the list) to contain pathname expansion pattern characters `*`,
`?`, or `[`, you should wrap the `select` statement block above with `set -f` and `set +f` which
will prevent those patterns to expand to pathnames. It'll also prevent the expansion of extended
globs (if you enabled `extglob`).


---


## Question: [Accept newline in bash `case`](https://stackoverflow.com/q/44991587/404556)

I have the following script that continuously accepts input from the user until he/she enters some
form of `y*` or `Y*`.

    while true; do
    read -p "Are you ready? " yn
    case $yn in
    	[Yy]* ) break;;
    	[Nn]* ) ;;
    	* ) echo "Please answer yes or no.";;
    esac
    done

However, I would like to break out of the while loop when the user simply presses `enter`. I tried
using `\n`, `\r`, and `\r\n` but these don't seem to be the right patterns.


## Answer

Because `read` strips the newline, you'll have to match an empty string, `""`:

    #!/bin/bash
    while true; do
        read -p "Are you ready? " yn
        case "$yn" in
            [Yy]*|"") break;;
            [Nn]*) ;;
            *) echo "Please answer yes or no.";;
        esac
    done

Depending on your application logic, you can (obviously) make that a separate case branch, like:
`"") break;;`.


---


## Question: [Reading files with leading zero bash command line](https://stackoverflow.com/q/45015515/404556)

For example I have 15 files as follows:

    abc01.txt, abc02.txt, ..., abc09.txt, abc10.txt, abc11.txt, ..., abc15.txt

I want to read these files from command line using bash and perform some operation.

    for i in {1..15}; do COMMAND abc$i.txt; done

Above statement only reads files from 10 to 15 because of leading 0 for the first nine files. If I
use `[0]` before $ in the above command then it only reads first 9 files. I want to read all files.


## Answer

Since `bash` 4.0 leading zero 
[is supported](http://wiki.bash-hackers.org/scripting/bashchanges#quoting_expansions_substitutions_and_related)
in `{0x..0y}` (zero-padded brace expansion). With it, you can do it like this:

    for i in {01..15}; do COMMAND "abc$i.txt"; done


---


## Question: [Bash `if` with arithmetic expression](https://stackoverflow.com/q/45238867/404556)

    # prints "here" when run in bash
    if [[ ((9 > 220)) ]]; then echo "here"; fi

I'm confused why the above if statement gets evaluated to true.  Wouldn't `((9 > 220))` evaluate to
false which would make the if statement false? The code below behaves as expected.  I'm confused why
using the double parentheses in a double brackets "if" isn't working above though.

    # doesn't print anything
    if ((9 > 220)); then echo "here"; fi


## Answer

There's a fundamental difference between those two
[compound commands](https://www.gnu.org/software/bash/manual/bash.html#Compound-Commands)
inside a conditional construct.

### `[[ expression ]]` compound command

In your first example you're using the `[[ expression ]]` command:

    if [[ ((9 > 220)) ]]; then echo "here"; fi

where **parenthesis are treated merely as grouping operators**, used to override the normal
precedence of other operators (like `!`, `&&`, `||`, `>`, `-gt`, `-e`, etc.). The `>` operator in
this case is a lexicographic greater than.

This is nicely described in `man bash`:

> `[[ expression ]]`

> Return  a  status of 0 or 1 depending on the evaluation of the conditional expression expression.
> Expressions are composed of the primaries described below under CONDITIONAL EXPRESSIONS.  Word
> splitting  and  pathname  expansion are  not  performed  on the words between the [[ and ]]; tilde
> expansion, parameter and variable expansion, arithmetic expansion, command substitution, process
> substitution, and quote removal are performed.  Conditional  operators such as `-f` must be
> unquoted to be recognized as primaries.

> When used with `[[`, the `<` and `>` operators **sort lexicographically** using the current locale.

So, to compare integers in the `[[` compound command, you can use the [conditional expression operators](https://www.gnu.org/software/bash/manual/bash.html#Bash-Conditional-Expressions),
the same one used by `test` and `[` commands. For example like this:

    if [[ 9 -gt 220 ]]; then echo "here"; fi

The result is the same like when the `-gt` operator is grouped with parenthesis:

    if [[ ((9 -gt 220)) ]]; then echo "here"; fi

Alternatively, you can use the [arithmetic expansion](https://www.gnu.org/software/bash/manual/bash.html#Arithmetic-Expansion-1)
and exploit the fact that the boolean results are represented as `"0"` or `"1"`:

    if [[ $((9 > 200)) == 1 ]]; then echo "here"; fi


### `(( expression ))` compound command

In your second example, you're using the `(( expression ))` command:

    if ((9 > 220)); then echo "here"; fi

where the `expression` is evaluated according to the rules of the
[shell arithmetic](https://www.gnu.org/software/bash/manual/bash.html#Shell-Arithmetic).
The `man bash` says:

> `((expression))`

> The  expression  is evaluated according to the rules described below under ARITHMETIC EVALUATION.
> If the value of the expression is non-zero, the return status is 0; otherwise the return status is
> 1.  This is exactly  equivalent to `let "expression"`.


---


## Question: [Round number to 10 in Bash](https://stackoverflow.com/q/45240943/404556)

I made a simple function where the result is e.g. 64.998472747. Is there a way this can be rounded
up to 60? If it is 65.998472747 to be rounded to 70? Is this possible in Bash?

    light=62.998472747
    printf "$.0f" "$light"

So my function only rounds to a whole number. In this case it is going to round it to 63.


## Answer

Since you asked explicitly for a **solution in `bash`**, here it is:

    round10() {
        echo $(( ((${1%.*}+5)/10)*10 ))
    }

For example:

    $ round10 64.998472747
    60
    $ round10 65.998472747
    70
    $ round10 3.14
    0
    $ round10 6.28
    10

We're using [Parameter Expansion](https://www.gnu.org/software/bash/manual/bash.html#Shell-Parameter-Expansion) to first strip suffix decimals after the dot, in combination with [Arithmetic Expansion](https://www.gnu.org/software/bash/manual/bash.html#Arithmetic-Expansion) to perform (now) integer arithmetic.


---


## Question: [Why does this string comparison if statement fail to work](https://stackoverflow.com/q/45265858/404556)

I expect the following code to print only one `x`, but it always gives two:

    #!/bin/sh
    read i
    if [[ $i!=1 ]];then
        echo x
    fi
    if [[ $i==1 ]];then
        echo x
    fi

Why is this happening?

## Answer

First of all, your script is invalid in POSIX `sh`, since **`[[` is undefined in `sh`**.

In `bash`, on the other hand, **you're missing spaces around comparison operators** `!=` and `==`.
That's why the expression inside `[[ ]]` is treated as a non-zero length string, which is truthy.
Hence, `echo` is printed twice.

As suggested so many times on StackOverflow, it's always good to run your shell scripts through
[shellcheck](http://www.shellcheck.net/) (available as command line tool also), which will help you
catch and explain many of such errors.
