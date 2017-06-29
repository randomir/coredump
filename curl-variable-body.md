# [How to call `curl` with request body data from a shell variable](https://stackoverflow.com/q/35312950/404556)

## Question

When I try to use a shell variable in `curl` request data, variable value is not used:

    curl -s -POST --header 'Content-Type: application/json' 'http://www.dummy.com/projectname/page_relevance' -d '{"query": "$query_string", "results": [{"abstract": "abs_string", "title": "title_string"}, "mode": "value", "cache": true, "source": "value"}'

### Tags: `bash`, `shell`, `curl`, `json`

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