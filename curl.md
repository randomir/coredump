# [How to call `curl` with request body data from a shell variable](https://stackoverflow.com/q/35312950/404556)


## Question

When I try to use a shell variable in `curl` request data, variable value is not used:

    curl -s -POST --header 'Content-Type: application/json' 'http://www.dummy.com/projectname/page_relevance' -d '{"query": "$query_string", "results": [{"abstract": "abs_string", "title": "title_string"}, "mode": "value", "cache": true, "source": "value"}'


## Answer

The single quotes `'` (you're using in the `-d` argument) preserve the literal value of each
character, including the `$` (see [this SO answer][1]), and that's why your variable `query_string`
is not being expanded.

Try this:

    ~$ query_string="my query"

    ~$ echo '$query_string'
    $query_string

    ~$ echo "$query_string"
    my query
    
So, you need to use double quotes `"` if you wish your variables to expand to its values.

However, in order to nest double quotes (inside other double quotes), as in you JSON data, you must either:

 1. **escape** the inner quotes, like this:

        ~$ echo "{\"query\": \"$query_string\"}"
        {"query": "my query"}

    but that gets very ugly, very soon; or

 2. **concatenate** strings under alternating single and double quotes, like this:

        ~$ echo '{"query": "'"$query_string"'"}"'
        {"query": "my query"}"
   
    which may be more readable for shorter strings; or

 3. use a **here-document**:

        ~$ read query <<-END
        {"query": "$query_string"}
        END

        ~$ echo "$query"
        {"query": "my query"}

    [Here-documents][2] are particularly convenient for longer documents in which you wish for parameter/variable expansion, command substitution, arithmetic expansion, etc.

In summary, after defining your JSON query with one of the above ways (perhaps via a here-document), you can write your `curl` command like this:

    curl -s -X POST -H 'Content-Type: application/json' 'http://www.dummy.com/projectname/page_relevance' -d "$query"

  [1]: https://stackoverflow.com/questions/6697753/difference-between-single-and-double-quotes-in-bash
  [2]: http://tldp.org/LDP/abs/html/here-docs.html


---


## Question: [Assign variable to curl get request](https://stackoverflow.com/q/45444497/404556)

    #!/bin/bash
    while IFS='' read -r line || [[ -n "$line" ]]; do
        echo "Text read from file: $line"
        curl 'https://shoesworkshop.net/libraries/ajax/ajax.invoice.php?act=viewallinvoice&invoiceid="${line}"&sEcho=1&iColumns=8&iDisplayStart=0&iDisplayLength=20&bRegex=false&bRegex_0=false&bSearchable_0=true&bRegex_1=false&bSearchable_1=true&bRegex_2=false&bSearchable_2=true&bRegex_3=false&bSearchable_3=true&bRegex_4=false&bSearchable_4=true&bRegex_5=false&bSearchable_5=true&bRegex_6=false&bSearchable_6=true&bRegex_7=false&bSearchable_7=true&iSortCol_0=0&sSortDir_0=asc&iSortingCols=1&bSortable_0=true&bSortable_1=true&bSortable_2=true&bSortable_3=true&bSortable_4=true&bSortable_5=true&bSortable_6=true&bSortable_7=true' -H 'Host: shoesworkshop.net'| sed  's/^[^[[]*:/:/'
    done < "$1"

inside $line there is a value like this

    AAAAA
    SSSSS
    DDDDD

and I want to pass $line into curl command can someone help me how? I tried "'${line}'" and
'${line}' and it still not working. I want to make a repeat call using curl get request from the url
using variable from $line.


## Answer

For simple URLs, one way is to just use double quotes for the complete URL, including your variable
expansion, `${line}`, like this:

    curl "https://shoe...&invoiceid=${line}&sEcho=1&iCo...table_7=true"

(Under single quotes, your shell variable `line` is not expanded.)

If your URL contains shell-special characters like `$`, it's best to combine both single and double
quotes (and concatenate several strings, like explained [here](https://stackoverflow.com/a/44035918/404556)). For Example:

    curl 'https://shoe...&invoiceid='"$line"'&sEcho=1&iCo...table_7=true'
    #     ^------ fixed part ------^  ^var^  ^------- fixed part ------^

However, if your variable contains **characters that have to be URL-encoded** (like space, `&`, `?`,
etc.) it's best to **let `curl` handle that with `--data-urlencode` option**. When called with this
option, `curl` will default to `POST` method, but you can override this with `-G`, in which case
your parameters will be appended to URL query. For example:

    line="1&2?3 4"
    curl "http://httpbin.org/get?x=1&y=2" --data-urlencode z="$line" -G

produces the right URL:

    http://httpbin.org/get?x=1&y=2&z=1%262%3F3%204

Your script, fixed:

    #!/bin/bash
    while IFS='' read -r line || [[ -n "$line" ]]; do
        echo "Text read from file: $line"
        curl --data-urlencode invoiceid="$line" -G 'https://shoesworkshop.net/libraries/ajax/ajax.invoice.php?act=viewallinvoice&sEcho=1&iColumns=8&iDisplayStart=0&iDisplayLength=20&bRegex=false&bRegex_0=false&bSearchable_0=true&bRegex_1=false&bSearchable_1=true&bRegex_2=false&bSearchable_2=true&bRegex_3=false&bSearchable_3=true&bRegex_4=false&bSearchable_4=true&bRegex_5=false&bSearchable_5=true&bRegex_6=false&bSearchable_6=true&bRegex_7=false&bSearchable_7=true&iSortCol_0=0&sSortDir_0=asc&iSortingCols=1&bSortable_0=true&bSortable_1=true&bSortable_2=true&bSortable_3=true&bSortable_4=true&bSortable_5=true&bSortable_6=true&bSortable_7=true' -H 'Host: shoesworkshop.net' | sed 's/^[^[[]*:/:/'
    done < "$1"
