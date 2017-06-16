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

