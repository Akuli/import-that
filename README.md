# Import that!

This repository contains tutorials for writing bad Python code. Among other
things, you will learn some useless skills and bad habits:

- writing David Beazley style metaprogramming code that only you can understand
- violating [PEP 8][] and [PEP 20][]
- [start writing classes](https://www.youtube.com/watch?v=o9pEzgHorH0)
- and much more!

[PEP 8]: https://www.python.org/dev/peps/pep-0008/
[PEP 20]: https://www.python.org/dev/peps/pep-0020/

Here's an example from [the implicitself chapter](fun/implicitself.md):

```python
import tokenize,io

# this is a really good idiom, use this everywhere you can
class MutableString(list):
    def __str__(self):
        return ''.join(self)

def fix_code(code_bytes, decoding_errors):
    code_lines = list(map(MutableString, str(code_bytes, 'utf-8', decoding_errors).split('\n')))
    ...
```

## Fun Tutorials
- [Implicit self](fun/implicitself.md)

## Boring explanation chapters
- [Encodings](boring/encodings.md)
- [How does Python run code?](boring/how-python-runs-code.md)

## See also
This repository is inspired by the work of other mad scientists :)
- [seriously.dontusethiscode.com](http://seriously.dontusethiscode.com/)
- [David Beazley's type checking framework](https://www.youtube.com/watch?v=js_0wjzuMfc)
