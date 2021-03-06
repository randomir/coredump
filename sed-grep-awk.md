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


---


## Question: [Passing variables to `sed` command](https://stackoverflow.com/q/45039931/404556)

I am trying to run the following in a script. It basically appends TXT_NEW after TXT with the define
variables. Running the script wrapped in a small bash script throws an error:

    sed: -e expression #1, char 1: unknown command: `''

The script:

    #!/bin/bash

    user="Alpine"
    new_user="Volverine"
    file_name=file.txt

    TXT="This Text is by $user"
    TXT_NEW="This is owned by $new_dev"
    
    sed -i "'/$TXT/a ${TXT_NEW}'" $file

Appreciate if someone can have a look and provide some pointers.


## Answer

First of all, you have a problem with double quoting. Single quotes inside outer double quotes are
making `sed` fail, since it's reading the first character `'` as an invalid command.

So, just remove those and the example you provided should work:

    #!/bin/bash
    user="Alpine"
    new_user="Volverine"
    file_name="file.txt"
    
    TXT="This Text is by $user"
    TXT_NEW="This is owned by $new_user"
    
    sed -i "/$TXT/a ${TXT_NEW}" "$file_name"

----------

However, since your question is about **passing a variable text** to `sed` append command, you might
consider using the *append from file command*, `r <filename>`, like this:

    sed -i "/$TXT/r /dev/stdin" $file_name <<<"$TXT_NEW"

The `r` command is similar to `a` (append text), but it reads the text to be appended from the file
specified. If we say to read from `/dev/stdin` (should work in `bash`), we can provide the text via
[here-string](http://tldp.org/LDP/abs/html/x17837.html).

Another thing you should do **to generalize** this is to handle `sed` regex metacharacters in your
"search string" variable `$TXT`, by escaping `\`, `/` and `&`. Perhaps via a helper function
`escape()`:

    escape() {
        sed -e 's/[\/&]/\\&/g' <<<"$1"
    }
    sed -i "/$(escape "$TXT")/r /dev/stdin" $file_name <<<"$TXT_NEW"


---


## Question: [Simple `awk` on `ifconfig`](https://stackoverflow.com/q/45101155/404556)

I'm trying to save my internet IP address to a variable in the shell.

I tried

    ip=`host 'localhost' | awk '{print $4}'`

But that returns

    127.0.0.1

How to save internet IP address to a variable in the shell?


## Answer

To get IP addresses of all you local interfaces, try using `ifconfig`, for example like this:

    $ ifconfig | awk '/inet / { print $2 }'
    127.0.0.1
    10.8.0.34
    192.168.1.2

Here you can see I have 3 addresses (the loopback adapter, ethernet and wifi). If you know your
local network address has `192.*` form, you can get only that one with:

    $ ifconfig | awk '/inet *192/ { print $2 }'
    192.168.1.2

And to store it into a variable:

    ip=$(ifconfig | awk '/inet *192/ { print $2 }')

Or, to store all addresses in a `bash` array:

    $ ips=($(ifconfig | awk '/inet / { print $2 }'))
    $ printf "ip: %s\n" "${ips[@]}"
    ip: 127.0.0.1
    ip: 10.8.0.34
    ip: 192.168.1.2

To get IPv6 addresses, look for `inet6` instead of `inet`.


---


## Question: [Proper use of capture groups in SED command](https://stackoverflow.com/q/45235233/404556)

I need to convert a string `"1,234"` to `1234`. this string is just a part of a bigger line. There
are thousands of such lines in the file.

I have written a sed command which is not working as I expect it to (command omitted).


## Answer

I would suggest you use [POSIX Extended Regular Expressions](http://pubs.opengroup.org/onlinepubs/009695399/basedefs/xbd_chap09.html) (ERE),
where you don't have to escape parentheses and the repetition operator. To enable ERE in `sed`, you can
use the `-E` switch (or `-r` in GNU `sed`). Your expression will then look like this:

    $ echo '"1,234"' | sed -E 's/"([0-9]+),([0-9]+)"/\1\2/g'
    1234

For completeness, your original BRE expression will function properly if you escape the `+`:

    echo \"1,234\" | sed 's/\("\)\([0-9]\+\)\(,\)\([0-9]\+\)\("\)/\2\4/g'
    1234


---


## Question: [Filter table in text file with IDs from second file](https://stackoverflow.com/q/45242251/404556)

Okay, so I'm pretty new to this sort of thing, so please bear with me. 

I have two files:

`search_results_accesions.txt` is a list of identifiers, one per line. It looks like this (note that
not all of the identifiers will start with "NP_"):

    $ more search_results_accessions.txt 
    NP_000020.1
    NP_000026.2
    NP_000027.2
    NP_000029.2
    NP_000034.1
    NP_000042.3
    ...

`prot.accession2taxid.txt` is a file which lists each of the identifiers (and many, many more which
are not in my list), and gives the corresponding `taxid`. Here is what that looks like (the third
column contains the `taxid`s):

    $ more prot.accession2taxid
    accession       accession.version       taxid   gi
    APZ74649        APZ74649.1      36984   1137646701
    AQT41667        AQT41667.1      1686310 1150388099
    WP_080502060    WP_080502060.1  95486   1169627919
    ASF53620        ASF53620.1      492670  1211447116
    ASF53621        ASF53621.1      492670  1211447117
    ASF53622        ASF53622.1      492670  1211447118
    ASF53623        ASF53623.1      492670  1211447119
    ...

Fields are tab-separated.

I need to get the `taxid` for each `accession` in my `searchresults_accessions.txt` file. I'm on a
Unix system and would prefer to use command line or Python, if at all possible.


## Answer

Here's a solution with `awk` (you did say *command line or Python*):

    awk 'NR==FNR {ids[$1]=1} NR>FNR && ($1 in ids) {print $1 "\t" $3}' accessions taxids

Explanation:

- we split input lines using the default separators (space or tab)
- first we read the `accessions` file, then `taxids`
- for the lines in the first file (while total number of records read is equal to the number of records from current file, `NR==FNR`), we add values from the first column to the associative map `ids`
- for the lines in the second file, we print the first and the third field, separated by a tab character, but only if the value in the first field is present in our map of accession ids


---


## Question: [unix awk subtracting integer field from month of date field](https://stackoverflow.com/q/45313517/404556)

I have a file with around 10M records. Here is my sample src file:

    0000000566 2017/01/01 0
    0000000055 2017/01/01 0
    0000000109 2017/01/01 1
    0000000940 2017/01/01 0
    0000000566 2017/01/01 1
    0000000055 2017/01/01 1
    0000000109 2017/01/01 2

I essentially need to subtract the last integer value off of the month in the date and print the new value without the integer, thus:

    0000000566 2017/01/01
    0000000055 2017/01/01
    0000000109 2016/12/01
    0000000940 2017/01/01
    0000000566 2016/12/01
    0000000055 2016/12/01
    0000000109 2016/11/01


## Answer

If your `awk` supports [time functions](https://www.gnu.org/software/gawk/manual/html_node/Time-Functions.html#Time-Functions)
`mktime` and `strftime` (which are a GNU extension), you can simply do it like this:

    awk -F'[ /]' '{print $1 " " strftime("%Y/%m/%d", mktime($2" "($3-$5)" "$4" 0 0 0"))}' file

First we convert the date into a Unix timestamp. `mktime` accepts dates only in `"YYYY MM DD HH MM
SS"` format, that's why we need to construct it manually. But it does the normalization
automatically, and it will happily convert `"2017 -1 1 0 0 0"` to the same timestamp as `"2016 11 1
0 0 0"`.

