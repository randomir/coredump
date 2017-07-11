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


---


## Question: [How to use `find` to search for all files matching pattern for name?](https://stackoverflow.com/q/44995142/404556)

For a list of files:

    ./color_a.txt
    ./color_b.txt
    ./color_c.txt
    ./color/color_d.txt
    ./color/blue.txt
    ./color/red.txt
    ./color/yellow.txt

I want to get all files that have a `color` in name, like this:

    ./color_a.txt
    ./color_b.txt
    ./color_c.txt
    ./color/color_d.txt

The command I use:

    find ./*color* -type f

is not working like that. How should I write it?


## Answer

What you probably want for filename filtering is a simple `-name <glob-pattern>` test:

    find -name '*color*' -type f

From `man find`:

     -name pattern
            Base  of  file  name  (the  path with the leading directories removed) matches shell
            pattern pattern.  Because the leading directories are removed, the file names
            considered for a match with -name will never include a  slash,  so `-name a/b' will 
            never match anything (you probably need to use -path instead).

Just as a side note, when you wrote:

    find ./*color* -type f

the shell expanded the (unquoted) glob pattern `./*color*`, and what was really executed (what `find` saw) was this:

    find ./color ./color_a.txt ./color_b.txt ./color_c.txt -type f

thus producing a list of files in all of those locations.


---


## Question: [find seems to be much slower with -print0 option](https://stackoverflow.com/q/45033988/404556)

I am trying to resize photos larger than specific dimensions for 100s of thousands of photos
collected by a system over past 10 years. I am using `find` and `imagemagick`.

I wrote this script to do it.

    #!/bin/bash
    
    find . -type f -iname '*JPG' -print0 | \
    while IFS= read -r -d '' image; do
        ...snip...
    done

The script works on a small amount of files but it takes a long time to start with lots of files. I
have tested on the command line `find . -type f -iname '*JPG' -print0` vs `find . -type f -iname '*JPG'`.
The later finds files within a few seconds but the former takes minutes before anything is
found? Unfortunately the `-print0` is required for dealing with filenames with special characters
(which are mainly spaces in my case). How can I get this script to be more efficient?


## Answer

I can not reproduce the behavior you're experiencing, but can think of two possible explanations.

**First**, you might be experiencing positive effects of page (disk) caching.

When you call `find` for the first time, it traverses files (metadata in inodes), actually reading
from the data media (HDD) via kernel `syscall`. But kernel (transparently to `find`, or other
applications) also stores that data in unused areas of memory, which acts as a cache. If this data
is read again later, it can be quickly read from this cache in memory. This is called
[page caching](https://en.wikipedia.org/wiki/Page_cache).

So, your second call to `find` (no matter what output separator is used) will be *a lot faster*,
assuming you are searching over the same files, with the same criteria.

**Second**, since `find`'s output might be buffered, if your files are in many different locations,
it might take some time before the actual first output to the `while` command. Also if the output is
line-buffered, that would explain why `-print0` variant takes longer to produce the first output
(since there are no lines at all).

You can try running `find` with unbuffered output, via `stdbuf` command:

    stdbuf -o0 find . -iname '*.jpg' -type f -print0 ...

One more thing, unrelated to this; to speed-up your `find` search, you might want to consider
calling it like this:

    find . -iname '*.jpg' -type f -print0

Here we put the `-iname` test before the `-type` test in order to avoid having to call `stat(2)` on
every file. Even better would be to remove the `-type` test all together, if possible.
