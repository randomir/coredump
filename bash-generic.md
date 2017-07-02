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

