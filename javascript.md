# Javascript


## Question: [evaluating "()" in node.js causes wierd behaviour, but browser says syntax error (bug in node.js?)](https://stackoverflow.com/q/44980706/404556)

When I evaluate the expression "()" in node.js (without the quotes), it corrupts the environment.
The global object is trashed and nothing works. When I evaluate the same express in the browser
console it reports a syntax error. From my research it appears the browser is correct. Is this a bug
in the node.js expression parser? Can anyone explain what is happening under the hood?


## Answer

Imagine you're writing your own JavaScript REPL. You know JavaScript's grammar in and out, and you
want to abort processing of an invalid input as soon as possible (but NOT before that). So, you
decide to read character by character, build token by token and abort as soon as you encounter the
impossible token (for a current state, state defined by input processed thus far).

So, you know if users enters `(}`, it's pretty safe to abort - there's no way something meaningful
is going to come out of that!

Also, if user enters `(2`, you don't want to abort yet - there are a plenty of valid characters to
follow, like `)`, `*`, `,`, etc.

You also know what to expect after `()`. If it isn't `=>...` (lambda expression), than it's invalid
token.

Now back to reality. REPL implementations in Node.js, Chrome, Firefox, etc. differ because they can.
Some are "smarter" than other, and will abort sooner than others, but that's really up to them.

Node.js chose to have "less smart" REPL, as it won't abort reading your input, as soon as it can. In
fact, if you entered something that can never form a valid expression, it will leave you hanging,
reading junk until you explicitly abort with `Ctrl+C`.

Chrome's console is a little bit smarter. When you type `()`, you'll get:

    > ()
    Uncaught SyntaxError: Unexpected token )

but, when you start typing a lambda `()=>`, it will allow you to finish it in the next line:

    > ()=>
    2

As a comfort note, you'll get the same error in Node.js REPL if you indicate you're done with
entering of your expression, like this:

    > eval("()")
    SyntaxError: Unexpected token )


---


## Question [Robust selector to find all elements of class](https://stackoverflow.com/q/44981644/404556)

How can I get all elements who have a class that begins with `vd-`? My code below works for most cases except for the following:

    class="foo vd-bar"

It works for these cases:

    class="vd-bar foo"

Here is my selector, will it work if the HTML markup uses single quotation marks? For eg; `class='vd-foo'`?

	$('[class^="vd-"]', myElement).each(function(index, elem) {
		
	});


## Answer

Unfortunately, CSS [Level 3 attribute selectors](https://www.w3.org/TR/css3-selectors/#attribute-selectors) are very limited (and [Level 4
in draft](https://www.w3.org/TR/selectors4/#attribute-selectors) are not much better).

You will have to use a combination of infix-match and prefix-match attribute selectors to achieve a
robust selection of all elements with a class that begins with `vd-`:

    [class*=" vd-"], [class^="vd-"] {
        ...
    }

The first one will select elements with value of attribute `class` containing `<space>vd-` (cases
like `foo vd-...`) and the second one will patch the corner case of `vd-` class being the first one
(cases like `vd-bar foo`).

### Performance

One might be tempted to use *only infix* (substring) selector, on account of assumed performance
penalty incurred by usage of double attribute selectors (infix and prefix match), knowingly
sacrificing the accuracy and accepting mismatches like
[`yrivd-button`](https://en.wiktionary.org/wiki/yrivd) for `vd-button`.

However, [measurements show](https://jsperf.com/css-attribute-selectors-from-javascript/) there is
**no** statistically significant **difference** in performance between **only infix** match selector
**and** the combined **infix+prefix** match selector (compare `substringMatch` and
`classPrefixSelector` on [`jsperf.com`](https://jsperf.com/css-attribute-selectors-from-javascript/)
test).

The combined selector is only **2% to 7% slower** (depending on browser) than a single attribute
value match selector.

If someone is concerned about that, she shouldn't use attribute selectors at all, but only class
selectors `.vd-name`, **which are 2 to 3 times faster** than any of the attribute selectors
(including the simplest one `[attr]`, which only tests for attribute existence).

