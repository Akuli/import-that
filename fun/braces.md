# Python with Braces

One of Python's most controversial features is the indent-based syntax. Usually
people coming from programming languages like C and Java are horrified by how
tiny indentation issues are syntax errors while most Python users like not
having to write curly braces everywhere.

Python isn't really about the programmer's choice to do things in different
ways. It's more about one, consistent way to do things. The `__future__` module
is usually used for enabling Python 3 features in Python 2, but it does this
too:

```python
>>> from __future__ import asd       # nothing special here
  File "<stdin>", line 1
SyntaxError: future feature asd is not defined
>>> from __future__ import braces    # guido hates braces?
  File "<stdin>", line 1
SyntaxError: not a chance
```

This quote comes from [here](http://forums.xkcd.com/viewtopic.php?t=22725):

> I have very little experience with python, but have coded in many other
> languages before. I would love to use python, but I really miss those braces,
> the thought of whitespace with semantic value horrifies me (after a brief
> stint with the esoteric language of Whitespace). Is there a way to code in
> python with braces?

Don't believe those discouraging naw-sayers. Of course there is!

Before writing this tutorial I did a quick search for other braced Python
implementations. The best thing I found was
[Bython](https://github.com/mathialo/bython), but Bython files can't be imported
in Python files and, more annoyingly, Python files can't be imported into
Bython. In this tutorial we'll write something better.

## Tokenize issues

We'll make something that turns this...

```python
def thing() {
    if (stuff) {
        things;
    }
}
```

...or this...

```python
    def
  thing() {
        if (stuff)
{
      things; }}
```

...into this:

```python
def thing():
    if stuff:
        things
```

Let's try to [tokenize](../boring/how-python-runs-code.md#tokenizing) some
example code and see what happens.

```python
>>> import tokenize,pprint,io
>>> code = b'''
...     def
...   thing() {
...         if (stuff) {
...       things; }}
... '''
>>> pprint.pprint(list(tokenize.tokenize(io.BytesIO(code).readline)))
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/akuli/.local/lib/python3.7/tokenize.py", line 571, in _tokenize
    ("<tokenize>", lnum, pos, line))
  File "<tokenize>", line 3
    thing() {
    ^
IndentationError: unindent does not match any outer indentation level
>>>
```

Oh crap, that didn't work. :/ Python is so picky about whitespace that even the
tokenizer emits `INDENT` and `DEDENT` tokens.

But that doesn't happen if everything is in parentheses, braces or square brackets:

```python
>>> (
...     "these",
...   "can",
...           "be", "indented"
...   , "however",
... "you", "want", )
('these', 'can', 'be', 'indented', 'however', 'you', 'want')
>>> code = b'(' + code + b')'
>>> pprint.pprint(list(tokenize.tokenize(io.BytesIO(code).readline)))
[lots of output, but no errors]
```

Let's implement the brace syntax :D

## The Code

Here's `braces.py`:

```python
import tokenize,io

# beautiful, idiomatic and overall really pythonic utility
# classes, use these everywhere you can
class MutableString(list):
     __str__ = lambda self: ''.join(self)
class Stack(list):
    push = list.append


class HandyDandyTokenIteratorThingyThing:
    # handy dandy init
    def __init__(self,tokens):
        self.iterator=( t for t in tokens if not t[0] in{ tokenize.NL, tokenize.NEWLINE, tokenize.COMMENT })
        self._coming_up = None

    # handy dandy property
    @property
    def coming_up(self):
         if self._coming_up is None:
           try: self._coming_up = next(self.iterator)
           except StopIteration: return None
         return self._coming_up

    # handy dandy iterator/sequence-ish kinda like interface u know
    def __iter__(self):
        return self
    def __next__(self):
        if self._coming_up is None:
            return next(self.iterator)
        result = self._coming_up
        self._coming_up = None
        return result
    def __bool__(self):
        return self.coming_up is not None

class CodeFixer:
    open2close=dict([ '()','{}','[]' ])
    close2open=dict(map( (lambda item: item[::-1] ), open2close.items()))

    def __init__(self, tokens):
        self.tokens = HandyDandyTokenIteratorThingyThing(tokens)
        (self.indent, self.output) = (0, MutableString())
        while self.tokens: self.do_token()

    def do_token(self):
        token = next(self.tokens)
        if token[:2] == ( tokenize.OP, ';' ):
            self.press_enter()
        else:
            self.output.extend(token[1] + ' ')
            if token[0] == tokenize.NAME and token[1] in ( 'if', 'elif', 'else', 'while', 'for', 'with', 'try', 'except', 'finally', 'class', 'def' ):
                if token[1] in ( 'if', 'elif', 'while', 'for', 'with', 'except' ):
                    # keyword (stuff) { ... }   ->   keyword stuff: ...
                    # note that parentheses around stuff are removed
                    paren = next(self.tokens)   # skip '('
                    assert paren[:2] == ( tokenize.OP, '(' ), "should be '(', not '%s'"%paren[ 1 ]
                    self.do_braces(')')
                    next(self.tokens)           # skip ')'
                if token[1] in ( 'class', 'def' ):
                    funcname = self.do_token()
                    import keyword
                    assert funcname[0]==tokenize.NAME and not( keyword.iskeyword(funcname[1]) ), "'%s' is not a valid function name" % funcname[1]
                    if self.tokens.coming_up[1] == '(':
                        # class Thing(...) { ... }
                        # def thing(...) { ... }
                        paren = self.do_token()     # include '('
                        assert paren[:2] == ( tokenize.OP, '(' ), "should be '(', not '%s'" % (paren[1])
                        self.do_braces(')')
                        self.do_token()             # include ')'
                    else:
                        # class Thing { ... }
                        # def thing { ... }     # ERROR
                        assert token[1] != 'def', "should be 'def %s(...)', not 'def %s%s...'"%( funcname[1], funcname[1], self.tokens.coming_up[1] )
                self.output.append(':')
                self.press_enter()
                brace = next(self.tokens)   # skip '{'
                assert brace[:2] == ( tokenize.OP, '{' ), "should be '{', not '%s'"%brace[ 1]
                self.press_tab()
                self.do_braces('}')
                next(self.tokens)           # skip '}'
                self.press_shift_tab()
        return token

    # processes tokens until closing_brace, leave the closing brace in self.tokens
    def do_braces(self, closing_brace):
        # contains '{', '[' and '(' characters only, pushed on '{[(', popped on '}])'
        braces = Stack([ self.close2open[closing_brace] ])

        while True:
            if self.tokens.coming_up[0] == tokenize.OP:
                if self.tokens.coming_up[1] in self.open2close:
                    braces.push( self.tokens.coming_up[1] )
                if self.tokens.coming_up[1] in self.close2open:
                    assert braces, "'%s' without '%s'"%( self.tokens.coming_up[1], self.close2open[ self.tokens.coming_up[1] ])
                    last = braces.pop()
                    assert self.open2close[last] == self.tokens.coming_up[1], "expected '%s', got '%s'"%( self.open2close[last], self.tokens.coming_up[1] )
                    if(not braces):
                        # we got all the closing braces we were looking for
                        return
            self.do_token()

        assert False, "there should be a '%s' before end of file" %closing_brace

    def press_tab(self):
        self.indent += 1
        self.output.append('\t')

    def press_shift_tab(self):
        assert self.indent >= 1, "too many '}' characters"
        self.indent -= 1
        assert self.output.pop() == '\t'

    def press_enter(self):
        self.output.extend('\n' + '\t'*self.indent)


def fix_code(code_bytes):
    tokens = list(tokenize.tokenize(io.BytesIO(b'(' + code_bytes + b')').readline))
    del tokens[:2]      # encoding and '('
    del tokens[-2:]     # ')' and endmarker
    print(CodeFixer(tokens).output)


fix_code(b'''
    def
  thing( ) {
        if (stuff) {
      things; }}
  def stuff() { pass; }      # we really need this pass, see exercises below

class Thing {
    def stuff(self) {
        for (letter in 'abc') { print(letter); }

        # this is as close to a c99/java/whatever for loop we can do right now
        # but also an example of how being able to indent freely can be useful
        i = 0; while (i < 3) { i+=1;
            print(i);
        }
    }

    def things(self) {
        try {
            1 / 0;
        } except (ZeroDivisionError as e) {
            print(e);
        }
    }
}
Thing().stuff();
Thing().things();
''')
```

Let's run it:

```
akuli@Akuli-Desktop:~$ python3 braces.py
def thing ( ) :
    if stuff :
        things 
def stuff ( ) :
    pass 
class Thing :
    def stuff ( self ) :
        for letter in 'abc' :
            print ( letter ) 
        i = 0 
        while i < 3 :
            i += 1 
            print ( i ) 
    def things ( self ) :
        try :
            1 / 0 
        except ZeroDivisionError as e :
            print ( e ) 
Thing ( ) . stuff ( ) 
Thing ( ) . things ( ) 

akuli@Akuli-Desktop:~$
```

That code violates PEP-8 even worse than our `braces.py`, but it seems to be
valid Python syntax. Let's run it and see what happens.

```
akuli@Akuli-Desktop:~$ python3 braces.py | python3
a
b
c
1
2
3
division by zero
akuli@Akuli-Desktop:~$
```

Great, it works!

If you fear braces for some reason, note that our code converts braces to colons
and indents, so if you need to work with someone's brace code you can convert it
to indent-based code first. You can also use autopep8:

```
akuli@Akuli-Desktop:~$ python3 braces.py | autopep8 --aggressive --experimental -
def thing():
    if stuff:
        things


def stuff():
    pass


class Thing:
    def stuff(self):
        for letter in 'abc':
            print(letter)
        i = 0
        while i < 3:
            i += 1
            print(i)

    def things(self):
        try:
            1 / 0
        except ZeroDivisionError as e:
            print(e)


Thing() . stuff()
Thing() . things()
```

## The Codec

Now we just need some [basic codec
stuff](../boring/encodings.md#custom-encodings) and we're done. Delete the
`fix_code` function and everything after it, and add this:

```python
import codecs,re

def encode(string, errors='strict'):
    raise NotImplementedError ("cannot create brace code from non-brace code :(")

def decode(byteslike, errors='strict'):
    code_bytes = bytes(byteslike)

    # there's no need to delete the encoding comment because we're adding a '(' to
    # the beginning, and only ASCII whitespace can occur before encoding comments
    # if you delete the '(', tokenize reuses the encoding comment and we have
    # infinite recursion :D
    tokens = list(tokenize.tokenize(io.BytesIO(b'(' + code_bytes + b')').readline))
    return (str(CodeFixer(tokens[2:-2]).output), len(code_bytes))

codec_info = codecs.CodecInfo(encode, decode, name='braces')
codecs.register({'braces': codec_info}.get)
```

Create a `test.py` like this:

```python
# coding=braces
class Thing {
    def __init__(self, target) {
        self.target = target;
    }
    def stuff(self) {
        print("Hello %s!" % self.target);
    }
    async def async_stuff(self) {
        print("Hello %s!" % self.target);
    }
}
```

Let's try it out.

```python
>>> import braces
>>> import test
>>> test.Thing("World").hello()
Hello World!
>>> with __import__('contextlib').suppress(StopIteration):
...     test.Thing("World").async_hello().send(None)
...
Hello World!
```

## Exercises

-   Currently `if (thing) { }` doesn't work and you need to do `if (thing) { pass; }`
    instead. Fix this. Make sure that `thing = { };` creates an empty dictionary.

**Note:** Rest of these exercises are all about adding new features. If dumping
everything into one class feels too complicated you can also go through the
token list in separate functions before using the `CodeFixer` class. It only
cares about the token types and strings, so you can create new tokens like this:

```python
>>> import tokenize
>>> tokenize.TokenInfo(tokenize.STRING, '"wolo"', None, None, None)
TokenInfo(type=3 (STRING), string='"wolo"', start=None, end=None, line=None)
```

-   Another common complaint about Python's syntax is that lambdas are limited
    to one line. That makes parsing the indentation based code a lot simpler.
    Add new function syntax so that this...

    ```python
    stuff.sort(key=def(item) {
        return item.thing;
    });
    ```

    ...is equivalent to this:

    ```python
    def __temp(item) {
        return item.thing;
    }
    stuff.sort(key=__temp);
    del __temp;
    ```

    Make sure that this doesn't conflict with other variables defined in the
    file (`__temp = 'lel'` works) and the function expressions can be nested
    (e.g. `def() { def() { ... } }`).

-   While you're at it, also add nicer lambdas. This...

    ```python
    stuff.sort(key = (item)=>item.thing()+"lol", reverse=True);
    ```

    ...should be equivalent to this:

    ```python
    stuff.sort(key = lambda item: item.thing()+"lol", reverse=True);
    ```

    The left side of `=>` is just like a `def` argument list. Anything that can
    be after `lambda` should also work after a `=>`.

    **Hint:** One problem with this is that the `lambda` keyword needs to be
    inserted before the argument list. When `CodeFixer.do_token()` receives a
    `(` token, it should do something like `self.last_paren = len(self.output)`
    before adding the `(` to `self.output`. Then you can do something like
    `self.output[self.last_paren:self.last_paren+1] = 'lambda'`.

-   Add traditional for loops. This...

    ```python
    for (INIT; COND; INCR) {
        ...
    }
    ```

    ...should be equivalent to this:

    ```python
    INIT;
    while (COND) {
        ...
        INCR;
    }
    ```

    The default values for `INIT`, `COND` and `INCR` should be `pass`, `True`
    and `pass`, respectively.

    Examples:

    ```python
    for (i=0; i<10; i+=1) { print(i); } # the classic :)
    for (i in range(10)) { print(i); }  # this must work too
    for (;;) { ... }                    # infinite loop
    for (;;;) { ... }                   # AssertionError: too many semicolons in for(...)
    for (;) { ... }                     # AssertionError: not enough semicolons in for(...)
    for (thing = ";;";;) { ... }        # OK because the semicolons are in a string
    ```
