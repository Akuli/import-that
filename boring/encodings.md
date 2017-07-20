# Encodings

We can only save 8-bit bytes to files. Each bit can only be in 2 different
states; it's either on or off, often represented with 1 or 0. So 8 of them gives
us `2**8 = 256` different possible combinations of bytes, where each combination
can be represented as a number in `range(256)`.

Half of that (128 characters) is enough for representing a minimal set of
English characters, a few whitespace characters like `\n` and some punctuation.
The ASCII character set is just that:

```python
>>> bytes(range(128)).decode('ASCII')
'\x00\x01\x02\x03\x04\x05\x06\x07\x08\t\n\x0b\x0c\r\x0e\x0f\x10\x11\x12\x13\x14
\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f !"#$%&\'()*+,-./0123456789:;<=>?@A
BCDEFGHIJKLMNOPQRSTUVWXYZ[\\]^_`abcdefghijklmnopqrstuvwxyz{|}~\x7f'
```

The problem is that this is only a subset of American English, and not anywhere
near enough for most things. It's missing lots of characters like ö or ß.

At first, a common solution was to use the rest of our 256 combinations for
other characters. The problem with this is that different characters would need
to be added for different languages, and as a result, the *same* number of
`range(128, 256)` often meant *different* characters in different
character sets.

Nowadays [the Unicode standard][] assigns exactly one number to many characters,
but most of these numbers are not in `range(256)` so we can't represent each
character with one byte. It's still possible to save these characters to files
with encodings like `utf-8` that simply use more than 1 byte for each character
when needed.

Unicode also contains some other fun symbols. Run `print('\N{PILE OF POO}')` to
get started.

[the Unicode standard]: http://www.unicode.org/standard/WhatIsUnicode.html

## Encodings in Python

Python 2 uses ASCII as its default encoding everywhere, and sometimes it makes
weird errors that can be really hard to find and fix.

```python
>>> u'ß' + 'a'       # Unicode ß + ASCII a
u'\xe4a'
>>> 'ß' + u'a'       # ß encoded with the terminal's encoding + Unicode a
Traceback (most recent call last):
  ...
UnicodeDecodeError: 'ascii' codec can't decode bla bla bla
```

Pretty much everything is Unicode in Python 3, so you don't need to worry about
this stuff if you use Python 3.

Python 2 also uses ASCII when reading files that will imported or executed. So
if you have `print 'öß'` in a file that you are trying to run in Python 2, you
get an error like this:

      File "thing.py", line 1
    SyntaxError: Non-ASCII character '\xc3' in file thing.py on line 1, but no
    encoding declared; see http://www.python.org/peps/pep-0263.html for details

The link contains some details about changing this encoding. If you add a
comment like `# coding=utf-8` at the top of the file, it runs. If your terminal
is using UTF-8 too, the code prints `öß` correctly; otherwise you need to do
this:

```python
# coding=utf-8
import sys

if sys.stdout.encoding is None:
    # our program is piped to a file, print UTF-8 as is
    print 'öß'
else:
    # let's turn our UTF-8 string into a unicode object and then encode that
    # back to str with whatever encoding the terminal uses
    print 'öß'.decode('utf-8').encode(sys.stdout.encoding)
```

This is one of the reasons why I like Python 3. I can just `print('öß')` and it
works.

## Custom encodings

Comments like `# coding=utf-8` are also supported in Python 3. In both Pythons
it's also possible to create our *own* encodings and use them in encoding
comments. Let's make an encoding that is otherwise like UTF-8, but converts
UPPERCASE to lowercase and lowercase to UPPERCASE.

Here's `swapcase.py`:

```python
import codecs, re
utf8 = codecs.lookup('utf-8')

# str -> bytes
# usually this isn't called at all when importing or tokenizing something
def encode(string, errors='strict'):
    return utf8.encode(string.swapcase(), errors)

# kinda bytes-ish -> (str, how many bytes were decoded)
# this is called when a file with this encoding is imported (see below)
def decode(byteslike, errors='strict'):
    string, bytes_used = utf8.decode(byteslike, errors)
    return (string.swapcase(), bytes_used)

codec_info = codecs.CodecInfo(encode, decode, name='swapcase')
codecs.register({'swapcase': codec_info}.get)
```

Not everything works with our codec because we didn't implement
[incrementalencoder and other stuff like
that](https://docs.python.org/3/library/codecs.html#codecs.CodecInfo),
but most of the time this is good enough.

Here's `test.py`:

```python
# coding=swapcase
DEF HELLO():
    PRINT("hELLO wORLD!")
```

Let's try it out.

```python
>>> import swapcase
>>> b'hELLO wORLD'.decode('swapcase')
'Hello World'
>>> import test     # needs to be imported after swapcase
>>> test.hello()
Hello World!
```

## Cache Issues

If you import everything as above, then add a print to the `decode()` function
in `swapcase.py` and import everything again, your print doesn't actually run.
The problem is that Python cached our `test` module to a folder named
`__pycache__` *after* decoding it, and it didn't notice that we changed
`swapcase.py`. This wouldn't have happened if we had changed `test.py` because
Python's encodings aren't meant to be changed as aften as we change them.

So, if you have weird codec issues and your encoding doesn't even run, delete
the `__pycache__` folder and everything will work. This one-liner is an easy way
to do that from within Python:

```python
>>> __import__('shutil').rmtree('__pycache__', ignore_errors=True)
```
