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

  ... XXX TODO XXX ...
  


