# Bash (or POSIX shell) file ops

## Question: [How to rename file name contains backslash in bash?](https://stackoverflow.com/questions/44490413/how-to-rename-file-name-contains-backslash-in-bash/)

I got a tar file, after extracting, there are many files naming like

    a
    b\c
    d\e\f
    g\h

I want to correct their name into files in sub-directories like

    a
    b/c
    d/e/f
    g/h

I face a problem when a variable contains backslash, it will change the original file name. I want to write a script to rename them.


## Answer

Renaming a file with backslashes is simple: ``mv 'a\b' 'newname'`` (just quote it), but you'll need more than that.

You need to:

  - find all files with a backslash (e.g. ``a\b\c``)
  - split path from filename (e.g. ``a\b`` from ``c``)
  - create a complete path (e.g. ``a/b``, dir ``b`` under dir ``a``)
  - move the old file under a new name, under a created path (e.g. rename ``a\b\c`` to file named ``c`` in dir ``a/b``)

Something like this:

    #!/bin/bash
    find . -name '*\\*' | while read f; do
        base="${f%\\*}"
        file="${f##*\\}"
        path="${base//\\//}"
        mkdir -p "$path"
        mv "$f" "$path/$file"
    done
