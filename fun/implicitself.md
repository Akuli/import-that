# Implicit self

This chapter is inspired by [a tutorial on
seriously.dontusethiscode.com](http://seriously.dontusethiscode.com/2013/05/07/implicit-self-with-eval.html).

## The Problem

People coming from languages like Java have gotten used to writing code
roughly like this:

```java
public class Coord {
    public int x;
    public int y;

    public Coord(int x, int y) {
        this.x = x;
        this.y = y;
    }
}
```

Here's an equivalent Python class:

```python
class Coord:

    def __init__(self, x, y):
        #        ^^^^
        #    this sucks :(
        self.x = x
        self.y = y
```

The Java code is longer, but it would be nice if our Python methods wouldn't
require passing in a self parameter. The seriously.dontusethiscode.com tutorial
does that with [inspect.getsource()][], but like the author explains, it doesn't
work correctly in all cases.

## So... why not just edit the `locals()` dict?

Unfortunately there isn't a handy way to add variables inside a function after
it's defined. If you have used the [locals()][] function before you might be
thinking about using that:

```python
>>> def thing():
...     a = 1
...     b = 2
...     print(locals())
... 
>>> thing()
{'b': 2, 'a': 1}
>>> def thing():
...     locals()['self'] = 123   # makes a new dict, adds self to it, throws it away
...     print(self)      # NameError :(
... 
>>> thing()
Traceback (most recent call last):
  ...
NameError: name 'self' is not defined
```

Python remembers which local variables each function can have, and changing them
is not possible afterwards. That's why `locals()` made a new dict of variables
and returned that.

```python
>>> def thing():
...     a = 1
...     b = 2
... 
>>> thing.__code__.co_varnames
('a', 'b')
>>> thing.__code__.co_varnames = ('a', 'b', 'c')
Traceback (most recent call last):
  ...
AttributeError: readonly attribute
```

Global variables are stored in a dictionary instead, so a similar thing would
work just fine with [globals()][]:

```python
>>> globals()['hello'] = 'lel'
>>> hello
'lel'
```

## The Plan

We'll make something that turns this...

```python
implicit class Thing:
    def stuff(a, b):
        ...
```

...into this:

```python
class Thing:
    def stuff(self, a, b):
        ...
```

We can't use [the AST][ast] for this because trying to `ast.parse` code with
`implicit def` raises SyntaxError. We'll use [tokenize][] instead, and we'll
also use [a custom encoding][encodings] to make our thing as easy to use as
possible.

## Code Manipulation with Tokens

Here's the code that will handle our new `implicit` keyword. Save this to
`implicit.py`:

```python
import tokenize,io

def fix_code(code_bytes):
    tokens = tokenize.tokenize(io.BytesIO(code_bytes).readline)
    encoding_token = next(tokens)

    code_lines = str(code_bytes, encoding_token[1]).split('\n')
    code_lines.insert(0, None)      # token line numbers start at 1

    indent_level = 0 #  how many tabs (or 4-space chunks or whatever) currently indented
    implicit_level = None #where the implicit class started

    for token in tokens:
        if token[:2] == (tokenize.NAME, 'implicit'):
            classs = next(tokens)
            if classs[:2] != (tokenize.NAME, 'class'):
                # implicit is used as a variable name
                continue

            # we need to delete the implicit keyword like this:
            #
            #   implicit class Thing: ...
            #   ^^^^^^^^^
            # as you can see, from beginning of implicit to beginning of class
            (lineno, start), (lineno2, end) = token.start, classs.start
            assert lineno == lineno2, "'implicit' and 'class' on different lines"
            code_lines[lineno] = code_lines[lineno][:start] + code_lines[lineno][end:]

            if implicit_level==None:
                # start of a new class that is not inside another class
                implicit_level = indent_level

        if token[:2] == (tokenize.NAME, 'def') and implicit_level is not None:
            func_name = next(tokens); assert func_name[0] == tokenize.NAME
            paren = next(tokens); assert paren[:2] == (tokenize.OP, '(')
            line, column = paren.end
            code_lines[line] = code_lines[line][:column] + 'self,' + code_lines[line][column:]
        if token[0] == tokenize.INDENT:
            indent_level += 1
        if token[0] == tokenize.DEDENT:
            indent_level -= 1
            if indent_level == implicit_level:
                # end of class definition
                implicit_level = None

    return '\n'.join(map(str, code_lines[1:]))


if __name__ == '__main__': print(fix_code(b'''
implicit class Thing:
    def __init__(a, b):
        self.a = a
        self.b = b

    def stuff():
        pass

    # also works with async functions :D comment this out if your
    # python is too old for async functions
    async def async_stuff()
        pass

def normal():    # no self
    pass
'''))
```

If you have read [the boring tokenizing chapter][tokenize] you should have no
trouble understanding this.

Here's the output:

```python
class Thing:
    def __init__(self,a, b):
        self.a = a
        self.b = b

    def stuff(self,):
        pass

    # also works with async functions :D
    # comment this out if your python is too old for async functions
    async def async_stuff(self,)
        pass

def normal():    # no self
    pass
```

## The Encoding

Delete the `if __name__ == '__main__'` line and everything after it, and add
some [basic codec stuff][encodings]:

```python
import codecs

def decode(byteslike, errors='strict'):
    # the byteslike contains the encoding comment, we need to override with
    # another encoding comment that comes before it
    result = fix_code(b'# coding=utf-8\n' + bytes(byteslike), errors)
    return (result.split('\n', 1)[1], len(byteslike))

# the encoder could turn normal defs into implicit defs, but we'll just do
# nothing because normal defs also work with our thing
codec_info = codecs.CodecInfo(codecs.lookup('utf-8').encode, decode, name='implicit')
codecs.register({'implicit': codec_info}.get)
```

Now we just need a file that we can test this with. Here's `test.py`:

```python
# coding=implicit

implicit class Test:

    def __init__(target):
        self.target = target

    def hello():
        print("Hello %s!" % self.target)

    async def async_hello():
        print("Hello %s!" % self.target)
```

Let's try it out.

```python
>>> import implicit
>>> import test
>>> test.Test("World").hello()
Hello World!
>>> with __import__('contextlib').suppress(StopIteration):
...     test.Test("World").async_hello().send(None)
... 
Hello World!
```

## Exercises

-   Currently nested functions don't work in implicit classes:

    ```python
    implicit class Thing:
        def stuff():
            def inner():      # this also gets a self parameter :(
                print("hi")
            inner()     # error!
    ```

    Fix this. It's easier than you think it is.

-   Typically static methods are defined with the `@staticmethod` decorator.
    Implement Java-style syntax that work like this:

    ```python
    class Thing:
        static def stuff():
            print("hello")

    Thing.stuff()       # prints hello
    Thing().stuff()     # also prints hello
    ```

    Make sure that your implementation works if the user does
    `staticmethod = 'lol'`. somewhere

[inspect.getsource()]: https://docs.python.org/3/library/inspect.html#inspect.getsource
[locals()]: https://docs.python.org/3/library/functions.html#locals
[globals()]: https://docs.python.org/3/library/functions.html#globals

[ast]: ../boring/how-python-runs-code.md#ast
[tokenize]: ../boring/how-python-runs-code.md#tokenizing
[encodings]: ../boring/encodings.md#custom-encodings