After that we just need to convert the timestamp to "y/m/d" format and print it.

---

Or, you could do the date arithmetic "by hand" in the simple case that doesn't require date
normalization -- if the day of the month is always `<= 28`. (For days greater that `28`, like `31`,
you would also need to add clipping/clamping or overflowing to the script below, but then you would
have to take care of leap years, etc.)

    #!/usr/bin/awk -f
    
    BEGIN {
        FS = "[ /]";
    }

    {
        mm = $2 * 12 + ($3 - 1) - $5;
        y = int(mm / 12);
        m = mm % 12 + 1;
        d = $4;
        printf("%s %04d/%02d/%02d\n", $1, y, m, d);
    }

So, the idea is simple. We split the line on spaces and slashes, so we can convert year/month into a
total number of months (`12 * y + m`). Then we subtract the month from the last column and convert
the total number of months back to year/month via `divmod` operation.

Output:

    $ ./script.awk file
    0000000566 2017/01/01
    0000000055 2017/01/01
    0000000109 2016/12/01
    0000000940 2017/01/01
    0000000566 2016/12/01
    0000000055 2016/12/01
    0000000109 2016/11/01


---


## Question: [sed: How to extract only first occurence of value](https://stackoverflow.com/q/45315434/404556)

I have file `settings.inc.php` with the following content:

    <?php
    define('_DB_SERVER_', 'mariadb');
    define('_DB_NAME_', 'organic');
    define('_DB_USER_', 'prestashop');
    define('_DB_PASSWD_', 'prestashop');

