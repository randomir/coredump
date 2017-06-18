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

---

## Question: [How to exclude subfolders containing "[xxx] - (yy) - [xxx]"](https://stackoverflow.com/questions/44619982/how-to-exclude-subfolders-containing-xxx-yy-xxx/)

I have folders organized like this:

    Folder1
    [xxx] - ( aa) - [xxx]
     1.zip
     2.zip
    Folder2
    [xxx] - ( bb ) - [xxx]
     11.zip
     22.zip

I need to:

- go into every Folderx unzip the files it contains 
- log if the extracted file was not successfully extracted 
- move the extracted file to another folder
- log if the extracted file was not successfully moved

I came up with that at the begining:

    for D in `find . -type d`
    do
    	cd $D
    	echo "$(pwd)"
    	for z in *.zip; do
    		unzip -u $z
    		echo "File: " $z; 
    		if [ $? -eq 0 ]
    			then
    				echo "Successfully created file"
    			else
    				echo "Could not create file" >&2
    		fi; 
    	done
    done

My main problem, I think is that I haven't found a way of avoiding splitting the path at every space.


## Answer

Since your directories contain whitespaces, word splitting is performed on them (in each line
``find`` returns). See a nice explanation of this very scenario (``for`` loop over list of files) on
the [bash pitfalls][1] page.

One of the ways to write your outermost ``for`` loop is like this:

    find . -type d | while read D; do
        cd "$D"
        ...
    done

Another approach, for simpler actions, is to just use ``find`` with ``-execdir`` action, like this:

    find . -name '*.zip' -execdir unzip -u '{}' \;

Note we use ``execdir`` action which executes the command *in the file directory*, and not ``exec``
which executes the command in the current directory, because ``unzip`` extracts in the current dir,
unless given the destination dir thru ``-d`` option.

Or in bash 4+, you can use the glob recursive expansion:

    shopt -s globstar
    for z in ./**/*.zip; do
        dir="$(dirname "$z")"
        cd "$dir"
        unzip -u "$z" && echo "OK" || echo "Failed to extract $z"
    done

Also, don't forget to (almost) **always quote your variables**.

  [1]: http://mywiki.wooledge.org/BashPitfalls#for_i_in_.24.28ls_.2A.mp3.29

