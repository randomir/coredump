# `sed`/`grep`/`awk` magic


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


---


## Question: [How do I insert text to the 1st line of a file using sed?](https://stackoverflow.com/q/44894652/404556)

Hi I'm trying to add text to the 1st line of a file using sed
so far I've tried

    #!/bin/bash
    touch test
    sed -i -e '1i/etc/example/live/example.com/fullchain.pem;\' test

And this doesn't work. Also tried

    #!/bin/bash
    touch test
    sed -i "1i ssl_certificate /etc/example/live/example.com/fullchain.pem;" test

this doesn't seem to work either
oddly when I try 

    #!/bin/bash
    touch test
    echo "ssl_certificate /etc/example/live/example.com/fullchain.pem;" > test

I get the 1st line of text to appear when i use `cat test`
but as soon as i type `sed -i "2i ssl_certificate_key /etc/example/live/example.com/privkey.pem;"`
I can't see the information that I should do on line 2 this being `ssl_certificate_key /etc/example/live/example.com/privkey.pem`;

So my question to summarise:

 1. Can text be inserted into the 1st line of a newly created file using sed?
 2. If yes whats the best way of inserting text after the 1st line of text?


## Answer

Suppose you have a `file` like this:

    one
    two

Then to append to the first line:

    $ sed '1 s_$_/etc/example/live/example.com/fullchain.pem;_' file
    one/etc/example/live/example.com/fullchain.pem;
    two

To insert before the first line:

    $ sed '1 i /etc/example/live/example.com/fullchain.pem;' file
    /etc/example/live/example.com/fullchain.pem;
    one
    two

Or, to append after the first line:

    $ sed '1 a /etc/example/live/example.com/fullchain.pem;' file
    one
    /etc/example/live/example.com/fullchain.pem;
    two

Note the number `1` in those `sed` expressions - that's called [**the address**](https://www.gnu.org/software/sed/manual/html_node/Addresses-overview.html#Addresses-overview) in `sed` terminology. It tells you on which **line** [the command](https://www.gnu.org/software/sed/manual/html_node/Other-Commands.html#Other-Commands) that follows is to operate.

If your file doesn't contain the line you're addressing, the `sed` command won't get executed. That's why you can't insert/append on line 1, if your file is empty.

Instead of using *stream editor*, to append (to empty files), just use a shell redirection `>>`:

    echo "content" >> file


---


## Question: [Linux bash regex global flag](https://stackoverflow.com/q/44970561/404556)

I want to match string with regex and echo matches. 

    if [[ "${PLUGS}"  =~  \"[a-zA-Z0-9.-]+ ]]; then
        echo ${BASH_REMATCH[*]}
    else
        echo "nothing"
    fi

But i don't know how to add global flag. If i write regex like ` /\"[a-zA-Z0-9.-]+/g` i get only first match. What i am doing wrong?

Samples data is mysql return of wordpress active plugins `SELECT * FROM wp_options WHERE option_name = 'active_plugins';`

    a:31:{i:0;s:13:"AddMySite.php";i:1;s:19:"akismet/akismet.php";i:2;s:23:"all_in_one_seo_pack.php";i:3;s:16:"authenticate.php";i:4;s:28:"breadcrumb-navigation-xt.php";i:5;s:18:"codeautoescape.php";i:6;s:37:"contact-coldform/contact_coldform.php";i:7;s:32:"custom-query-string-reloaded.php";i:8;s:30:"customizable-post-listings.php";i:9;s:33:"dd-sitemap-gen/dd-sitemap-gen.php";i:10;s:20:"download-counter.php";i:11;s:13:"feedcount.php";i:12;s:13:"full_feed.php";i:13;s:15:"get-weather.php";i:14;s:36:"google-sitemap-generator/sitemap.php";i:15;s:13:"gravatars.php";i:16;s:19:"kill-admin-nags.php";i:17;s:18:"landingsites13.php";i:18;s:30:"nofollow-free/nofollowfree.php";i:19;s:17:"ol_feedburner.php";i:20;s:16:"plugins-used.php";i:21;s:22:"popularity-contest.php";i:22;s:39:"search-everything/search_everything.php";i:23;s:27:"simple-tags/simple-tags.php";i:24;s:26:"simple_recent_comments.php";i:25;s:18:"simple_twitter.php";i:26;s:25:"subscribe-to-comments.php";i:27;s:24:"the-excerpt-reloaded.php";i:28;s:18:"theme-switcher.php";i:29;s:9:"top10.php";i:30;s:16:"wp-db-backup.php";}


## Answer

There is no global flag for `bash` regex, `BASH_REMATCH` array contains matches of parenthesized
subexpressions. So, you would have to rework your regex substantially.

But, if all you want to do is extract all matches and echo them, consider using `grep` with
`-o`/`--only-matching` flag:

    $ grep -oE '("[a-zA-Z0-9.-]+")' <<<"${PLUGS}"
    "AddMySite.php"
    "authenticate.php"
    "breadcrumb-navigation-xt.php"
    "codeautoescape.php"
    "custom-query-string-reloaded.php"
    "customizable-post-listings.php"
    "download-counter.php"
    "feedcount.php"
    "get-weather.php"
    "gravatars.php"
    "kill-admin-nags.php"
    "landingsites13.php"
    "plugins-used.php"
    "popularity-contest.php"
    "subscribe-to-comments.php"
    "the-excerpt-reloaded.php"
    "theme-switcher.php"
    "top10.php"
    "wp-db-backup.php"

We also need to use `-E`/`--extended-regexp`, since your regex is POSIX ERE.

Or, to also handle the non-matching case (like in your question):

    if ! grep -oEq '"([a-zA-Z0-9.-]+)"' <<<"${PLUGS}"; then
        echo "nothing"
    fi

Note we also use `-q`/`--quiet` to prevent matches output.


---


## Question [Track the last word of each line starting with pattern](https://stackoverflow.com/q/45011972/404556)

I have a `*.dat` file that incrementally grows for several hours. I want to monitor a certain value
in time (the last word in line that contains a pattern "15 RT") so that I can compare them, watch
its trend and so on.

    ...snip... [convoluted attempt in bash]


## Answer

Yes, this should do it:

    tail -f growing.dat | awk '/15 RT/ {print $NF}'

`tail -f` is very efficient, as it listens for file modify event and only outputs new lines when
added (no need to loop and constantly check if file was modified). `awk` script will simply output
the last field for each line that contains `15 RT`.

**Edit**. Additionally, if you wish to store that output to a file, and monitor the values in
terminal, you can use `tee`:

    tail -f growing.dat | awk '/15 RT/ {print $NF}' | tee values.log

Since `awk` is buffering output, to see the values in real-time, you can flush the output after each
update:

    tail -f growing.dat | awk '/15 RT/ {print $NF; fflush()}' | tee values.log

**Edit 2**. If the file doesn't exist initially, you should use `tail -F`:

    tail -F growing.dat | awk '/15 RT/ {print $NF}'

that way, `tail` will keep retrying to open file if it is inaccessible, it looks like this (message
is printed to `stderr`):

    tail: cannot open 'growing.dat' for reading: No such file or directory
    tail: 'growing.dat' has appeared;  following new file
    -5.1583E+04

