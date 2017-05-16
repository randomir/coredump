# [Change text copied to the clipboard using `xclip`](https://stackoverflow.com/q/44009724/404556)

## Question

I am trying to convert text copied to the clipboard from something like this:

    +50.8863-114.0157/

to something like this:

    geo:50.8863,-114.0157,0

### Tags: `xclip`, `sed`, `bash`

## Answer

This oneliner is basically all you need:

    xclip -o | sed -rne's/\+?(-?[[:digit:].]+)\+?(-?[[:digit:].]+)\//geo:\1,\2,0/p' | xclip -i

Explanation:

- `xclip -o` outputs the X selection to the standard output
- `sed <regex>` parses the format you gave (ignoring leading `+`'es) and prints the replacement text
  - `-r` switch instructs the `sed` to interpret regular expressions as *Extended Regular Expressions* (ERE) (quick intro [here][1]),
  - `-n` suppresses the output of (unmatched/unwanted) input -- so we have to explicitly print with the `p` command (the last letter in sed script)
  - `-e script` defines the sed script:
     - `s/regexp/replacement/` will match `regexp` in each line of input (only the first occurrence) and replace it with `replacement` (which can include input groups, like `\1`). The `p` in the sed script actually prints the replacement text.
     - regexp (in short) is made up of two identical consecutive subpatterns: `<optional +>(<optional -><one or more digits/dot>)`. Parentheses define a group which we use in the replacement.
- `xclip -i` sets X selection from stdin (sed's output)


  [1]: https://en.wikibooks.org/wiki/Regular_Expressions/POSIX-Extended_Regular_Expressions
