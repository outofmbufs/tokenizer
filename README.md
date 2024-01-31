# tokenizer
A simple tokenizer, layered on top of python regular expressions ('re' module)

This is inpired by the simple tokenizer example given in the python [re](https://docs.python.org/3/library/re.html) module, with some additional capabilities.

The Tokenizer class:
 - Keeps the regexp/token specification separate from the tokenizing logic.
 - Has simple "post processing" function capability so, for example, tokens representing numbers can have their values as int rather than str.
 - Creates a `TokenID` Enum for token types.
 - Provides "source string" information with each token, which can be helpful for better syntax error messages.
 - Has very simple modal capabilities if different rules should be triggered by specific tokens

There is also a TokStreamEnhancer providing:
 - Concatenation of multiple input streams
 - N-level "peek" / "unget"
 - A way to remember ("mark") a spot ijn the token stream and, if desired, unget tokens ("unwind") all the way back to that point.
 - Two more ways (beyond just StopIteration) to handle EOF: a one-time EOF token prior to the StopIteration, or an infinite supply of EOF tokens (never causing StopIteration).

The TokStreamEnhancer does not depend on the `Tokenizer` class; it can be layed onto any iterator that provides a stream of arbitrary objects.

## Using the Tokenizer

The `Tokenizer` takes a `TokenRuleSuite` made up of `TokenMatch` objects.

Each `TokenMatch` is:
 - A string name. This will become the identifier in the `TokenID` Enum automatically created (by the `TokenRuleSuite`)
 - A regular expression.
 - An optional post-processing function, called a `ppf`

In the simplest scenario, multiple `TokenMatch` objects are given to `TokenRuleSuite` in a list. For example, to tokenize input consisting of simple identifiers and numeric values, separated by whitespace:

    from tokenizer import TokenMatch, TokenRuleSuite, Tokenizer

    rules = [
        TokenMatch('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatch('CONSTANT', r'-?[0-9]+'),
    ]

    rule_suite = TokenRuleSuite(rules)

At this point the `rule_suite` is suitable for initializing a `Tokenizer` object.

## The Tokenizer object

A `Tokenizer` is created from a `TokenRuleSuite` and (typically) an open file:

    tkz = Tokenizer(rule_suite, open('example-input', 'r'))

Anything that is an iterable of strings works as input:

    tkz = Tokenizer(rule_suite, ["first string, line 1", "second, line 2"])

The input can also be None; `tokens()` can take optional input parameters instead, and `string_to_tokens()` works directly on a supplied string.

The most common/simplest code uses the tokens() method which returns (generates) a sequence of Token objects from the input at initialization time:

    with open('example-input', 'r') as f:
        for token in Tokenizer(rule_suite, f).tokens():
            print(token.id, token.value)

and, given this example-input file:

    abc123 def    ghi_jkl     123456

the above code will output:

    TokenID.IDENTIFIER abc123
    TokenID.WHITESPACE  
    TokenID.IDENTIFIER def
    TokenID.WHITESPACE  
    TokenID.IDENTIFIER ghi_jkl
    TokenID.WHITESPACE  
    TokenID.CONSTANT 123456
    TokenID.WHITESPACE  

Another way to say the same thing:

    tkz = Tokenizer(rule_suite, None)
    with open('example-input', 'r') as f:
        for token in tkz.tokens(f):
            print(token.id, token.value)

To directly tokenize a specific string:

    tkz = Tokenizer(rule_suite, None)
    for token in tkz.string_to_tokens("this   is\ninput 123"):
        print(token.id, token.value)
    
which will output:

    TokenID.IDENTIFIER this
    TokenID.WHITESPACE    
    TokenID.IDENTIFIER is
    TokenID.WHITESPACE 

    TokenID.IDENTIFIER input
    TokenID.WHITESPACE  
    TokenID.CONSTANT 123

The values in these examples are all strings, and that second WHITESPACE token value contains a newline. This would be more apparent using repr():

    tkz = Tokenizer(rule_suite, None)
    for token in tkz.string_to_tokens("this   is\ninput 123"):
        print(token.id, repr(token.value))      # now using repr() here

Now the output will look like this:

    TokenID.IDENTIFIER 'this'
    TokenID.WHITESPACE '   '
    TokenID.IDENTIFIER 'is'
    TokenID.WHITESPACE '\n'
    TokenID.IDENTIFIER 'input'
    TokenID.WHITESPACE ' '
    TokenID.CONSTANT '123'

## Post-processing functions (`ppf`)

It's probably more useful in this example if the CONSTANT tokens were integer values not strings. Perhaps it would also be helpful if WHITESPACE tokens were simply discarded. This can all be handled with a post-processing function in the TokenMatch:

    rules = [
        TokenMatch('WHITESPACE', r'\s+', TokenRuleSuite.ppf_ignored),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatch('CONSTANT', r'-?[0-9]+', TokenRuleSuite.ppf_int),
    ]

The `TokenRuleSuite` class provides a few handy post-processing functions:

 - TokenRuleSuite.ppf_ignored: suppresses the token. The input is matched but the token is not returned.
 - TokenRuleSuite.ppf_int: converts the .value string using int()
 - ppf_keepnewline: suppresses the token, UNLESS it contains a newline (see discussion)

With the above rules and the same string_to_token() call, the output would now be:

    TokenID.IDENTIFIER 'this'
    TokenID.IDENTIFIER 'is'
    TokenID.IDENTIFIER 'input'
    TokenID.CONSTANT 123

## Skipping whitespace but keeping newlines

Using `ppf_ignored` to suppress all WHITESPACE is handy, but sometimes the end of a line has semantic meaning and needs to be visible as a token. We could write an explicit NEWLINE rule, and then change WHITESPACE to not use the 're' \s escape. However, the scenario is common enough that a `ppf_keepnewline` function is available for it:

    rules = [
        TokenMatch('WHITESPACE', r'\s+', TokenRuleSuite.ppf_keepnewline),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatch('CONSTANT', r'-?[0-9]+', TokenRuleSuite.ppf_int),
    ]

but running this leads to an error:

    ... some output and then ...
    KeyError: 'NEWLINE'

because ppf_keepnewline assumes there is a TokenID `NEWLINE` but we have no rule named `NEWLINE`. To fix this, a `TokenMatch` can be created with no corresponding regexp:

    from tokenizer import TokenMatch, TokenRuleSuite, Tokenizer

    rules = [
        TokenMatch('WHITESPACE', r'\s+', TokenRuleSuite.ppf_keepnewline),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatch('CONSTANT', r'-?[0-9]+', TokenRuleSuite.ppf_int),
        TokenMatch('NEWLINE', None)     # no regexp; for ppf_keepnewline
    ]

This fixes the "KeyError: NEWLINE" problem. The "no rule" `TokenMatch` entry for NEWLINE can be in any spot; order doesn't matter.

Now the output will look like:

    TokenID.IDENTIFIER 'abc123'
    TokenID.IDENTIFIER 'def'
    TokenID.IDENTIFIER 'ghi_jkl'
    TokenID.CONSTANT 123456
    TokenID.NEWLINE '\n'

The WHITESPACE has been ignored, except if it contains one (or more) newlines, in which case a NEWLINE token has been generated. Note that only one NEWLINE token will be generated even if there are multiple \n characters in a given run of WHITESPACE. This is usually desired behavior (multiple newlines collapse into a single NEWLINE token). Or, if not, write explicit TokenMatch rules accordingly.

Any number of "no corresponding regexp" tokens can be added this way; they are often useful as sentinels or for other `ppf` functions.

## Writing custom ppf functions

You can supply your own `ppf` functions. They should look like this:

    def ppf_something(trs, id, val):
        return id, do_something(val)

The arguments supplied are:
 - `trs` :: the `TokenRuleSuite` object. One reason this is needed is to return a different TokenID than the one given; use `trs.TokenID` to get at the mapping.
 - `id`  :: the TokenID (note: not the `tokname`) of the rule that fired.
 - `val` :: the (string) value

The function should return a tuple: (id, val) where `id` is a possibly-different tokenID than handed in, and `val` is a possibly-different (i.e., converted) value. For example, to convert float values write this:

    def ppf_something(trs, id, val):
        return id, float(val)

Any exceptions raised by a `ppf` function will bubble out. Applications should catch these exceptions with a try/except around the Tokenizer iteration. Typically if an application wants to report better error messages it will have to manage the iteration itself (i.e., using `next` in a try/except) rather than relying on a `for` loop as these simple examples do.

To cause a token to be ignored, a `ppf` should return `None, None`

Use [partial](https://docs.python.org/3/library/functools.html#functools.partial) if additional arguments are needed in a custom `ppf` function.

Here, for example, is a `ppf` implementation that can be used to turn an IDENTIFIER into a KEYWORD if it matches the given keyword table:

    from tokenizer import TokenMatch, TokenRuleSuite, Tokenizer
    import functools

    keywords = ['for', 'while', 'if', 'else' ]

    def _ppf_keyword(trs, id, val, keyword_table=None):
        if val in keyword_table:
            return trs.TokenID.KEYWORD, val
        else:
            return id, val

    # demonstration of using partial() for more ppf arguments
    ppf_keyword = functools.partial(_ppf_keyword, keyword_table=keywords)

    rules = [
        TokenMatch('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*', ppf_keyword),
        TokenMatch('KEYWORD', None),     # for ppf_keyword
    ]

    rule_suite = TokenRuleSuite(rules)

    for t in Tokenizer(rule_suite, ["in a while\n", "crocodile"]).tokens():
        print(t.id, repr(t.value))

The output will be:

    TokenID.IDENTIFIER 'in'
    TokenID.WHITESPACE ' '
    TokenID.IDENTIFIER 'a'
    TokenID.WHITESPACE ' '
    TokenID.KEYWORD 'while'
    TokenID.WHITESPACE '\n'
    TokenID.IDENTIFIER 'crocodile'

## Tokenizer odds and ends

Input can be pre-processed, because anything that duck-types as an iterable of strings is acceptable. For example, if backslash-newline sequences need to be elided (in effect combining two adjacent lines), that's easy to do. A built-in filter, `linefilter' does this:

    from tokenizer import TokenMatch, TokenRuleSuite, Tokenizer

    rules = [
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
    ]

    rule_suite = TokenRuleSuite(rules)

    f = open('example-input', 'r')
    tkz = Tokenizer(rule_suite, Tokenizer.linefilter(f))
    for t in tkz.tokens():
        print(t.id, repr(t.value))

If given this example-input file:

    foo\
    bar

where the first line ends with "backslash newline", the output will be:

    TokenID.IDENTIFIER 'foobar'

Note that the backslash/newline has been completely filtered out by `linefilter` and a single IDENTIFIER that was "split" across that escaped line boundary has been produced. Applications can provide their own, more-elaborate, input filters if necessary.

## Multiple TokenMatch groups

Some lexical processing is modal - the appearance of a token will change the lexical processing for the following tokens. To allow for simple versions of this, a `TokenRuleSuite` can group rules into named subgroups. To use this provide the rules in a mapping (dict) instead of a list:

    from tokenizer import TokenMatch, TokenRuleSuite, Tokenizer

    group1 = [
        TokenMatch('ZEE', r'z'),
        TokenMatch('SWITCH', '/', TokenRuleSuite.ppf_altrules)
    ]

    group2 = [
        TokenMatch('ZED', r'z'),
        TokenMatch('SWITCH', '/', TokenRuleSuite.ppf_mainrules)
    ]

    rules = {TokenRuleSuite.DEFAULT_NAME: group1,
             TokenRuleSuite.ALT_NAME: group2}

    rule_suite = TokenRuleSuite(rules)

    tkz = Tokenizer(rule_suite, None)
    for token in tkz.string_to_tokens('zz/z/z'):
        print(token.id, repr(token.value))

This will output:

    TokenID.ZEE 'z'
    TokenID.ZEE 'z'
    TokenID.SWITCH '/'
    TokenID.ZED 'z'
    TokenID.SWITCH '/'
    TokenID.ZEE 'z'

See the code for details on naming/specifying more than two sets of rules. Note, of course, that at some point of complexity it may become a better idea to write a custom lexical analyzer rather than get too fancy with `ppf` functions and multiple rule sets.

# TokStreamEnhancer

A separate class, `TokStreamEnhancer`, provides higher-level functionality on the token stream generated by `Tokenizer`.

In the simplest case, if `tkz` is a `Tokenizer` object, an enhanced stream can be made like this:

    xz = TokStreamEnhancer(tkz)

Any number of token streams can be concatenated:

    rules = blah blah blah
    tkz1 = Tokenizer(rules, open('file1', 'r'))
    tkz2 = Tokenizer(rules, open('file2', 'r'))
    xz = TokStreamEnhancer(tkz1, tkz2)

    for t in xz:
        print(t)


This will print tokens from file1 followed by tokens from file2. Any number of token streams can be specified this way.

# peek and unget

In all of these examples, `xz` is a `TokStreamEnhancer`.

To peek at a token in an enhanced token stream, without "getting" it:

    t0 = xz.peektok()
    t1 = xz.peektok()
    print(t0 == t1)

This will print `True`.

To "unget" (put a token back):

    t0 = xz.peektok()         # peek, doesn't get it
    t1 = xz.gettok()          # gets the token
    xz.ungettok(t1)           # puts it back
    t2 = xz.gettok()          # gets it again

    print(t == t1, t1 == t2)

will print `True True`.

It "works" to unget any arbitrary object, but is considered bad practice:

    t1 = xz.gettok()
    s = "doesn't even have to be a Token object"
    xz.ungettok(s)
    t2 = xz.gettok()
    print(t2)

this will print the string `s` and cause your coworkers to stick pins in the eyes of your voodoo doll.

To peek ahead multiple tokens at a time:

    t0, t1, t2, = xz.peektoks(3)

To get multiple tokens at a time:

    t0, t1, t2 = xz.gettoks(3)

There is no atomicity or other semantics implied by getting multiple tokens at once vs making N individual calls.

If the end of the (last) token stream is encountered during a peek or get, the behavior depends on some of the TokStreamEnhancer arguments, so this is a good time to look at those:

    class TokStreamEnhancer:
    def __init__(self, *tokstreams, lasttok=None, eoftok=None):

The arguments are an arbitrary number of token streams, as already discussed, plus two other optional keyword arguments:

    lasttok -- will be supplied before reaching StopIteration
    eoftok -- will be supplied IN LIEU (repeatedly) of StopIteration

Specifying a `lasttok` is equivalent to adding a one-token stream (of `lasttok`) to the tokstreams argument. This last token will be returned as the last token, and the next token request after that will either raise StopIteration or return eoftok if one was given.

Specifying an `eoftok` is a request to turn OFF the raising of StopIteration and instead return the given `eoftok`, INDEFINITELY (i.e., multiple times). NOTE: This breaks the "iterator protocol" and, indeed, the `TokStreamEnhancer` will not allow itself to be used as an __iter__ if an `eoftok` has been given.

These two arguments allow for several different ways to handle the end of the token stream:

 - If neither `lasttok` nor `eoftok` was specified, StopIteration is raised when peektok() or gettok() are called and there are no more tokens in the last of the tokstreams.
 - If only a `lasttok` was specified, then after the last token from the last of the tokstreams has been given out, the next token will be `lasttok`. After `lasttok` has been given out (i.e., by gettok), StopIteration will be raised by peektok or gettok.
 - If only an `eoftok` was specified, then instead of raising StopIteration that token will be returned, indefinitely (i.e., repeatedly) for peektok/gettok, once the last of the tokstreams has been exhausted.
 - If both `lasttok` and `eoftok` are specified, then the two semantics are combined in the obvious way.
 
## EOF testing

   no_more_tokens = xz.at_eof()

The `at_eof` method returns True if peektok or gettok would return `eoftok` if one had been specified, or would raise StopIteration if no `eoftok` was specified.

## Conditional peek

Notionally it's sometimes convenient to combine a conditional test (usually a test on the token id) with a peek. Method `peek_if` does this:

    def peekif(self, pred, /, *, eofmatch=_NOTGIVEN):
        """Return (peek) a token, if pred(token) is True, else return None.

        If optional argument eofmatch is given, it is returned (regardless
        of pred match) if at_eof(). Otherwise peek semantics for eof.
        """

## marking and unwinding

Sometimes in a recursive-descent parser it may be convenient to save a spot in the token stream, try to parse something, and then be able to easily unwind (unget) all the tokens back to the saved spot.

The methods `tokmark` and `acceptmarks` provide this.

To mark a spot in a stream, use `tokmark` as a context manager:

    with xz.tokmark():
        t1 = xz.gettok()
        t2 = xz.gettok()

    print(xz.gettok() == t1)

This will print `True` because without a call to `acceptmarks`, any tokens gotten within the context manager will be put back when the context manager exits.

To "accept" the tokens and stop the unwinding, invoke `acceptmarks()`:

    with xz.tokmark():
        t1 = xz.gettok()
        t2 = xz.gettok()
	xz.acceptmarks()

    print(xz.gettok() == t1)

Note that the `acceptmarks` affects the ENTIRE tokmark context. It does not matter whether tokens are gotten before or after the accept; what matters is whether or not an accept occurred ANYWHERE within the context. All tokens in that context will be accepted and not unwound when the context exits. This is equivalent:

    with xz.tokmark():
	xz.acceptmarks()
        t1 = xz.gettok()
        t2 = xz.gettok()

    print(xz.gettok() == t1)

It is possible to nest contexts, though programmers are cautioned that the semantics of this start to get subtle fast. If necessary, save an explicit context variable in the WITH statement and use it for acceptmarks calls:

    with xz.tokmark() as ctx:
        t1 = xz.gettok()
        with xz.tokmark() as ctx2:
            t2 = xz.gettok()
            ctx2.acceptmarks()
        t3 = xz.gettok()
    t4 = xz.gettok()

Figuring out what tokens `t3` and `t4` are is left as an exercise for the programmer.

