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
implicit def thing():
    stuff
```

...into this:

```python
def thing(self):
    stuff
```

We can't use [the AST][ast] for this because trying to `ast.parse` code with
`implicit def` raises SyntaxError. We'll use [tokenize][] instead, and we'll
also use [a custom encoding][encodings] to make our thing as easy to use as
possible.

## Code Manipulation with Tokens

Here's the code that will handle our new `implicit` keyword. Let's write this
code to `that.py`:

```python
import tokenize,io

# this is a really good idiom, use this everywhere you can
class MutableString(list):
    def __str__(self):
        return ''.join(self)

def fix_code(code_bytes):
    code_lines = list(map(MutableString, str(code_bytes, 'utf-8').split('\n')))
    code_lines.insert(0, None)      # token line numbers start at 1

    tokens = tokenize.tokenize(io.BytesIO(code_bytes).readline)
    next(tokens)    # skip encoding token

    for implicit in tokens:
        if implicit[:2] == (tokenize.NAME, 'implicit'):
            the_def = next(tokens)
            if the_def[:2] != (tokenize.NAME, 'def'):
                # implicit is used as a variable name because it's not before a def
                continue

            func_name = next(tokens); assert func_name[0] == tokenize.NAME
            paren = next(tokens); assert paren[:2] == (tokenize.OP, '(')

            # self must be inserted first because deleting stuff before it
            # changes the indexes
            line, column = paren.end
            code_lines[line].insert(column, 'self,')

            # we need to delete 'implicit' and the space after it to avoid
            # screwing up indentation
            #
            #     implicit def stuff(): ...
            #     ^^^^^^^^^
            #
            # as you can see, from beginning of implicit to beginning of def
            assert implicit.start[0] == the_def.start[0]   # must be on the same line
            code_lines[ the_def.start[0] ][ implicit.start[1] : the_def.start[1] ] = ''

    return '\n'.join(map(str, code_lines[1:]))


if __name__ == '__main__': print(fix_code(b'''
class Thing:
    implicit def __init__(a, b):
        self.a = a
        self.b = b

    # also works with async functions :D
    # comment this out if your python is too old for async functions
    async implicit def stuff():
        pass

    def normal(self):
        print("hi")
'''))
```

There's not much magic going on in this code. If you read [the boring tokenizing
chapter][tokenize] you should have no trouble understanding this.

## The Encoding

Next we'll add a convenient way to use the `implicit` keyword in our code.
Modify your `fix_code` in `that.py` to also accept a new `decoding_errors`
argument:

```python
def fix_code(code_bytes, decoding_errors):
    code_lines = list(map(MutableString, str(code_bytes, 'utf-8', decoding_errors).split('\n')))
    # rest of this function doesn't need to be changed
```

Then delete the `if __name__ == '__main__'` thing and add some [basic codec
stuff][encodings]:

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

class Thing:

    implicit def __init__(name):
        self.name = name

    implicit def hello():
        print("Hello %s!" % self.name)

    async implicit def async_hello():
        print("Hello %s!" % self.name)
```

Let's try it out.

```python
>>> import that
>>> import test
>>> test.Thing("World").hello()
Hello World!
>>> import contextlib
>>> with contextlib.suppress(StopIteration):
...     test.Thing("World").async_hello().send(None)
... 
Hello World!
```

## Exercises

1.  `implicit def thing()` is annoying and repetitive to type. Implement an
    `implicit class` syntax.

    Instead of this...

    ```python
    # coding=implicit
    class Thing:
        implicit def __init__():
            ...
        implicit def stuff():
            ...
    ```

    ...it should be possible to do this:

    ```python
    # coding=implicit
    implicit class Thing:
        def __init__():
            ...
        def stuff():
            ...
    ```

    You don't need to support `implicit def` functions, implicit classes are
    enough.

2.  Typically static methods are defined with the `@staticmethod` decorator.
    Implement Java-style syntax that work like this:

    ```python
    class Thing:
        static def stuff():
            print("hello")

    Thing.stuff()       # prints hello
    Thing().stuff()     # also prints hello
    ```

    Your implementation doesn't need to be compatible with the `implicit`
    keyword. It doesn't matter if adding `staticmethod = 'lol'` before the class
    definition screws things up.

[inspect.getsource()]: https://docs.python.org/3/library/inspect.html#inspect.getsource
[locals()]: https://docs.python.org/3/library/functions.html#locals
[globals()]: https://docs.python.org/3/library/functions.html#globals

[ast]: ../boring/how-python-runs-code.md#ast
[tokenize]: ../boring/how-python-runs-code.md#tokenizing
[encodings]: ../boring/encodings.md#custom-encodings
