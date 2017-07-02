# `sed` magic


## Question: [Replace all occurrences of a string on lines that contain a pattern](https://stackoverflow.com/questions/44599449/sed-replace-all-occurrences-of-a-string-in-the-second-last-line-of-a-file/)

I have a bash script that I am using to modify a sql file (test.sql).

The sql file is as follows:

    select count(*) from (
    select 
    I.ID,
    I.START_DATE, 
    I.END_DATE
    from MY_TABLE I with (nolock)
    where I.START_DATE >= '20170101' and I.START_DATE < '20170201'
    ) as cnt

I want to replace **all** the occurrences of the string `START_DATE` **but only** in the `WHERE`
clause of the sql file.


## Answer

Try it like this:

    sed -e '/where/ s/START_DATE/END_DATE/g' -i test.sql

Explanation:

- operate only on lines that contain ``where`` (we "address" only the lines that match the regex pattern ``where``)

- replace each occurrence of ``START_DATE`` with ``END_DATE`` - notice the "global" flag ``g`` at the end

- the ``-i`` flag tells sed to edit the file "in place" (no need to redirect output).


---


## Question: [Delete duplicate blocks in a file, each n lines long](https://stackoverflow.com/questions/44781734/delete-n-duplicate-lines-in-a-file/)

I have a large text file (14MB). I need to remove in a file text blocks, contains 7 duplicate lines.

Example file:

	Millionaire
	123456788763237476
	QUESTION|2402394827049882049
	Who is the greatest Goddess of the world?
	Sasha
	Kristina
	Sasha
	Katya
	Valeria
	AuthorOfQuestion
	Millionaire
	459385734954395394
	QUESTION|9845495845948594999
	Where Sasha live?
	Novgorod
	St. Petersburg
	Kazan
	Novgorod
	Chistopol
	Another author
	Millionaire
	778845225202502505
	QUESTION|984ACFBBADD8594999A
	Who is the greatest Goddess of the world?
	Sasha
	Kristina
	Sasha
	Katya
	Valeria

Expected output has all repeated blocks removed. In this example:

	Millionaire
	123456788763237476
	QUESTION|2402394827049882049
	Who is the greatest Goddess of the world?
	Sasha
	Kristina
	Sasha
	Katya
	Valeria
	AuthorOfQuestion
	Millionaire
	459385734954395394
	QUESTION|9845495845948594999
	Where Sasha live?
	Novgorod
	St. Petersburg
	Kazan
	Novgorod
	Chistopol
	Another author
	Millionaire


## Answer

Here's a simple solution to your problem (if you have access to GNU ``sed``, ``sort`` and ``uniq``):

    sed 's/^Millionaire/\x0&/' file | sort -z -k4 | uniq -z -f3 | tr -d '\000'

An explanation is in order:

- since *all your blocks* begin with the word/line ``Millionaire``, we can use that to split the file in (variably long) blocks by prepending a ``NUL`` character to each ``Millionaire``;
- then we sort those ``NUL``-separated blocks (using to the ``-z`` flag), but ignoring the first 3 fields (in this case lines: ``Millionaire``, ``\d+``, ``QUESTION|ID...``), using the ``-k``/``--key`` option with start position being the field ``4`` (in your case line 4), and the stop position being the end of the block;
- after sorting, we can filter-out the duplicates with ``uniq``, again using the ``NUL`` delimiter instead of newline (``-z``), and ignoring the first 3 fields (with ``-f``/``--skip-fields``);
- finally, we remove ``NUL`` delimiters with ``tr``.

In general, solution for removing duplicate blocks like this should work whenever there's a way to
split the file into blocks. Note that *block-equality* can be defined on a subset of fields (as we
did above).


---


## Question: [Search and Replace operation with reporting the files and lines that got changed](https://stackoverflow.com/q/44843457/404556)

I have a input file "test.txt" as below - 

    hostname=abc.com hostname=xyz.com
    db-host=abc.com db-host=xyz.com

In each line, the value before space is the old value which needs to be replaced by the new value after the space recursively in a folder named "test". I am able to do this using below shell script.

    #!/bin/bash

    IFS=$'\n' 
    for f in `cat test.txt`
    do
      OLD=$(echo $f| cut -d ' ' -f 1) 
      echo "Old = $OLD"
      NEW=$(echo $f| cut -d ' ' -f 2)
      echo "New = $NEW"
      find test -type f | xargs sed -i.bak "s/$OLD/$NEW/g"
    done

"sed" replaces the strings on the fly in 100s of files.

Is there a trick or an alternative way by which i can get a report of the files changed like absolute path of the file & the exact lines that got changed ?

PS - I understand that sed or stream editors doesn't support this functionality out of the box. I don't want to use versioning as it will be an overkill for this task. 


## Answer

Let's start with a simple rewrite of your script, to make it a little bit more robust at handling a wider range of replacement values, but also faster:

    #!/bin/bash
    
    # escape regexp and replacement strings for sed
    escapeRegex() { sed 's/[^^]/[&]/g; s/\^/\\^/g' <<<"$1"; }
    escapeSubst() { sed 's/[&/\]/\\&/g' <<<"$1"; }
    
    while read -r old new; do
        find test -type f -exec sed "/$(escapeRegex "$old")/$(escapeSubst "$new")/g" -i '{}' \;
    done <test.txt

So, we loop over pairs of whitespace-separated fields (`old`, `new`) in lines from `test.txt` and run a standard `sed` in-place replace on all files found with `find`.

Pretty similar to your script, but we [properly read lines](http://mywiki.wooledge.org/BashFAQ/001) from `test.txt` (no word splitting, pathname/variable expansion, etc.), we use Bash builtins whenever possible (no need to call external tools like `cat`, `cut`, `xargs`); and we [escape `sed` metacharacters](https://stackoverflow.com/questions/29613304/is-it-possible-to-escape-regex-metacharacters-reliably-with-sed) in `old`/`new` values for proper use as `sed`'s regexp and replacement expressions.

Now let's add [logging from sed](https://unix.stackexchange.com/questions/97297/how-to-report-sed-in-place-changes):

    #!/bin/bash
    
    # escape regexp and replacement strings for sed
    escapeRegex() { sed 's/[^^]/[&]/g; s/\^/\\^/g' <<<"$1"; }
    escapeSubst() { sed 's/[&/\]/\\&/g' <<<"$1"; }
    
    while read -r old new; do
        find test -type f -printf '\n[%p]\n' -exec sed "/$(escapeRegex "$old")/{
            h
            s//$(escapeSubst "$new")/g
            H
            x
            s/\n/ --> /
            w /dev/stdout
            x
        }" -i '{}' > >(tee -a change.log) \;
    done <test.txt

The `sed` script above changes each `old` to `new`, but it also writes `old --> new` line to `/dev/stdout` (Bash-specific), which we in turn append to `change.log` file. The `-printf` action in `find` outputs a "header" line with file name, for each file processed.

With this, your "change log" will look something like:

    [file1]
    hostname=abc.com --> hostname=xyz.com

    [file2]

    [file1]
    db-host=abc.com --> db-host=xyz.com

    [file2]
    db-host=abc.com --> db-host=xyz.com

Just for completeness, a quick walk-through the `sed` script. We act only on lines containing the `old` value. For each such line, we store it to hold space (`h`), change it to `new`, append that new value to the hold space (joined with newline, `H`) which now holds `old\nnew`. We swap hold with pattern space (`x`), so we can run `s` command that converts it to `old --> new`. After writing that to the `stdout` with `w`, we move the `new` back from hold to pattern space, so it gets written (in-place) to the file processed.
