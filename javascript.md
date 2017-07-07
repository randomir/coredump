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

