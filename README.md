# Import that!

This repository contains tutorials for writing bad Python code. Among other
things, you will learn some useless skills and bad habits:

- writing David Beazley style metaprogramming code that only **you** can understand
- violating [PEP 8][] and [PEP 20][]
- [*start* writing classes](https://www.youtube.com/watch?v=o9pEzgHorH0)
- and much more!

[PEP 8]: https://www.python.org/dev/peps/pep-0008/
[PEP 20]: https://www.python.org/dev/peps/pep-0020/

Here's an example from [the f-string tutorial](fun/fstrings.md):

```python
import inspect,collections,codecs,sys,tokenize,io
...
# pylint: https://www.youtube.com/watch?v=th4Czv1j3F8
assert( string.start[0] == string.end[0] == f.start[0] == f.end[0] ), "sorry, multi-line f-strings are not supported :("
code_lines[ f.start[0] ][ f.start[1]:string.end[1] ] = '__import__("importlib").import_module(%r).do_the_f(%s)'%( __name__ , string.string )
```

And here's an example of the metaprogramming, taken from [the implicit self
tutorial](fun/implicitself.md):

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

## Fun Tutorials
- [Implicit self](fun/implicitself.md)
- [Backwards Compatible f-strings](fun/fstrings.md)

## Boring explanation chapters
- [Encodings](boring/encodings.md)
- [How does Python run code?](boring/how-python-runs-code.md)
- [Stack Frames](boring/stackframes.md)

## See also
This repository is inspired by the work of other mad scientists :)
- [seriously.dontusethiscode.com](http://seriously.dontusethiscode.com/)
- [David Beazley's type checking framework](https://www.youtube.com/watch?v=js_0wjzuMfc)
