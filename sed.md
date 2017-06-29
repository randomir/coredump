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

