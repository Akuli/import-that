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
# beautiful, idiomatic and overall really pythonic utility
# classes, use these everywhere you can
class MutableString(list):
     __str__ = lambda self: ''.join(self)
class Stack(list):
    push = list.append

class HandyDandyTokenIteratorThingyThing:

        ...

            if token[0] == tokenize.NAME and token[1] in ( 'if', 'elif', 'else', 'while', 'for', 'with', 'try', 'except', 'finally', 'class', 'def' ):
                    ...
                    funcname = self.do_token()
                    import keyword
                    assert funcname[0]==tokenize.NAME and not( keyword.iskeyword(funcname[1]) ), "'%s' is not a valid function name" %( funcname[ 1] )
```

And here's an example of the metaprogramming, taken from the same tutorial.
**This file can be imported** after importing another file that does the magic.
You can also import any other modules from files like this and in general do
pretty much everything that you can in normal Python files.

```python
# coding=braces
class Thing {
    def __init__(self, target) {
        self.target = target;
    }
    def hello(self) {
        print("Hello %s!" % self.target);
    }
    async def async_hello(self) {
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
