# How does Python work?

In this chapter we'll learn roughly what Python does to run its code. Later
we'll use these things to change the code before it runs.

When we tell Python to run something like `print("hello")`, it runs the code
roughly like this:

    code -> tokens -> AST -> bytecode -> ??? -> profit!

## Tokenizing

```python
>>> import tokenize, pprint, io
>>> file = io.BytesIO(b"print('hello')")    # you can also open('somefile', 'rb')
>>> pprint.pprint(list(tokenize.tokenize(file.readline)))
[TokenInfo(type=59 (ENCODING), string='utf-8', ...),
 TokenInfo(type=1 (NAME), string='print', ...),
 TokenInfo(type=52 (OP), string='(', ...),
 TokenInfo(type=3 (STRING), string="'hello'", ...),
 TokenInfo(type=52 (OP), string=')', ...),
 ...]
>>>
```

As you can see, tokenizing means just splitting the code into `print`, `(`,
`'hello'` and `)`, as well as identifying roughly what each of these tokens is.

The first token is an ENCODING token that Python uses when reading the code from
a file. If you have read [my encoding tutorial](encodings.md), the ENCODING
token is how our `# coding=whatever` comments work.

Turning tokens back into code is not as easy you might think because spaces
aren't turned into tokens at all and you need to handle indenting and dedenting
yourself. It's usually easier to modify the code based on the tokens.

## AST

```python
>>> import ast
>>> print(ast.dump(ast.parse("print('hello')")))
Module(body=[Expr(value=Call(func=Name(id='print', ctx=Load()), args=[Str(s='hello')], keywords=[], starargs=None, kwargs=None))])
```

It's kind of hard to see what's going on. If we open that up a bit, it looks
like this:

    Module(
        body=[
            Expr(
                value=Call(
                    func=Name(id='print', ctx=Load()),
                    args=[Str(s='hello')],
                    keywords=[],
                    starargs=None,
                    kwargs=None
                )
            )
        ]
    )

At this point we know that we have are calling (`Call`) a function that comes
from a variable (`Name`) called `print`, and we're giving it the string
`'hello'` as an argument.

Note that `ast.parse` takes the code as a string, but internally it tokenizes it
first. It doesn't jump directly from code to AST.

Going from AST back to tokens or code is difficult, and usually it's best to
avoid it. The AST isn't really useless though because the nodes contain
information about where they are defined:

```python
>>> the_call = ast.parse("print('hello')").body[0].value
>>> (the_call.lineno, the_call.col_offset)    # on line 1, the very beginning
(1, 0)
```

It's also possible to compile an AST tree and execute it:

```python
>>> the_ast = ast.parse("print('hello')")
>>> compile(the_ast, 'whatever', 'exec')
<code object <module> at 0x7fdc11f25e40, file "whatever", line 1>
>>> exec(compile(the_ast, 'whatever', 'exec'))
hello
```

There's more information about code objects below.

## Bytecode

Next the AST is turned into something even more machine-readable and less
human-friendly. It's not executed directly because it would be kind of complicated, and thus
slow. Let's have a look at the bytecode:

```python
>>> def thing():
...     print("hello")
... 
>>> thing.__code__
<code object thing at 0x7fef8a16a660, file "<stdin>", line 1>
>>> dir(thing.__code__)
[... 'co_argcount', 'co_cellvars', 'co_code', 'co_consts', 'co_filename',
 'co_firstlineno', 'co_flags', 'co_freevars', 'co_kwonlyargcount', 'co_lnotab',
 'co_name', 'co_names', 'co_nlocals', 'co_stacksize', 'co_varnames']
>>> thing.__code__.co_code
b't\x00d\x01\x83\x01\x01\x00d\x00S\x00'
>>> thing.__code__.co_consts
(None, 'hello')
```

There's also a module for printing a scary-looking table of what the `__code__` does:

```python
>>> import dis
>>> dis.dis(thing.__code__)
  2           0 LOAD_GLOBAL              0 (print)
              2 LOAD_CONST               1 ('hello')
              4 CALL_FUNCTION            1
              6 POP_TOP
              8 LOAD_CONST               0 (None)
             10 RETURN_VALUE
```

No, you can't `import dat` even though you can `import this` and `import dis`.
[lolcode][] is better than python.

I am not aware of anyone modifying the `__code__` of a function to make it
behave differently. It would probably be difficult, and it's possible to do many
scary things without touching the `__code__`. But [the dis docs][] say that the
whole bytecode stuff is a CPython implementation detail, so it might be fun to
make something that relies on it some day.

It's also possible to create new code objects:

```python
>>> import types
>>> types.CodeType
<class 'code'>
>>> isinstance(thing.__code__, types.CodeType)
True
>>> help(types.CodeType)
Help on class code in module builtins:

class code(object)
 |  code(argcount, kwonlyargcount, nlocals, stacksize, flags, codestring,
 |        constants, names, varnames, filename, name, firstlineno,
 |        lnotab[, freevars[, cellvars]])
 |
 |  Create a code object.  Not for the faint of heart.
```

[lolcode]: https://en.wikipedia.org/wiki/LOLCODE
[the dis docs]: https://docs.python.org/3/library/dis.html
