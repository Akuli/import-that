# Backwards Compatible f-strings

**Note:** This tutorial is kind of a failure :( It's way more complicated than
it needs to be, and [similar but better
things](https://github.com/syrusakbary/interpy) existed before f-strings were
added to Python. You can still read this tutorial if you are interested in my
stupid approach to it. Thanks (I guess?) to
[Summertime](https://github.com/Summertime/) for pointing this out.

You need Python 3.5 or **older** if you want to try out the example code
in this tutorial.

In new Pythons it's possible to conveniently add code in strings like this:

```python
>>> import sys;sys.version
'3.7.0a0 (heads/master-dirty:f2ffae1, Jul  1 2017, 12:46:13) \n[GCC 4.8.4]'
>>> target = "World"
>>> print(f"Hello {target}!")
Hello World!
>>> print(f"Hello {1 + 2}!")
Hello 3!
>>> print(f"Hello {input('Enter something: ')}!")
Enter something: world
Hello world!
```

Unfortunately that's invalid syntax on Python 3.5 and older, and most things
aren't using 3.6 yet.

```python
>>> import sys;sys.version
'3.5.3 (default, Jul 17 2017, 22:01:40) \n[GCC 4.8.4]'
>>> f"Hello {1 + 2}"
  File "<stdin>", line 1
    f"Hello {1 + 2}"
                   ^
SyntaxError: invalid syntax
```

The good news is that we can still [tokenize][] f-string code and add the
f-string functionality, so most of our code will be a lot like in [the implicit
self tutorial](implicitself.md). I highly recommend reading that first if you
want to continue reading this tutorial as many things will be based on it.

```python
>>> import tokenize,io,pprint
>>> pprint.pprint(list(tokenize.tokenize(io.BytesIO(b'f"Hello {1 + 2}"').readline)))
[TokenInfo(type=59 (ENCODING), string='utf-8', ...),
 TokenInfo(type=1 (NAME), string='f', ...),
 TokenInfo(type=3 (STRING), string='"Hello {1 + 2}"', ...),
 TokenInfo(type=0 (ENDMARKER), string='', ...)]
```

## Stack Frames

This `get_vars` function is based on things in the [stack frame tutorial][], so
you can read that if you want to learn more about it. `get_vars()` returns a
dict-like object of all global and local variables in whatever called it. We're
using a [ChainMap][] due to Python 3.5 compatibility.

```python
>>> import inspect,collections,pprint
>>> def get_vars():
...     caller_frame = inspect.currentframe().f_back
...     return collections.ChainMap(caller_frame.f_locals, caller_frame.f_globals)
... 
>>> def thing():
...     a, b = 1, 2
...     pprint.pprint(dict(get_vars()))
... 
>>> thing()
{...
 'a': 1,
 'b': 2,
 'collections': <module 'collections' from '.../collections/__init__.py'>,
 'get_vars': <function get_vars at 0x7f92efda2e18>,
 'inspect': <module 'inspect' from '.../inspect.py'>,
 'pprint': <module 'pprint' from '.../pprint.py'>,
 'thing': <function thing at 0x7f92ede9b950>}
```

Note that this does *not* work correctly in nested functions:

```python
>>> def thing():
...     a = 1
...     def stuff():
...         print('a' in get_vars())
...     stuff()
... 
>>> thing()
False
```

The problem was that there were no variables in `stuff()` so `f_locals` was
`{}`, but `a` wasn't in `f_globals` either because it was still local to
`thing()`. Most of the time this doesn't matter too much, so we won't fix it.

It's also possible to do this:

```python
>>> def thing():
...     a = 1
...     def stuff():
...         a           # now it's in f_locals :D
...         print('a' in get_vars())
...     stuff()
... 
>>> thing()
True
```

## Importing stuff

Our `fstrings.py` file will define a `do_the_f` function that fetches variables like
`get_vars()` and formats a string with them. A simple way to call it would be to
add `import fstrings` at the top of the file when decoding, and then replace all
occurances of `f'something'` with `that.do_the_f('something')`. But everything
screws up if someone does `that = 'lol'`.

We could `import fstrings as __that` or something, but there's a better solution.
[The built-in `__import__` function][__import__] imports and returns a module:

```python
>>> __import__('sys')
<module 'sys' (built-in)>
```

Nowadays it's recommended to use [importlib.import_module][] instead:

```python
>>> import importlib
>>> importlib.import_module('sys')
<module 'sys' (built-in)>
```

We could just insert `import importlib` to the top of the code when decoding,
but then the user could do `importlib = 'lol'`. In order to follow good
practices as much as possible while keeping our code robust and reliable, we end
up with this really *elegant* and **beautiful** code:

```python
>>> __import__('importlib').import_module('sys')
<module 'sys' (built-in)>
```

It's still possible to set `__import__` to something weird, but it doesn't
really matter because there's already an `__import__` variable and we aren't
bloating the namespace. It's also `__magic__`, so most people who don't read
these tutorials wouldn't even think about setting it to something.

Importing the same thing multiple times doesn't actually load the module
multiple times. It's cached in [sys.modules][]:

```python
>>> open('lol.py','w').write("print('LOADING LOL')")
20
>>> import lol
LOADING LOL
>>> import lol              # cached, doesn't print 'LOADING LOL'
>>> __import__('lol')       # cached, doesn't print, returns the module object
<module 'lol' from '/tmp/lol.py'>
>>> __import__('importlib').import_module('lol')   # same as __import__
<module 'lol' from '/tmp/lol.py'>
>>> del __import__('sys').modules['lol']
>>> import lol              # not cached anymore :D
LOADING LOL
```

This is useful because our `fstrings.py` can output code that imports `that`, but
it's loaded only once.

The `__name__` variable is also set to the module name such that
`importlib.import_module(__name__)` is the current module. We'll use that too.

## The Code

Here's `fstrings.py`:

```python
import inspect,collections,codecs,sys,tokenize,io

def do_the_f(string):
    frame = inspect.currentframe().f_back
    return string.format_map(collections.ChainMap(frame.f_locals, frame.f_globals))

class MutableString(list):
    def __str__(self): return ''.join(self)

def fix_code(code_bytes, decoding_errors):
    code_lines = list(map(MutableString, str(code_bytes, 'utf-8', decoding_errors).split('\n')))
    code_lines.insert(0, None)
    tokens = tokenize.tokenize(io.BytesIO(code_bytes).readline)
    next(tokens)

    for f in tokens:
        if f[:2] == (tokenize.NAME, 'f'):
            string = next(tokens)
            if string[0] != tokenize.STRING:
                continue

            # pylint: https://www.youtube.com/watch?v=th4Czv1j3F8
            assert( string.start[0] == string.end[0] == f.start[0] == f.end[0] ), "sorry, multi-line f-strings are not supported :("
            code_lines[ f.start[0] ][ f.start[1]:string.end[1] ] = '__import__("importlib").import_module(%r).do_the_f(%s)'%( __name__ , string.string )

    return '\n'.join(map(str, code_lines[1:]))


if sys.version_info < (3, 6):
    def decode(byteslike, errors='strict'):
        result = fix_code(b'# coding=utf-8\n' + bytes(byteslike), errors)
        return (result.split('\n', 1)[1], len(byteslike))
    codec_info = codecs.CodecInfo(codecs.lookup('utf-8').encode, decode, name='f')
    codecs.register({'f': codec_info}.get)
else:
    codecs.register({'f': codecs.lookup('utf-8')}.get)
```

If you have read [my implicit self tutorial](implicitself.md) you should have no
trouble understanding this code. This code also introduces a new pylint
directive that can be useful when linters hate your beautiful code. It works
just as well with any other linter too.

Here's `test.py`:

```python
# coding=f
target = "World"
print(f"Hello {target}!")
```

Let's try it out.

```python
>>> import fstrings
>>> import test
Hello World!
```

# Arbitary Code

Currently things like `f'{1 + 2}'` don't work at all because our `do_the_f`
function just formats variables. You probably knew already that `thing[stuff]`
is roughly equivalent to `thing.__getitem__(stuff)`. It turns out that *any*
object that supports `__getitem__` can be used with `format_map`:

```python
>>> class Thing:
...     def __getitem__(self, key):
...         return 123
... 
>>> 'the number is {key}'.format_map(Thing())
'the number is 123'
```

Remove the old `do_the_f` function, and replace it with this code:

```python
class FDict:
    def __init__(self, globalz, localz): (self.globalz,self.localz)=(globalz,localz)
    def __getitem__(self, code): return eval(code,self.globalz,self.localz)

def do_the_f(string):
    frame = inspect.currentframe().f_back
    return string.format_map(FDict(frame.f_globals, frame.f_locals))
```

Note that we're using `globalz` and `localz` as argument names because `globals`
and `locals` are names of built-in functions, and it's good style to avoid using
their names.

Update `test.py`:

```python
# coding=f
print(f"Hello {input('Enter something -->  ')}!")
```

Let's try it out.

```python
>>> import fstrings
>>> import test
Enter something -->  blah
Hello blah!
```

## Colon Problems

Our f-strings must not contain `:` characters because they already have a
special meaning in string formatting.

```python
>>> from math import*; '{:.2f}'.format(pi)
'3.14'
```

That's why we need this instead:

```python
# coding=f
print(f"Hello {input('Enter something\\x3a ')}!")
```

Save that code to `test.py`.

Here `"...\\x3a..."` is a string that contains the characters `\`, `x`, `3` and
`a`. When the string is evaluated, it turns into a colon because its [Unicode][]
number (ordinal) is `3a` in hexadecimal:

```python
>>> ord(':')
58
>>> '%x' % 58   # %x means hexadecimal
'3a'
>>> '\x3a'      # \x means hexadecimal
':'
```

Annoyingly, the `\\x3a` thing doesn't work with Python 3.6 f-strings:

```python
>>> import sys;sys.version
'3.7.0a0 (heads/master-dirty:f2ffae1, Jul  1 2017, 12:46:13) \n[GCC 4.8.4]'
>>> import fstrings        # our f encoding is just like utf-8 on this python
>>> import test
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/akuli/import-that/fstrings/test.py", line 2
    print(f"Hello {input('Enter something\\x3a ')}!")
         ^
SyntaxError: f-string expression part cannot include a backslash
```

## Error Handling

There's a weird problem in our code. With a `test.py` like this...

```python
# coding=f
f'{1/0}'
```

...we get this:

```python
>>> import fstrings
>>> import test
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/akuli/import-that/fstrings/test.py", line 2, in <module>
    f'{1/0}'
  File "/home/akuli/import-that/fstrings/fstrings.py", line 9, in do_the_f
    return string.format_map(FDict(frame.f_globals, frame.f_locals))
KeyError: '1/0'
```

We were supposed to get a `ZeroDivisionError` at least *somewhere*, but
something swallowed it completely.

Turns out that this is a bug in CPython. The `format_map` method silences all
exceptions raised in `__getitem__`, and raises `KeyError` instead:

```python
>>> class Thing:
...     def __getitem__(self, key):
...         1/0
... 
>>> Thing()['lel']
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 3, in __getitem__
ZeroDivisionError: division by zero
>>> '{lel}'.format_map(Thing())         # ??? guido hates us
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'lel'
```

Of course, there's no problem in [PyPy](https://pypy.org/).

```python
>>>> class Thing:
....     def __getitem__(self, key):
....         1/0
....         
>>>> '{lel}'.format_map(Thing())
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 3, in __getitem__
ZeroDivisionError: division by zero
>>>> 
```

If this will be fixed some day it won't help us because we're doing this whole
thing just for backwards compatibility.

Fortunately we're already calling `format_map` from our own `do_the_f` function,
so now we just need to catch the real error in `FDict.__getitem__`, bring that
to `do_the_f` and replace the stupid `KeyError` with that. Modify `FDict` and
`do_the_f` like this:

```python
class FDict:
    def __init__(self, globalz, localz): (self.globalz,self.localz)=(globalz,localz)
    def __getitem__(self, code):
        try:
            # the same eval() we had last time would work, but this way the
            # error message says that the code came from '<f-string>'
            codeobject = compile(code, '<f-string>', 'eval')
            return eval(codeobject,self.globalz,self.localz)
        except Exception as e:
            self.error = e

            # this error is ignored and turned into another KeyError, but we're
            # raising KeyError here because of pypy compatibility
            raise KeyError

def do_the_f(string):
    frame = inspect.currentframe().f_back
    fdict = FDict(frame.f_globals, frame.f_locals)
    try: return string.format_map(fdict)
    except KeyError:    # this can be the stupid KeyError or the one we raised
        raise fdict.error from None
```

Also modify the codec stuff to make it work correctly in tracebacks:

```python
utf8 = codecs.lookup('utf-8')
if sys.version_info < (3,6):
    def decode(byteslike, errors='strict'):
        result = fix_code(b'# coding=utf-8\n' + bytes(byteslike), errors)
        return (result.split('\n', 1)[1], len(byteslike))

    # python's tracebacks use incrementaldecoder when reading
    # files, so we want that to be utf-8; otherwise we get
    # __import__("importlib").import_module('that').do_the_f
    # in the traceback
    codec_info = codecs.CodecInfo(utf8.encode, decode, incrementaldecoder=utf8.incrementaldecoder, name='f')
    codecs.register({'f': codec_info}.get)
else:
    codecs.register({'f': utf8}.get)
```

Let's see how it works.

```python
>>> import fstrings
>>> import test
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/akuli/import-that/fstrings/test.py", line 2, in <module>
    f'{1/0}'
  File "/home/akuli/import-that/fstrings/fstrings.py", line 20, in do_the_f
    except KeyError: raise fdict.error from None
  File "/home/akuli/import-that/fstrings/fstrings.py", line 11, in __getitem__
    return eval(codeobject,self.globalz,self.localz)
  File "<f-string>", line 1, in <module>
ZeroDivisionError: division by zero
```

We still have some lines from `fstrings.py` in our traceback, but this is much
better than the KeyError we had before.

[tokenize]: ../boring/how-python-runs-code.md#tokenizing
[Unicode]: ../boring/encodings.md
[stack frame tutorial]: ../boring/stackframes.md

[ChainMap]: https://docs.python.org/3/library/collections.html#collections.ChainMap
[__import__]: https://docs.python.org/3/library/functions.html#__import__
[importlib.import_module]: https://docs.python.org/3/library/importlib.html#importlib.import_module
[sys.modules]: https://docs.python.org/3/library/sys.html#sys.modules
