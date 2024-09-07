# tokenizer
A simple tokenizer inspired by the example given in the python [re](https://docs.python.org/3/library/re.html) module, with some additional capabilities:

 - The `TokenMatch` class, and subclasses, allow specification of token names, processing rules, and corresponding regular expressions, separated from the tokenizing framework logic:

```
    rules = [
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatchIgnore('WHITESPACE', r'\s+'),
        TokenMatchInt('CONSTANT', r'-?[0-9]+'),
    ]

```
 - Automatically creates a `TokenID` Enum type from all of the token names given (e.g., the 'WHITESPACE', 'IDENTIFIER', etc above).
 - Defines a `Token` object containing an `id` (a `TokenID` Enum), a `value` (usually a string), and source location information (where it came from in the input). 
 - Generates a stream of `Token` objects from string inputs, which can come from file objects or indeed just be a string or an iterable of strings.
 - Provides some TokenMatch subclasses with additional capabilities, such as converting the `value` string to something more appropriate (e.g., an integer for a string of digits match).
  - Can be extended for further custom token processing by subclassing TokenMatch.
 - Can switch rule sets triggered by specific tokens for modal tokenizing.

There is also a TokStreamEnhancer providing:
 - Concatenation of multiple input streams
 - N-level "peek" / "unget"
 - A way to remember ("mark") a spot in the token stream and, if desired, unget tokens ("unwind") all the way back to that point.
 - Two more ways (beyond just StopIteration) to handle EOF: a one-time EOF token prior to the StopIteration, or an infinite supply of EOF tokens (never causing StopIteration).

The TokStreamEnhancer does not depend on the `Tokenizer` class; it can be layered onto any iterator that provides a stream of arbitrary objects.

## Using the Tokenizer

In the simplest case, a `Tokenizer` is constructed from a sequence of `TokenMatch` objects and an (optional) input source. Each `TokenMatch` has two attributes:

 - tokname: -- string. This becomes the identifier in the `TokenID` Enum automatically created.
 - regexp: -- string. A regular expression.

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

Some applications may need token types defined without any match; this can be specified with `None` for the regular expression or with the `TokenIDOnly` subclass of `TokenMatch` (they are equivalent):

    from tokenizer import TokenMatch, TokenIDOnly, Tokenizer
    rules = [
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatch('CONSTANT', r'-?[0-9]+'),
        TokenMatch('FOO', None),
        TokenIDOnly('BAR')
    ]
    tkz = Tokenizer(rules)
    print(tkz.TokenID)
    for id in tkz.TokenID:
        print(f"  {id!r}")


Output:

    <enum 'TokenID'>
      <TokenID.BAR: 1>
      <TokenID.CONSTANT: 2>
      <TokenID.FOO: 3>
      <TokenID.IDENTIFIER: 4>

Applications can call (classmethod) `create_tokenID_enum` directly to create a TokenID Enum separately from instantiating a `Tokenizer`:

    ids = Tokenizer.create_tokenID_enum(rules)

in which case `ids` will be the Enum. That Enum (or one constructed entirely custom by the application) can be passed into `Tokenizer` for use in lieu of generating a different one again. Here is an example with a hand-build Enum for token IDs:

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

    tkz = Tokenizer(rules, tokenIDs=Foo)
    print(tkz.TokenID)
    for id in tkz.TokenID:
        print(f"  {id!r}")

which will output:

    <enum 'Foo'>
      <Foo.WHITESPACE: 17>
      <Foo.IDENTIFIER: 42>
      <Foo.CONSTANT: 3>

## Available TokenMatch Enhancements

Enhanced matching/processing features can be implemented by subclasses of `TokenMatch`, several of which have been written and provided in the tokenizer module:

 - TokenIDOnly
 - TokenMatchIgnore
 - TokenMatchIgnoreButKeep
 - TokenMatchInt
 - TokenMatchConvert
 - TokenMatchKeyword

These, and how to write custom classes, will be described next.

### TokenIDOnly

This was already shown in an example earlier. This allows an application to create additional token types (i.e., elements of the TokenID Enum) that it may create on its own but will have no corresponding regular expression to match:

    from tokenizer import TokenIDOnly, Tokenizer
    rules = [
        TokenIDOnly('FOO'),
        TokenIDOnly('BAR'),
        TokenIDOnly('BAZ')
    ]
    tkz = Tokenizer(rules)

This creates a Tokenizer that never matches anything (a real usage would obviously have some TokenMatch elements) and has a `tkz.TokenID` Enum containing FOO, BAR, and BAZ.

### TokenMatchIgnore

`TokenMatchIgnore` can be used for a token that needs to be matched for syntax reasons but does not need to appear in the stream for upper-level parsing. The most common example of this is whitespace:

    # WITHOUT TokenMatchIgnore
    from tokenizer import TokenMatch, Tokenizer
    rules = [
        TokenMatch('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
    ]
    tkz = Tokenizer(rules)
    for token in tkz.string_to_tokens("This is a test"):
        print(token.id, repr(token.value))

This will output:

    TokenID.IDENTIFIER 'This'
    TokenID.WHITESPACE ' '
    TokenID.IDENTIFIER 'is'
    TokenID.WHITESPACE ' '
    TokenID.IDENTIFIER 'a'
    TokenID.WHITESPACE ' '
    TokenID.IDENTIFIER 'test'
        
Using TokenMatchIgnore like this will discard WHITESPACE tokens:

    from tokenizer import TokenMatch, Tokenizer, TokenMatchIgnore
    rules = [
        TokenMatchIgnore('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
    ]
    tkz = Tokenizer(rules)
    for token in tkz.string_to_tokens("This is a test"):
        print(token.id, repr(token.value))

Output:

    TokenID.IDENTIFIER 'This'
    TokenID.IDENTIFIER 'is'
    TokenID.IDENTIFIER 'a'
    TokenID.IDENTIFIER 'test'


### TokenMatchIgnoreButKeep
In some cases whitespace can be suppressed but newlines have semantic significance and should be preserved. `TokenMatchIgnoreButKeep` is made for this:

    from tokenizer import TokenMatch, Tokenizer
    from tokenizer import TokenMatchIgnoreButKeep

    rules = [
        TokenMatchIgnoreButKeep('NEWLINE', r'\s+', keep='\n'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
    ]
    tkz = Tokenizer(rules)
    for token in tkz.string_to_tokens('foo   bar  \n   \n  baz\n'):
        print(token.id, repr(token.value))

output:

    TokenID.IDENTIFIER 'foo'
    TokenID.IDENTIFIER 'bar'
    TokenID.NEWLINE '\n'
    TokenID.IDENTIFIER 'baz'
    TokenID.NEWLINE '\n'

In this example any whitespace (matching `r'\s+'`) will be ignored, UNLESS it contains one (or more) `keep` characters ('\n' in this example). If the `keep` character appears in the match then one token (NEWLINE in this example) is generated (with a `value` of just one `keep` regardless of how many were present). If different behavior is desired, it is easy enough to write a different customized subclass.

### TokenMatchInt

Converts the value attribute of a token to an integer, and is most-obviously useful for something like a CONSTANT:

    from tokenizer import TokenMatch, Tokenizer, TokenMatchInt
    rules = [
        TokenMatchInt('CONSTANT', r'-?[0-9]+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatch('EQUALS', r'='),
    ]
    tkz = Tokenizer(rules)
    for token in tkz.string_to_tokens("foo=17"):
        print(token.id, repr(token.value))

will output:

    TokenID.IDENTIFIER 'foo'
    TokenID.EQUALS '='
    TokenID.CONSTANT 17

Notice that the `value` attribute of the CONSTANT is now an integer, not a string, as a result of matching the `TokenMatchInt` rule.

### TokenMatchConvert

This is a generalization of `TokenMatchInt` and allows for arbitrary conversions.

Suppose constants can be simple decimal numbers OR numbers in python octal format. The `TokenMatchConvert` subclass takes an argument, `converter`, that will allow for this:

    from tokenizer import TokenMatch, Tokenizer, TokenMatchIgnore
    from tokenizer import TokenMatchConvert, TokenMatchInt
    import functools

    octal = functools.partial(int, base=8)

    rules = [
        TokenMatchConvert('CONSTANT', r'0o([0-7]+)', converter=octal),
        TokenMatchInt('CONSTANT', r'-?[0-9]+'),
        TokenMatchIgnore('WHITESPACE', r'\s+'),
    ]
    tkz = Tokenizer(rules)
    for token in tkz.string_to_tokens("42 0o377"):
        print(token.id, repr(token.value))

Output:

    TokenID.CONSTANT 42
    TokenID.CONSTANT 255


### TokenMatchKeyword

The `TokenMatchKeyword` subclass provides a simple way to make keywords be their own unique tokens:

    from tokenizer import TokenMatch, Tokenizer
    from tokenizer import TokenMatchKeyword, TokenMatchIgnore

    rules = [
        TokenMatchKeyword('if'),
        TokenMatchKeyword('then'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatchIgnore('WHITESPACE', r'\s+'),
    ]
    tkz = Tokenizer(rules)
    for token in tkz.string_to_tokens("if this then that"):
        print(token.id, repr(token.value))

Note that no regular expression should (generally) be given to `TokenMatchKeyword`. It takes the single string argument given (e.g., 'if') and creates a regular expression that will match it (case-sensitive) and automatically creates a token type with an upper-cased name. The output of the above example will be:

    TokenID.IF 'if'
    TokenID.IDENTIFIER 'this'
    TokenID.THEN 'then'
    TokenID.IDENTIFIER 'that'

__NOTE__: When working with regular expressions, order of presentation matters (this is true also of the examples given in the python 're' module on which this whole exercise is based). If the more-general IDENTIFIER rule appears before the less-general 'if' or 'while' keyword rule, those keywords will appear as IDENTIFIERs instead. This is just how the re-based matching works.

## Writing custom TokenMatch subclasses

To write other TokenMatch enhancments, create a subclass and override one or both of these methods:

    def __init__(self, tokname, regexp, /):
        ...

    def matched(self, minfo, /):
        ...

The `matched` method is called when the framework has a regexp match, and is passed in the argument `minfo` which is a `_MatchedInfo` namedtuple. The `matched` method is expected to return this `_MatchedInfo`, possibly with modifications.

A `matched` method in a subclass can change either or both of these `minfo` fields:

 - tokname
 - value

If `tokname` is set to None, this matched token will be ignored. This is how, for example `TokenMatchIgnore` works ... the entirety of that implementation is shown here:

    class TokenMatchIgnore(TokenMatch):
        def matched(self, minfo, /):
            return minfo._replace(tokname=None)

Similarly, the entirety of `TokenMatchIgnoreButKeep` is:

    class TokenMatchIgnoreButKeep(TokenMatch):
        def __init__(self, tokname, regexp, *args, keep, **kwargs):
            super().__init__(tokname, regexp, *args, **kwargs)
            self.keep = keep

        def matched(self, minfo, /):
            if self.keep in minfo.value:
                return minfo._replace(value=self.keep)
            else:
                return minfo._replace(tokname=None)

which shows the recommended way to add more (keyword) arguments to the init function and use *args/**kwargs to protect against future signature revisions.

The subclass can also change `tokname` to something other than None, in which case that becomes the token type that will be instantiated. Refer back to the example of OCTAL_CONSTANT and `TokenMatchConvert`; here is the source code showing how that works:


    class TokenMatchConvert(TokenMatch):
        def __init__(self, *args, converter=int, alt_tokname=None, **kwargs):
            super().__init__(*args, **kwargs)
            self.converter = converter
            self.alt_tokname = alt_tokname

        def matched(self, minfo, /):
            return minfo._replace(
                value=self.converter(minfo.value),
                tokname=self.alt_tokname or minfo.tokname)

Note that if alt_tokname has been given it is used to replace the tokname (as written - for notational convenience - the above code always replaces the tokname field it just replaces it with itself if no alt_tokname was given).


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


One last caution: sometimes, rather than going hog-wild with modal rulesets, it may be simpler to implement a pre-processor on the input instead. For example, most C compilers work that way rather than trying to tokenize the C comment format in some modal way, though it appears to be reasonably possible with this mechanism (see the tokenizer test code `test_C` example).


## Tokenizer odds and ends

### Input pre-processing

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

### Using a different `Token`
There are several ways to make the `Tokenizer` produce a different object than the built-in `Token` definition. The easiest is to supply keyword argument `tokenfactory` to `Tokenizer`:

    class MyToken:
        def __init__(self, id, value, location, /):
            self.id = id               # TokenID Enum
            self.value = value         # from the TokenMatch
            self.location = location   # TokLoc info 

    from tokenizer import TokenMatch, Tokenizer

    rules = [
        TokenMatch('A', 'a'),
        TokenMatch('B', 'b')
    ]
    tkz = Tokenizer(rules, tokenfactory=MyToken)
    for t in tkz.string_to_tokens('bba'):
        print(t.__class__.__name__, t.id, repr(t.value))

Output:

    MyToken TokenID.B 'b'
    MyToken TokenID.B 'b'
    MyToken TokenID.A 'a'

The framework passes the token factory into `matched` method invocations (via attribute `factory` in the `_MatchedInfo` argument). Therefore another way to create an alternate token object would be to subclass `TokenMatch` (and its subclasses as necessary) and have them _replace the `factory` attribute as needed.

A more elaborate scheme would also have those TokenMatch subclasses return something other than the built-in _MatchedInfo object; the framework simply requires that object to have a `factory` attribute and will ultimately invoke it like this:
    
    # for illustrative purpose; some code details left out
    minfo = tm.matched(_MatchedInfo(tokname=tm.tokname,
                                    value=value,
                                    tokenizer=self,
                                    factory=self.tokenfactory))
    .. code elided that finds 'id' and creates 'loc' etc .. then:
    token = minfo.factory(id, minfo.value, loc)

so if the `TokenMatch` subclass `matched` method returns a custom object with at least a `factory` and `value` attribute, it can regain control of token creation via the `factory` invocation.
				    
Examples of all three of these ideas can be found in the unittest code.


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

