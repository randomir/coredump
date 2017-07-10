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

