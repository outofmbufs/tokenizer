# tokenizer
A simple tokenizer inspired by the example given in the python [re](https://docs.python.org/3/library/re.html) module, with some additional capabilities.

The Tokenizer class:
 - Allows a straightforward specification of token names and corresponding regular expressions, separated from the tokenizing logic:

```
	rules = [
	        TokenMatch('WHITESPACE', r'\s+'),
	        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
	        TokenMatch('CONSTANT', r'-?[0-9]+'),
	    ]

```
 - Automatically creates a `TokenID` Enum type from all of the token names given (e.g., the 'WHITESPACE', 'IDENTIFIER', etc above).
 - Defines a `Token` type with an id (a `TokenID` Enum), a value (usually a string), and source location information (where it came from in the line).
 - Can perform simple type conversions so the `value` in a `Token` can be automatically converted to something else (e.g., an integer) if that is preferred over the default string representation.
 - Can switch rule sets triggered by specific tokens.

There is also a TokStreamEnhancer providing:
 - Concatenation of multiple input streams
 - N-level "peek" / "unget"
 - A way to remember ("mark") a spot in the token stream and, if desired, unget tokens ("unwind") all the way back to that point.
 - Two more ways (beyond just StopIteration) to handle EOF: a one-time EOF token prior to the StopIteration, or an infinite supply of EOF tokens (never causing StopIteration).

The TokStreamEnhancer does not depend on the `Tokenizer` class; it can be layered onto any iterator that provides a stream of arbitrary objects.

## Using the Tokenizer

In the simplest case, a `Tokenizer` is constructed from a sequence of `TokenMatch` objects and an (optional) input source. Each `TokenMatch` is:

 - A string name. This will become the identifier in the `TokenID` Enum automatically created.
 - A regular expression.

The TokenMatch class can also be subclassed to provide additional token-specific functionality as will be shown later.

A `Tokenizer` is created from a sequence of `TokenMatch` objects and (typically) an open file:

    from tokenizer import TokenMatch, Tokenizer

    rules = [
        TokenMatch('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatch('CONSTANT', r'-?[0-9]+'),
    ]
    tkz = Tokenizer(rules, open('example-input', 'r'))

Anything that is an iterable of strings works as input:

    tkz = Tokenizer(rules, ["first string, line 1", "second, line 2"])

The most common/simplest code uses the tokens() method which generates a sequence of Token objects from the input specified at initialization time:

    from tokenizer import TokenMatch, Tokenizer

    rules = [
        TokenMatch('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatch('CONSTANT', r'-?[0-9]+'),
    ]
    with open('example-input', 'r') as f:
        for token in Tokenizer(rules, f).tokens():
            print(token.id, repr(token.value))

and, given this example-input file:

    abc123 def    ghi_jkl     123456

outputs:

    TokenID.WHITESPACE '    '
    TokenID.IDENTIFIER 'abc123'
    TokenID.WHITESPACE ' '
    TokenID.IDENTIFIER 'def'
    TokenID.WHITESPACE '    '
    TokenID.IDENTIFIER 'ghi_jkl'
    TokenID.WHITESPACE '     '
    TokenID.CONSTANT '123456'
    TokenID.WHITESPACE '\n'

The input can be provided to the `tokens` method instead of being supplied at `Tokenizer` creation time:

    tkz = Tokenizer(rules)
    with open('example-input', 'r') as f:
        for token in tkz.tokens(f):
            print(token.id, repr(token.value))

To directly tokenize a specific string:

    tkz = Tokenizer(rules)
    for token in tkz.string_to_tokens("this   is\ninput 123"):
        print(token.id, repr(token.value))
    
resulting in this output:

    TokenID.IDENTIFIER 'this'
    TokenID.WHITESPACE '   '
    TokenID.IDENTIFIER 'is'
    TokenID.WHITESPACE '\n'
    TokenID.IDENTIFIER 'input'
    TokenID.WHITESPACE ' '
    TokenID.CONSTANT '123'

### Enum (TokenID) Creation/Management

A Tokenizer object has an attribute, `TokenID`, containing an Enum used for token identifiers; it is normally created automatically from the rules:

    from tokenizer import TokenMatch, Tokenizer

    rules = [
        TokenMatch('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatch('CONSTANT', r'-?[0-9]+'),
    ]
    tkz = Tokenizer(rules, open('example-input', 'r'))
    print(tkz.TokenID)
    for id in tkz.TokenID:
        print(f"  {id!r}")

This will output:

    <enum 'TokenID'>
      <TokenID.CONSTANT: 1>
      <TokenID.IDENTIFIER: 2>
      <TokenID.WHITESPACE: 3>

If the application wants more control over the TokenID Enum it can explicitly request one be created from the rules without instantiating a full Tokenizer:

    foo = Tokenizer.create_tokenID_enum(rules)
    print(foo)
    for id in foo:
        print(f"  {id!r}")

which generates the same output:

    <enum 'TokenID'>
      <TokenID.CONSTANT: 1>
      <TokenID.IDENTIFIER: 2>
      <TokenID.WHITESPACE: 3>

An application can also pass a TokenID Enum into Tokenizer (whether the application hand-built its own Enum or used the create_tokenID_enum method). Here is an example with a hand-build Enum for token IDs:

    from tokenizer import TokenMatch, Tokenizer
    from enum import Enum

    rules = [
        TokenMatch('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatch('CONSTANT', r'-?[0-9]+'),
    ]

    class Foo(Enum):
        WHITESPACE = 17
        IDENTIFIER = 42
        CONSTANT = 3
        
    tkz = Tokenizer(rules, open('example-input', 'r'), tokenIDs=Foo)
    print(tkz.TokenID)
    for id in tkz.TokenID:
        print(f"  {id!r}")

which will output:

    <enum 'Foo'>
      <Foo.WHITESPACE: 17>
      <Foo.IDENTIFIER: 42>
      <Foo.CONSTANT: 3>

## Subclassing TokenMatch

Additional token processing can be done by subclassing `TokenMatch`; here is an outline of the base class:

    class TokenMatch:
        def __init__(self, tokname, regexp, /):
            self.tokname = tokname
            self.regexp = regexp

        def matched(self, minfo, /):
            return minfo

If a subclass needs additional parameters it should override the init function something like this:

    # Example adding a keyword argument 'clown' to a TokenMatch subclass
    class ExampleSubclass(TokenMatch):
        def __init__(self, *args, clown='bozo', **kwargs):
	    super().__init__(*args, **kwargs)
	    self.clown = clown

When the framework encounters a regexp match, it creates a `MatchedInfo` namedtuple and passes it to the appropriate `matched` method for further processing. The method is expected to return a `MatchedInfo` -- possibly a new one with modified fields. As can be seen above, the base class implementation simply returns the given object unchanged.

Two fields in the `MatchedInfo` (`minfo` argument) are relevant for extending functionality:

 - tokname
 - value

and in some cases the `tokenizer` field will come into play.

The `tokname` field passed in will be `self.tokname` -- in other words, the name corresponding to this `TokenMatch` object. This is passed in to support two situations:

 - A `matched` method can request the token be ignored by replacing `tokname` with None. This is often useful for whitespace tokens that separate other tokens but otherwise should just be ignored.
 - If multiple regexp patterns (possibly each with a separate `matched` implementation) should ultimately result in a single token type, that name can be set explicitly by the `matched` method. See the OCTAL_CONSTANT example below for an example of this.

For the "ignore this token" situation, consider this subclass implementation:

    # NOTE: This is provided by the tokenizer module
    class TokenMatchIgnore(TokenMatch):
        def matched(self, minfo, /):
            return minfo._replace(tokname=None)

Here's an example with whitespace being ignored:

    from tokenizer import TokenMatch, Tokenizer
    from tokenizer import TokenMatchIgnore

    rules = [
        TokenMatchIgnore('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatch('CONSTANT', r'-?[0-9]+'),
    ]
    with open('example-input', 'r') as f:
        for token in Tokenizer(rules, f).tokens():
            print(token.id, repr(token.value))

and, given this example-input file:

    abc123 def    ghi_jkl     123456

the token sequence would now be:

    TokenID.IDENTIFIER 'abc123'
    TokenID.IDENTIFIER 'def'
    TokenID.IDENTIFIER 'ghi_jkl'
    TokenID.CONSTANT '123456'

In comparison to the earlier example, the WHITESPACE tokens have been discarded by the framework itself.

For some tokens it may be handy to have the value be something other than a string. Here is a fairly obvious example of this for integer tokens:

    class TokenMatchInt(TokenMatch):
        def matched(self, minfo, /):
            return minfo._replace(value=int(minfo.value))

This replaces the `value` field in the `minfo` with a converted (by `int()`) value. Again this particular subclass is common enough that it is already included in the tokenizer module.

As a more general concept:

    class TokenMatchConvert(TokenMatch):
        def __init__(self, *args, converter=int, **kwargs):
            super().__init__(*args, **kwargs)
            self.converter=converter

        def matched(self, minfo, /):
            return minfo._replace(value=self.converter(minfo.value))

This can be used to perform any simple conversion of a value, possibly with the help of `partial` from functools. For example, this converts python octal format:

    from tokenizer import TokenMatch, Tokenizer
    from tokenizer import TokenMatchInt, TokenMatchConvert, TokenMatchIgnore
    from functools import partial

    octal = partial(int, base=8)

    rules = [
        TokenMatchConvert('OCTAL_CONSTANT', r'0o([0-7]+)', converter=octal),
        TokenMatchInt('CONSTANT', r'-?[0-9]+'),
        TokenMatchIgnore('WHITESPACE', r'\s+'),
    ]    
    tkz = Tokenizer(rules)
    for token in tkz.string_to_tokens("123 0o377"):
            print(token.id, repr(token.value))

This will output:

    TokenID.CONSTANT 123
    TokenID.OCTAL_CONSTANT 255

Unfortunately this creates two different token types for higher levels to handle: CONSTANT and OCTAL_CONSTANT -- even though once the values have been converted to integer the distinction is most likely irrelevant. There are two ways to change this example to "collapse" the two token types back together.

One way that DOES NOT WORK would be to simply give two different TokenMatch lines with the same `tokname`:

    from tokenizer import TokenMatch, Tokenizer
    from tokenizer import TokenMatchInt, TokenMatchConvert, TokenMatchIgnore
    from functools import partial

    octal = partial(int, base=8)

    rules = [
        # DO NOT DO THIS -- DO NOT DUPLICATE tokname FIELDS
        TokenMatchConvert('CONSTANT', r'0o([0-7]+)', converter=octal),
        TokenMatchInt('CONSTANT', r'-?[0-9]+'),
        TokenMatchIgnore('WHITESPACE', r'\s+'),
    ]    
    tkz = Tokenizer(rules)
    for token in tkz.string_to_tokens("123 0o377"):
            print(token.id, repr(token.value))

This will result in:

    ValueError: Duplicate tokname: CONSTANT

Instead, the real `TokenMatchConvert` (see the source code) allows specification of an alt_tokname which will be used to replace the `tokname` as part of the post-processing. Here is the source code for that more-elaborate TokenMatchConvert:

    class TokenMatchConvert(TokenMatch):
        def __init__(self, *args, converter=int, alt_tokname=None, **kwargs):
            super().__init__(*args, **kwargs)
            self.converter = converter
            self.alt_tokname = alt_tokname

        def matched(self, minfo, /):
            replacements = {'value': self.converter(minfo.value)}
            if self.alt_tokname is not None:
                replacements['tokname'] = self.alt_tokname
            return minfo._replace(**replacements)

which is used this way:

    rules = [
        TokenMatchConvert(
	        'OCTAL_CONSTANT', r'0o([0-7]+)',
	        converter=octal, alt_tokname='CONSTANT'),
        TokenMatchInt('CONSTANT', r'-?[0-9]+'),
        TokenMatchIgnore('WHITESPACE', r'\s+'),
    ]    

and when the full example is run the output will be:

    TokenID.CONSTANT 123
    TokenID.CONSTANT 255

Note that the OCTAL_CONSTANT TokenID Enum name still exists, and is still the `tokname` of the TokenMatchConvert object itself, but gets altered in the `matched` call at runtime. OCTAL_CONSTANT exists but is only ever seen by the framework internals and never produced as an actual Token.

Recapping, the `tokenizer` module provides these subclasses automatically:

 - TokenMatchIgnore -- to match character sequences and ignore them
 - TokenMatchInt -- special case of TokenMatchConvert
 - TokenMatchConvert -- to allow arbitrary conversion functions and optional tokname masquerading.

In addition there are two more slightly more complex subclasses:

 - TokenMatchIgnoreWhiteSpaceKeepNewline
 - TokenMatchRuleSwitch

Suppressing WHITESPACE tokens is often handy, but sometimes the end of a line has semantic meaning and needs to be visible as a token. The `TokenMatchIgnoreWhiteSPaceKeepNewline` subclass handles this. The subclass implementation ignores character sequences that match the given regular expression, EXCEPT if they contain one or more '\n' character, in which case the NEWLINE token will be generated.

For example:

    from tokenizer import TokenMatch, Tokenizer
    from tokenizer import TokenMatchIgnoreWhiteSpaceKeepNewline
    from tokenizer import TokenMatchInt

    rules = [
        TokenMatchIgnoreWhiteSpaceKeepNewline('NEWLINE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatchInt('CONSTANT', r'-?[0-9]+')
    ]

    tkz = Tokenizer(rules)
    for token in tkz.string_to_tokens('foo  1234   bar  \n   \n  baz\n'):
        print(token.id, repr(token.value))

This will output:

    TokenID.IDENTIFIER 'foo'
    TokenID.CONSTANT 1234
    TokenID.IDENTIFIER 'bar'
    TokenID.NEWLINE '\n'
    TokenID.IDENTIFIER 'baz'
    TokenID.NEWLINE '\n'

Note in this example that any whitespace (matching `r'\s+'`) has been ignored, UNLESS it contains one (or more) newlines. If there are '\n' characters in there, then *one* NEWLINE token is generated (with a `value` of just '\n' regardless of how many newlines there were). If different behavior is desired, it is easy enough to write a different customized subclass.

Lastly, to switch between different sets of rules based on the appearance of a token, the subclass `TokenMatchRuleSwitch` is available, it will be discussed next.

## Multiple TokenMatch rulesets

Some lexical processing is modal - the appearance of a token will change the lexical processing for the following tokens. To allow for simple versions of this, rules can be grouped into named rulesets and selected based on tokens.

For example:

    from tokenizer import TokenMatch, Tokenizer
    from tokenizer import TokenMatchRuleSwitch

    group1 = [
        TokenMatch('ZEE', r'z'),
        TokenMatchRuleSwitch('ALTRULES', r'/', new_rulename='ALT')
    ]

    group2 = [
        TokenMatch('ZED', r'z'),
        TokenMatchRuleSwitch('MAINRULES', r'/', new_rulename=None)
    ]

    # NOTE: None (as a ruleset "name") must always be present.
    #       It is the initial (or "primary") ruleset. Other ruleset
    #       names are arbitrary.
    rules = {None: group1, 'ALT': group2}

    tkz = Tokenizer(rules)
    for token in tkz.string_to_tokens('zz/z/z'):
        print(token.id, repr(token.value))

This will output:

    TokenID.ZEE 'z'
    TokenID.ZEE 'z'
    TokenID.ALTRULES '/'
    TokenID.ZED 'z'
    TokenID.MAINRULES '/'
    TokenID.ZEE 'z'

In this example sometimes 'z' is a ZEE, and sometimes a ZED, depending on which ruleset has been activated. The `TokenMatchRuleSwitch` subclass takes a `new_rulename` argument and switches the active ruleset accordingly.

There can be any number of rulesets with arbitrary names, but one of the rulesets must always have None (the python object, not a string 'None') as its name. That is the default/primary ruleset and the one that is first active when processing begins. NOTE: in the earlier examples where a sequence of TokenMatch objects was passed in, not a mapping, internally they were converted into a one-deep mapping from None to the given sequence.

If no `new_rulename` is given to TokenMatchRuleSwitch then it cycles through the rules in (dictionary - as defined) order, wrapping around from the end back to the front. What this means is that in a simple case such as the above, it's not even necessary to specify `new_rulename` and a bare TokenMatchRuleSwitch will just switch back and forth. Thus, this works if substituted for the group1/group2 initializations:

    group1 = [
        TokenMatch('ZEE', r'z'),
        TokenMatchRuleSwitch('SWITCH', r'/')
    ]

    group2 = [
        TokenMatch('ZED', r'z'),
        TokenMatchRuleSwitch('SWITCH', r'/')
    ]

and the output will be:

    TokenID.ZEE 'z'
    TokenID.ZEE 'z'
    TokenID.SWITCH '/'
    TokenID.ZED 'z'
    TokenID.SWITCH '/'
    TokenID.ZEE 'z'


HOWEVER, there is a trap lurking here -- generally it is a bad idea to give multiple TokenMatch objects the same `tokname`, and indeed the framework prohibits that sort of duplication within a given ruleset name. It is also easy to just give each one a nominally different name:

    group1 = [
        TokenMatch('ZEE', r'z'),
        TokenMatchRuleSwitch('SWITCH_1', r'/')
    ]

    group2 = [
        TokenMatch('ZED', r'z'),
        TokenMatchRuleSwitch('SWITCH_2', r'/')
    ]

One last caution: sometimes, rather than going hog-wild with modal rulesets, it may be simpler to implement a pre-processor on the input instead. For example, most C compilers work that way rather than trying to tokenize the C comment format in some modal way, though it appears to be reasonably possible with this mechanism (see the tokenizer test code `test_C` example).


## Tokenizer odds and ends

Input can be pre-processed, because anything that duck-types as an iterable of strings is acceptable. For example, if backslash-newline sequences need to be elided (in effect combining two adjacent lines), that's easy to do. A built-in filter, `linefilter' does this:

    from tokenizer import TokenMatch, Tokenizer

    rules = [
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
    ]

    f = open('example-input', 'r')
    tkz = Tokenizer(rules, Tokenizer.linefilter(f))
    for t in tkz.tokens():
        print(t.id, repr(t.value))

If given this example-input file:

    foo\
    bar

where the first line ends with "backslash newline", the output will be:

    TokenID.IDENTIFIER 'foobar'

Note that the backslash/newline has been completely filtered out by `linefilter` and a single IDENTIFIER that was "split" across that escaped line boundary has been produced. Applications can provide their own, more-elaborate, input filters if necessary.

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

There is no requirement the underlying tokstreams be a `Tokenizer` or even be anything more than iterables. For example, this works just fine:

    xz = TokStreamEnhancer([1,2,3], [4,5,6])
    t1 = xz.gettok()
    t2 = xz.gettok()
    t3, t4 = xz.peektoks(2)
    xz.ungettok(t2)
    t2b, t3b, t4b = xz.gettoks(3)

    print(t1)
    print(t2)
    print(t3, t4)
    print(t2b, t3b, t4b)
    
and outputs:

    1
    2
    3 4
    2 3 4

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

