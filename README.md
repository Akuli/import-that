# Import that!

This repository contains tutorials for writing bad Python code. Among other
things, you will learn some useless skills and bad habits:

- writing David Beazley style metaprogramming code that only **you** can understand
- violating [PEP 8][] and [PEP 20][]
- [*start* writing classes](https://www.youtube.com/watch?v=o9pEzgHorH0)
- and much more!

[PEP 8]: https://www.python.org/dev/peps/pep-0008/
[PEP 20]: https://www.python.org/dev/peps/pep-0020/

Here's an example from [the brace tutorial](fun/braces.md):

```python
import tokenize,io

# beautiful, idiomatic and overall really pythonic utility
# classes, use these everywhere you can
class MutableString(list):
     __str__ = lambda self: ''.join(self)
class Stack(list):
    push = list.append


class _HandyDandyTokenIteratorThingyThing:
    # handy dandy init
    def __init__(self,tokens):
        self.iterator=( t for t in tokens if not t[0] in{ tokenize.NL, tokenize.NEWLINE, tokenize.COMMENT })
        self._coming_up = None

    # handy dandy property
    @property
    def coming_up(self):
        ...
```

And here's an example of the metaprogramming, taken from the same tutorial.
**This file can be imported** after importing another file that does the magic.

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

## Fun Tutorials
- [Python with Braces](fun/braces.md)
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