I want to extract these values to `bash`, so I managed to create the following command:

    sed -rn 's/^.*_DB_NAME_'\'', '\''(\w+)'\''\);/\1/p' settings.inc.php

This will return `organic`, just as it should, but I would like to improve it further. Let's say we would have this kind of file:

    <?php
    define('_DB_SERVER_', 'mariadb');
    define('_DB_NAME_', 'organic1');
    define('_DB_NAME_', 'organic2');
    define('_DB_USER_', 'prestashop');
    define('_DB_PASSWD_', 'prestashop');

Using above command on this file we would get:

    organic1
    organic2

The thing is: I want for this command to always return only one value, so let's say the first one. Can I achieve that without piping result into second command?


## Answer

If you convert your substitute command followed by print to a command block operating only on lines addressed by pattern (containing) `_DB_NAME_`, you can quit after the first match/print:

    $ sed -rn "/_DB_NAME_/ { s/.*'(\w+)'\);$/\1/p;q }" settings.inc.php
    organic1

Note the `q` command after `p`.

Also, your `sed` script can be simplified by using outer double quotes and anchoring on the end.


---


## Question: [Average column if value in other column matches and print as additional column](https://stackoverflow.com/q/45490144/404556)

I have a file like this:

    Score      1      24      HG      1  
    Score      2      26      HG      2  
    Score      5      56      RP      0.5  
    Score      7      82      RP      1  
    Score      12     97      GM      5  
    Score      32     104     LS      3  

I would like to average column 5 if column 4 are identical and print the average as column 6 so that
it looks like this:

    Score      1      24      HG      1      1.5
    Score      2      26      HG      2      1.5  
    Score      5      56      RP      0.5    0.75  
    Score      7      82      RP      1      0.75  
    Score      12     97      GM      5      5  
    Score      32     104     LS      3      3  

I have tried a couple of solutions I found on here.
e.g.

    awk '{ total[$4] += $5; ++n[$4] } END { for(i in total) print i, total[i] / n[i] }'

but they all end up with this:

    HG      1.5
    RP      0.75  
    GM      5  
    LS      3

Which is undesirable as I lose a lot of information.


## Answer

You can iterate through your table twice: calculate the averages (as you already) do on the first
iteration, and then print them out on the second iteration:

    awk 'NR==FNR { total[$4] += $5; ++n[$4] } NR>FNR { print $0, total[$4] / n[$4] }' file file

Notice the `file` twice at the end. While going through the "first" file, `NR==FNR`, and we sum the
appropriate values, keeping them in memory (variables `total` and `n`). During "second" file
traversal, `NR>FNR`, and we print out all the original data + averages:

    Score      1      24      HG      1     1.5
    Score      2      26      HG      2     1.5
    Score      5      56      RP      0.5   0.75
    Score      7      82      RP      1     0.75
    Score      12     97      GM      5     5
    Score      32     104     LS      3     3


