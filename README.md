# tokenizer
A simple `Tokenizer` class inspired by the python [re](https://docs.python.org/3/library/re.html) module example, with enhancements.

# Quick Start
A working example:

    from tokenizer import TokenMatch, TokenMatchInt, TokenRules, Tokenizer

    rules = TokenRules([
        TokenMatch('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', TokenMatch.ID_ASCII),    # see below
        TokenMatchInt('CONSTANT', r'-?[0-9]+'),
    ])

    input_strings = ["abc123 3750   def",
                     "xyzzy"]

    tkz = Tokenizer(rules, input_strings)
    for token in tkz.tokens():
        print(f"ID = {token.id:20s} VALUE = {token.value!r}")

This will output:

    ID = TokenID.IDENTIFIER   VALUE = 'abc123'
    ID = TokenID.WHITESPACE   VALUE = ' '
    ID = TokenID.CONSTANT     VALUE = 3750
    ID = TokenID.WHITESPACE   VALUE = '   '
    ID = TokenID.IDENTIFIER   VALUE = 'def'
    ID = TokenID.IDENTIFIER   VALUE = 'xyzzy'

Notes:
- For convenience, `TokenMatch` defines some regular expression constants such as `TokenMatch.ID_ASCII` (`r'[A-Za-z_][A-Za-z_0-9]*'`). See [Unicode](#unicode).
- The second argument to `Tokenizer` is an iterable of strings (not a "naked" string). An open file can be used because it acts like an iterable of strings.
- The `value` attribute of each token is a string when the match came from a `TokenMatch`. It is an integer when it came from `TokenMatchInt`. There are many other `TokenMatch` variations available.
- The token `id` attribute is an Enum value, a `TokenID` Enum which was created automatically from the names given in all the `TokenMatch` objects supplied. The `TokenID` Enum is available as `rules.TokenID` (i.e., in the `TokenRules` object)

This is enough to get started; the rest of this document dives into details.

## TokenMatch Details

`TokenMatch` objects specify token names, processing rules, and corresponding regular expressions. Subclasses of `TokenMatch` automate typical conversion actions, such as converting a string of digit characters into a native integer - as shown above in the Quick Start.

The `tokenizer` module provides:
 - TokenMatch
 - TokenIDOnly
 - TokenMatchIgnore
 - TokenMatchIgnoreButKeep
 - TokenMatchInt
 - TokenMatchConvert
 - TokenMatchKeyword

### TokenMatch
The base class. All other variations are subclassed off this. Whatever matches the regexp is put into the token `value` as a string.

### TokenIDOnly

`TokenIDOnly` allows an application to create additional `TokenID` Enum entries that do not have a corresponding regular expression. This is useful for special tokens that might be inserted manually at upper levels of processing.

To see this, consider this contrived example:

    from tokenizer import TokenIDOnly, TokenMatch, Tokenizer, TokenRules
    rules = TokenRules([
        TokenIDOnly('FOO'),
        TokenIDOnly('BAR'),
        TokenIDOnly('BAZ'),
        TokenMatch('BANG', r'!')
    ])

    print(list(rules.TokenID))

This will output

    [<TokenID.FOO: 1>, <TokenID.BAR: 2>, <TokenID.BAZ: 3>, <TokenID.BANG: 4>]

TokenIDs FOO, BAR, and BAZ will never be produced by a `Tokenizer` built from this set of rules, but are available for use by the application however it needs.

### TokenMatchIgnore

 Another `TokenMatch` subclass, `TokenMatchIgnore` matches the regular expression but discards the token. This is especially useful for white space processing.

Here is the same example from Quick Start but using `TokenMatchIgnore` for white space:


    from tokenizer import TokenMatch, TokenMatchInt, TokenMatchIgnore
    from tokenizer import TokenRules, Tokenizer

    rules = TokenRules([
        TokenMatchIgnore('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatchInt('CONSTANT', r'-?[0-9]+'),
    ])

    input_strings = ["abc123 3750   def",
                     "xyzzy"]

    tkz = Tokenizer(rules, input_strings)
    for token in tkz.tokens():
        print(f"ID = {token.id:20s} VALUE = {token.value!r}")

This will output:

    ID = TokenID.IDENTIFIER   VALUE = 'abc123'
    ID = TokenID.CONSTANT     VALUE = 3750
    ID = TokenID.IDENTIFIER   VALUE = 'def'
    ID = TokenID.IDENTIFIER   VALUE = 'xyzzy'

### TokenMatchIgnoreButKeep

In some cases whitespace can be suppressed but newlines have semantic significance and should be preserved. `TokenMatchIgnoreButKeep` is made for this:

    from tokenizer import TokenMatch, Tokenizer, TokenRules
    from tokenizer import TokenMatchIgnoreButKeep

    rules = TokenRules([
        TokenMatchIgnoreButKeep('NEWLINE', r'\s+', keep='\n'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*')
    ])
    tkz = Tokenizer(rules, ["foo   bar  \n   \n  baz\n"])
    for token in tkz.tokens():
        print(token.id, repr(token.value))

output:

    TokenID.IDENTIFIER 'foo'
    TokenID.IDENTIFIER 'bar'
    TokenID.NEWLINE '\n'
    TokenID.IDENTIFIER 'baz'
    TokenID.NEWLINE '\n'

In this example any whitespace (matching `r'\s+'`) will be ignored, UNLESS it contains one (or more) `keep` characters ('\n' in this example). If the `keep` character appears in the match then one token (NEWLINE in this example) is generated (with a `value` of just one `keep` regardless of how many were present). If different behavior is desired, it is easy enough to write a different customized subclass.

### TokenMatchInt

Converts the value attribute of a token to an integer, and is most-obviously useful for something like a CONSTANT as shown in the Quick Start.

### TokenMatchConvert

This is a generalization of `TokenMatchInt` and allows for arbitrary conversions.

Suppose constants can be simple decimal numbers OR numbers in python octal format. The `TokenMatchConvert` subclass takes an argument, `converter`, that will allow for this:

    from tokenizer import TokenMatch, TokenRules, Tokenizer, TokenMatchIgnore
    from tokenizer import TokenMatchConvert, TokenMatchInt
    import functools

    octal = functools.partial(int, base=8)

    rules = TokenRules([
        TokenMatchConvert('CONSTANT', r'0o([0-7]+)', converter=octal),
        TokenMatchInt('CONSTANT', r'-?[0-9]+'),
        TokenMatchIgnore('WHITESPACE', r'\s+'),
    ])
    tkz = Tokenizer(rules, ["42 0o377"])
    for token in tkz.tokens():
        print(token.id, repr(token.value))

Output:

    TokenID.CONSTANT 42
    TokenID.CONSTANT 255

This example shows another feature - the same token name ('CONSTANT' in this example) can appear in more than one TokenMatch. Sometimes that's advantageous as a way to simplify the individual regular expressions rather than having a single, multi-clause, regular expression for two different formats. As this example shows it also makes it possible to have per-format processing details (e.g., integer conversion or octal conversion, depending on which expression matched).

### TokenMatchKeyword

The `TokenMatchKeyword` subclass provides a simple way to make keywords be their own unique tokens:

    from tokenizer import TokenMatch, Tokenizer, TokenRules
    from tokenizer import TokenMatchKeyword, TokenMatchIgnore

    rules = TokenRules([
        TokenMatchKeyword('if'),
        TokenMatch('IDENTIFIER', TokenMatch.ID_ASCII),
        TokenMatchIgnore('WHITESPACE', r'\s+'),
    ])

    tkz = Tokenizer(rules, ["if iff biff"])
    for token in tkz.tokens():
        print(token.id, repr(token.value))

`TokenMatchKeywod` will automatically create a regular expression to capture exactly the word given, and will automatically create a token name by upper-casing the keyword.

Output from the above code will be:

    TokenID.IF 'if'
    TokenID.IDENTIFIER 'iff'
    TokenID.IDENTIFIER 'biff'

Compare this to the naive:

    # same code as above but with these rules:
    TokenMatch('IF', 'if'),
    TokenMatch('IDENTIFIER', TokenMatch.ID_ASCII),
    TokenMatchIgnore('WHITESPACE', r'\s+'),

which will output:

    TokenID.IF 'if'
    TokenID.IF 'if'
    TokenID.IDENTIFIER 'f'
    TokenID.IDENTIFIER 'biff'

With `TokenMatchKeyword` keep in mind that [order in the rules matters](#ordering); if a generic IDENTIFIER style match occurs before a `TokenMatchKeyword`, it will match first, which is likely not desired:

    # same code but with these rules
    TokenMatch('IDENTIFIER', TokenMatch.ID_ASCII),
    TokenMatchKeyword('if'),
    TokenMatchIgnore('WHITESPACE', r'\s+'),

output:

    TokenID.IDENTIFIER 'if'
    TokenID.IDENTIFIER 'iff'
    TokenID.IDENTIFIER 'biff'


### Subclassing TokenMatch
It is fairly straightforward to subclass `TokenMatch` to cover other special circumstances; see [Advanced Topics](#advanced) for details.

## TokenRules

The `TokenRules` object collects multiple `TokenMatch` objects together and automatically creates a `TokenID` Enum from all the names present.

For example:

    from tokenizer import TokenMatch, TokenMatchIgnore, TokenMatchInt
    from tokenizer import TokenRules

    rules = TokenRules(
          [
              TokenMatch('VARIABLE', r'[A-Za-z][A-Za-z0-9]*'),
              TokenMatchIgnore('WHITESPACE', r'\s+'),
              TokenMatchInt('CONSTANT', r'-?[0-9]+'),
          ]
    )
    print(rules.TokenID)
    print(list(rules.TokenID))

This will output:

    <enum 'TokenID'>
    [<TokenID.VARIABLE: 1>, <TokenID.WHITESPACE: 2>, <TokenID.CONSTANT: 3>]


A parser interpreting the stream of tokens will need this `TokenID` to understand the stream of tokens coming from the `Tokenizer`. The `TokenID` can be found in the `TokenRules` object as shown above. The `TokenRules` object is also available as `.rules` in a `Tokenizer` object, as this full example shows:

    from tokenizer import TokenMatch, TokenMatchInt, TokenMatchIgnore
    from tokenizer import TokenRules, Tokenizer

    input_strings = ["abc123 3750   def",
                     "xyzzy"]
    tkz = Tokenizer(TokenRules([
                       TokenMatch('VARIABLE', r'[A-Za-z][A-Za-z0-9]*'),
                       TokenMatchIgnore('WHITESPACE', r'\s+'),
                       TokenMatchInt('CONSTANT', r'-?[0-9]+'),
                   ]), input_strings)
    print(list(tkz.rules.TokenID))

which will print:

    [<TokenID.VARIABLE: 1>, <TokenID.WHITESPACE: 2>, <TokenID.CONSTANT: 3>]

<a name=ordering></a>
### TokenMatch Order in a TokenRule

Note that the ordering of `TokenMatch` objects supplied to `TokenRules` becomes the order in which the regular expressions are examined when looking for a match. The ordering can easily affect the match results. For example:

    from tokenizer import Tokenizer, TokenMatch, TokenRules

    rules = TokenRules([
        TokenMatch('ONE', r'1'),
        TokenMatch('TWO', r'2'),
        TokenMatch('ANY', r'.')
        ])

    tkz = Tokenizer(rules, ["123"])

    for tok in tkz.tokens():
        print(tok.id)
        
This will output:

    TokenID.ONE
    TokenID.TWO
    TokenID.ANY

But:

    from tokenizer import Tokenizer, TokenMatch, TokenRules

    rules = TokenRules([
        TokenMatch('ANY', r'.'),
        TokenMatch('ONE', r'1'),
        TokenMatch('TWO', r'2')
        ])

    tkz = Tokenizer(rules, ["123"])

    for tok in tkz.tokens():
        print(tok.id)
        
This outputs:

    TokenID.ANY
    TokenID.ANY
    TokenID.ANY

because the '.' comes first and matches every individual character so the 'ONE' and 'TWO' rules never fire. Pay attention to ordering accordingly.


### Multiple Named Rule Sets
Using `NamedRuleSet` objects it is possible to supply named groups of `TokenMatch` objects and switch between them on certain conditions. 

This is covered in [Advanced Topics](#advanced).

## Tokenizer
The `Tokenizer` object has already been introduced through the examples, but additional capabilities are discussed here.

## Input variations
The examples used above have all provided the input as an iterable of strings supplied to the `Tokenizer` creation. Note that this form works especially well for input files:

    from tokenizer import TokenMatch, TokenMatchInt, TokenRules, Tokenizer

    rules = TokenRules([
        TokenMatch('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatchInt('CONSTANT', r'-?[0-9]+'),
    ])

    # The file "example-input" should contain two lines:
    #       abc123 3750   def
    #       xyzzy

    with open('example-input', 'r') as f:
        tkz = Tokenizer(rules, f)
	for token in tkz.tokens():
	    print(f"ID = {token.id:20s} VALUE = {token.value!r}")


This will produce almost the same output as the Quick Start example, except that now there are newline characters between the lines and they show:

    ID = TokenID.IDENTIFIER   VALUE = 'abc123'
    ID = TokenID.WHITESPACE   VALUE = ' '
    ID = TokenID.CONSTANT     VALUE = 3750
    ID = TokenID.WHITESPACE   VALUE = '   '
    ID = TokenID.IDENTIFIER   VALUE = 'def'
    ID = TokenID.WHITESPACE   VALUE = '\n'
    ID = TokenID.IDENTIFIER   VALUE = 'xyzzy'
    ID = TokenID.WHITESPACE   VALUE = '\n'

Instead of tying the object creation and input together, there are two other ways to supply the input later. One is to supply it to the `tokens` method:


    from tokenizer import TokenMatch, TokenMatchInt, TokenRules, Tokenizer

    rules = TokenRules([
        TokenMatch('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatchInt('CONSTANT', r'-?[0-9]+'),
    ])

    # NOTE: No input specified here
    tkz = Tokenizer(rules)

    # The file "example-input" should contain two lines:
    #       abc123 3750   def
    #       xyzzy
    with open('example-input', 'r') as f:
	for token in tkz.tokens(f):
	    print(f"ID = {token.id:20s} VALUE = {token.value!r}")

This produces output identical to the previous example. This also works with an iterable of strings provided instead of a file.

Finally, there is a method for explicitly working on one string:


    from tokenizer import TokenMatch, TokenMatchInt, TokenRules, Tokenizer

    rules = TokenRules([
        TokenMatch('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatchInt('CONSTANT', r'-?[0-9]+'),
    ])

    # NOTE: No input specified here
    tkz = Tokenizer(rules)

    for token in tkz.string_to_tokens('this is a string'):
        print(f"ID = {token.id:20s} VALUE = {token.value!r}")

which, somewhat obviously, produces:

    ID = TokenID.IDENTIFIER   VALUE = 'this'
    ID = TokenID.WHITESPACE   VALUE = ' '
    ID = TokenID.IDENTIFIER   VALUE = 'is'
    ID = TokenID.WHITESPACE   VALUE = ' '
    ID = TokenID.IDENTIFIER   VALUE = 'a'
    ID = TokenID.WHITESPACE   VALUE = ' '
    ID = TokenID.IDENTIFIER   VALUE = 'string'

which method of providing input is best is entirely application specific.



<a name="advanced"></a>
# Advanced Topics

## Writing custom TokenMatch subclasses

Writing custom TokenMatch classes is straightforward; create a subclass that overrides `__init__` (if needed, e.g., for extra arguments) and `action`:

    def __init__(self, tokname, regexp, /):

    def action(self, ta, /):
        INPUTS: ta - A (mutable) TokenAction object containing
	        data useful for constructing the token
       RETURNS: A token, or None (to discard this match)

If the `action` for a particular `TokenMatch` subclass will always return None, it suffices to set the `action` attribute itself to None. This, in fact, is how the TokenMatchIgnore subclass is implemented:

    class TokenMatchIgnore(TokenMatch):
        action = None

The simplest subclasses are just conversions from matched value string to something else. To do this a subclass can access and alter the `ta` object, which is a TokenAction object declared like this:

    @dataclass
    class TokenAction:
        value: typing.Any
        location: TokLoc
        tkz: Tokenizer
        token_id: Enum
        token_cls: typing.Callable   # usually this is Token (the class)

The `TokenAction` class has one method, `maketoken` that will take all the data in the `TokenAction` object and return a token:

    def maketoken(self):
        return self.token_cls(self.token_id, self.value, self.location)

Subclasses can alter those attributes as necessary to persuade `maketoken` to do what they want, or, of course, can simply construct the Token object themselves. Whatever works best for the application is fine.

Returning to the example of a value conversion, here is one way to write `TokenMatchInt`:

    # NOTE: The real implementation of TokenMatchInt is done as
    #       a special case of TokenMatchConvert, which is more general.
    #
    class TokenMatchIntExample(TokenMatch):
        def action(self, ta, /):
            ta.value = int(ta.value)
            return super().action(ta)

This converts the `ta.value` attribute, converting the string (it always starts out as a string) into a python integer. It then lets the superclass create the token using the mutated `ta` object.


If the subclass needs more arguments it can override `__init__` as needed and stash the arguments in the TokenMatch object for use in the subsequent `action` call. Here is an absurd example:

    from tokenizer import TokenMatch, TokenRules, Tokenizer


    class TokenMatchPrintSomething(TokenMatch):
        def __init__(self, *args, msg, **kwargs):
            super().__init__(*args, **kwargs)
            self.msg = msg

        def action(self, ta, /):
            print(self.msg)
            return super().action(ta)


    rules = TokenRules(
          [
              TokenMatchPrintSomething('A', r'a', msg="foo-A"),
              TokenMatchPrintSomething('B', r'b', msg="foo-B"),
          ]
    )

    tkz = Tokenizer(rules)

    for token in tkz.string_to_tokens('aaba'):
        print(f"ID = {token.id:20s} VALUE = {token.value!r}")

When run this will output:

    foo-A
    ID = TokenID.A            VALUE = 'a'
    foo-A
    ID = TokenID.A            VALUE = 'a'
    foo-B
    ID = TokenID.B            VALUE = 'b'
    foo-A
    ID = TokenID.A            VALUE = 'a'

Perusing the implementations of the various TokenMatch subclasses already provided, plus also looking at some of the unittest code, is the best way to get a more detailed understanding for writing complicated TokenMatch subclasses.

<a name=unicode></a>
### Unicode conveniences
The IDENTIFIER TokenMatch used in all the examples, whether using ID_ASCII or the explicit regular expression, won't accept Unicode characters.
For example:

    from tokenizer import TokenMatch, Tokenizer, TokenRules

    rules = TokenRules([
        TokenMatch('WHITESPACE', r'\s+'),
        TokenMatch('IDENTIFIER', r'[A-Za-z_][A-Za-z_0-9]*'),
        TokenMatch('CONSTANT', r'-?[0-9]+'),
    ])

    tkz = Tokenizer(rules)
    for token in tkz.string_to_tokens("MötleyCrüe rulz"):
        print(token.id, repr(token.value))


will recognize the first M as an IDENTIFIER by itself:

    TokenID.IDENTIFIER 'M'

and then raise an exception because the `ö` character does not match any rule.

As a convenience, the TokenMatch class defines some ready-made regular expressions for typical identifier patterns.

For Unicode identifiers that are permitted to start with ANY Unicode "alphabetic" character or underscore, followed by any number of such characters but then also including digits (just not in the first character):

    TokenMatch.ID_UNICODE               # = r'[^\W\d]\w*'

If underscores are not allowed:

    TokenMatch.ID_UNICODE_NO_UNDER:    # = r'[^\W\d_][^\W_]*'

The equivalent ASCII-only patterns are fairly obvious, but are also provided as class variables for symmetry:

    TokenMatch.ID_ASCII                 # = r'[A-Za-z_][A-Za-z_0-9]*'
    TokenMatch.ID_ASCII_NO_UNDER        # = r'[A-Za-z][A-Za-z0-9]*'

Thus, for example:

    TokenMatch('IDENTIFIER', TokenMatch.ID_UNICODE)

will accept identifiers like:

    MötleyCrüe

whereas

    TokenMatch('IDENTIFIER', TokenMatch.ID_ASCII)

will not.

## Overriding the `Token` type

By default a `Tokenizer` generates `Token` objects:

    @dataclass(frozen=True)
    class Token:
        id: Enum                # The TokenID Enum created automatically
        value: typing.Any       # typically string but could be int or others
        location: TokLoc        # source stream info for error reporting


This `Token` class is defined in the `tokenizer` module, and is referenced by the `Tokenizer` class using a class variable `Tokenizer.Token` (which defaults to the `Token` class shown above).

If, for example, an application requires a mutable `Token` object (or requires any other additional features in a `Token`), it can define its own `Token` and subclass `Tokenizer` like this:

    from dataclasses import dataclass
    from tokenizer import Tokenizer, TokenMatch, TokenRules, TokLoc
    from enum import Enum
    import typing

    @dataclass             # note: no frozen=True
    class MutableToken: 
        id: Enum
        value: typing.Any
        location: TokLoc

    # Override the Token type that will be produced by Tokenizer
    class MyTokenizer(Tokenizer):
        Token = MutableToken

    rules = TokenRules([
        TokenMatch('A', 'a'),
        TokenMatch('B', 'b')
    ])

    tkz = MyTokenizer(rules)
    for token in tkz.string_to_tokens("aabab"):
        print(token.id, repr(token.value))
	# this would fail with the built-in immutable Token class
        token.foo = 'bar'

Note that the `Tokenizer` framework expects the signature for creating a token object to be the three arguments as defined in the built-in `Token` class; if other arguments are needed they will have to be supplied by partial or other magic:

    from tokenizer import Tokenizer, TokenMatch, TokenRules, Token
    from functools import partial

    class ClownToken(Token):
        def __init__(self, id, value, loc, /, clown):
            super().__init__(id, value, loc)
            self.clown = clown

    class MyTokenizer(Tokenizer):
        Token = partial(ClownToken, clown='bozo')

    rules = TokenRules([
        TokenMatch('A', 'a'),
        TokenMatch('B', 'b')
    ])

    tkz = MyTokenizer(rules)
    for token in tkz.string_to_tokens("aabab"):
        print(token.id, repr(token.value), token.clown)

This will output:

    TokenID.A 'a' bozo
    TokenID.A 'a' bozo
    TokenID.B 'b' bozo
    TokenID.A 'a' bozo
    TokenID.B 'b' bozo


## Multiple TokenMatch rulesets

Some lexical processing is modal - the appearance of a token will change the lexical processing for the following tokens. To allow for simple versions of this, rules can be grouped into multiple `NamedRuleSet` objects and selected based on tokens.

For example:

    from tokenizer import TokenMatch, Tokenizer, TokenRules, NamedRuleSet
    from tokenizer import TokenMatchRuleSwitch

    r1 =[TokenMatch('ZEE', r'z'),
         TokenMatchRuleSwitch('ALTRULES', r'/', rulename='ALT')]

    r2 = [TokenMatch('ZED', r'z'),
          TokenMatchRuleSwitch('MAINRULES', r'/', rulename=None)]

    ns1 = NamedRuleSet(rules=r1, name=None)
    ns2 = NamedRuleSet(rules=r2, name='ALT')

    rules = TokenRules(ns1, ns2)

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

In this example sometimes 'z' is a ZEE, and sometimes a ZED, depending on which ruleset has been activated. The `TokenMatchRuleSwitch` subclass takes a `rulename` argument and switches the active ruleset accordingly.

If the primary ruleset (the first argument to `TokenRules`) is a naked list of `TokenMatch` objects as has been the case in most of the examples, it is converted to a `NamedRuleSet` with a name of None. Or, said differently, by convention the primary rule set has a name of None. This can be changed though by explicitly specifying a `NamedRuleSet` with a name even for the first (primary) rule set.

In cases where it makes semantic sense for a `TokenRuleSwitch` to simply switch to the "next" rule set, applications can use `TokenRuleSwitch.NEXTRULE` in place of the actual `NamedRuleSet` name for the next rule. Whether this makes things easier or more obscure is a stylistic choice by the application writer.

One last caution: sometimes, rather than going hog-wild with modal rulesets, it may be simpler to implement a pre-processor on the input instead. For example, most C compilers work that way (preprocessor phase) rather than trying to tokenize the C comment format in some modal way (or some hyper-clever non-modal way).


## Tokenizer odds and ends

### Input pre-processing

Input can be pre-processed, because anything that duck-types as an iterable of strings is acceptable. For example, if backslash-newline sequences need to be elided (in effect combining two adjacent lines), that's easy to do. A built-in filter, `linefilter' does this:

    from tokenizer import TokenMatch, TokenMatchIgnore, Tokenizer, TokenRules

    rules = TokenRules([
        TokenMatch('IDENTIFIER', TokenMatch.ID_UNICODE),
        TokenMatchIgnore('WHITESPACE', r'\s+'),
    ])

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

### Returning a different TokenID in an action() method

Suppose an application wants all positive integers between from 0 to 255 (inclusive) to be called INT8 tokens, values from 256 to 65535 (inclusive) to be called INT16 tokens, and any other kind of integer to just be an INT.

Conceptually the INT8 cases could be handled this way, ignoring the INT16 cases:

    from tokenizer import Tokenizer, TokenMatchInt, TokenRules
    from tokenizer import TokenMatch, TokenMatchIgnore, TokenIDOnly

    rules = TokenRules([
        TokenMatchInt('INT8', '0'),
        TokenMatchInt('INT8', '1'),

	# 253 more TokenMatchInt patterns go here

        TokenMatchInt('INT8', '255'),
        TokenMatchInt('INT', r'-?[0-9]+'),
        TokenMatchIgnore('WHITESPACE', r'\s+'),
    ])

    input_strings = ["0 1 255 500 1000"]
    tkz = Tokenizer(rules, input_strings)
    for token in tkz.tokens():
        print(f"ID = {token.id:20s} VALUE = {token.value!r}")

This will print:

    ID = TokenID.INT8         VALUE = 0
    ID = TokenID.INT8         VALUE = 1
    ID = TokenID.INT8         VALUE = 255
    ID = TokenID.INT          VALUE = 500
    ID = TokenID.INT8         VALUE = 1            # see commentary below
    ID = TokenID.INT8         VALUE = 0
    ID = TokenID.INT8         VALUE = 0
    ID = TokenID.INT8         VALUE = 0

What happened here? The digits '1' and '0' (in the `1000`) matched the INT8 expressions and got interpreted as INT8 tokens. Despite that flaw, this could be made to work, but the regexps will get complicated and there would still have to be 256 TokenMatchInt entries for every possibly INT8 value. This would be even worse for the INT16 pattern.

Instead a TokenMatch subclass can create Tokens with different TokenID values based on internal logic, and mutate the `TokenAction` object accordingly.

Thus, for example:


    class TokenMatchSizedInt(TokenMatchInt):
        def action(self, ta):
            ta.value = int(ta.value)      # convert from string

            if ta.value >= 0 and ta.value < 256:
                ta.token_id = 'INT8'
            elif ta.value >= 256 and ta.value < 65536:
                ta.token_id = 'INT16'

            return super().action(ta)

The `token_id` attribute is normally the Enum associated with the token defined by this `TokenMatch`. The `action` method may override that however, and set it to some other value of the Enum or - for convenience - set it to a string (`INT8` or `INT16` in this case). The framework will convert that to the appropriate `TokenID` Enum as necessary.

With this subclass, the rules will look like this:

    rules = TokenRules([
        TokenMatchSizedInt('INT', r'-?[0-9]+'),
        TokenMatchIgnore('WHITESPACE', r'\s+'),
        TokenIDOnly('INT8'),
        TokenIDOnly('INT16'),
    ])

and if one extra number, "8675309" is added o the input string, the output will 
look like this:

    ID = TokenID.INT8         VALUE = 0
    ID = TokenID.INT8         VALUE = 1
    ID = TokenID.INT8         VALUE = 255
    ID = TokenID.INT16        VALUE = 500
    ID = TokenID.INT16        VALUE = 1000
    ID = TokenID.INT          VALUE = 8675309

Note that `TokenIDOnly` is needed to introduce the INT8 and INT16 TokenID values. It does not matter where they appear in the order, but they must be there so they are part of the automatically generated TokenID Enum.

<a name="tkenhance"></a>
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