---


## Question: [sed to replace multiple pairs of identical pattern on same line](https://stackoverflow.com/q/45490296/404556)

I want to replace the file `test.txt` containing

    some text $\alpha$ some text $\alpha$ some text
    some text $\beta$ some text
    some text $\gamma$.
    some text $\delta$
    $\epsilon$ some text $\epsilon$
    $\mu$

    some text `$a$` some text `$a$` some text
    some text `$b$` some text
    some text `$c$`.
    some text `$d$`
    `$e$` some text `$e$`
    `$f$`

	$$\Alpha$$
	    $$\Beta$$

	`$$A$$`
	    `$$B$$`

by

    some text '$\alpha$' some text `$\alpha$` some text
    some text '$\beta$' some text
    some text '$\gamma$'.
    some text '$\delta$'
    `$\epsilon$` some text `$\epsilon$`
    `$\mu$`

    some text `$a$` some text
    some text `$b$` some text
    some text `$c$`. 
    some text `$d$`
    `$e$` some text `$e$`
    `$m$`

	`$$\Alpha$$`
	    `$$\Beta$$`

	`$$A$$`
	    `$$B$$`

In short, I want to do the replacements

    $..$ --> `$..$`

and

    $$..$$ --> `$$..$$`

in one set of of `sed` commands. But if the set of commands is reapplied on the file is should not add extra (`) symbols.

So far I have tried the following set: 

        sed -e 's/^\(\$\$.*\$\$\)/`\1`/g' -i test.txt
        sed -e 's/[^`]\(\$\$[^`].*[^\$]\$\$\)[^`]/`\1`/g' -i test.txt
        sed -e 's/^\(\$.[^\$]*\$\)/`\1`/g' -i test.txt
        sed -e 's/[^`$$]\(\$[^`].[^\$]*\$\)[^`$$]/ `\1` /g' -i test.txt

but this doesnt work fully... 


## Answer

You should be able to get off with a single `sed` expression:

    # ERE
    sed -E 's/([^`$]|^)(\${1,2}[^`$]+\${1,2})([^`$]|$)/\1`\2`\3/g' test.txt

    # or, if you prefer BRE (but only with GNU sed)
    sed 's/\([^`$]\|^\)\(\$\{1,2\}[^`$]\+\$\{1,2\}\)\([^`$]\|$\)/\1`\2`\3/g' test.txt

gives you:

    some text `$\alpha$` some text `$\alpha$` some text
    some text `$\beta$` some text
    some text `$\gamma$`.
    some text `$\delta$`
    `$\epsilon$` some text `$\epsilon$`
    `$\mu$`
    
    some text `$a$` some text `$a$` some text
    some text `$b$` some text
    some text `$c$`.
    some text `$d$`
    `$e$` some text `$e$`
    `$f$`
    
    `$$\Alpha$$`
        `$$\Beta$$`
    
    `$$A$$`
        `$$B$$`

We match three groups:

- prefix: a single non-backtick or non-dollar or line beginning
- the meat: one or two `$`, followed by content, followed by one/two `$`
- suffix: same as prefix, except we match line ending

then print them out, quoting with backticks only the middle group. We need to anchor on those prefix
and suffix to avoid that double quoting you're experiencing.

**Note** that the POSIX BRE (basic regular expression) form above uses several GNU extensions,
namely: anchoring on line beginning and line ending used in the middle of expression (and not as the
first/last character in pattern), alternation (`\|`) and one-or-more duplication operator (`\+`). If
you need this expression to work in POSIX BRE, you'll need to break it into several (i.e. 3)
subexpressions and use `\{1,\}` instead of `\+`.

But also note that the POSIX ERE (extended regular expressions) form given should work in all modern
`sed` environments, on both GNU and BSD systems.

