# Stack Frames

You have probably seen quite a few tracebacks. They contain information about
which functions ran when you got the error. Like this:

```python
>>> def thing(): raise ValueError
... 
>>> def stuff(): thing()
... 
>>> def thingie(): stuff()
... 
>>> thingie()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 1, in thingie
  File "<stdin>", line 1, in stuff
  File "<stdin>", line 1, in thing
ValueError
```

There's a module called [traceback][] that can print similar things.

```python
>>> import traceback,sys
>>> try: thingie()
... except: traceback.print_exception(*sys.exc_info())
...
Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
  File "<stdin>", line 2, in thingie
  File "<stdin>", line 2, in stuff
  File "<stdin>", line 2, in thing
ValueError
>>>
```

The interesting thing is that the whole module is actually written in plain
Python.

```python
>>> traceback
<module 'traceback' from '/home/akuli/.local/lib/python3.7/traceback.py'>
```

You can read the source code from the file on your computer. This chapter is all
about how it figures out what called what and prints information about it.

## The Stack

When Python is running `thing()`, each function [has been compiled into bytecode
and Python is running that like we all know](how-python-runs-code.md#bytecode),
but then whatever is needed for running that bytecode is stored in frames. They
are on a list-like thing known as a stack, like this:

    thingie()
    stuff()
    thing()

The stack also has a maximum size, and it's 1000 by default. So if we have more
than 1000 functions that call each other in a chain like this it doesn't work.
But it doesn't even need to be different functions; a function calling itself
will also cause trouble. If the stack gets full we get a `RuntimeError`.

The interesting thing is that we can inspect the [stack frames as Python
objects][frame objects] using [inspect.currentframe()][]. Of course, it's a
CPython implementation detail so we absolutely **must** use it everywhere we
can.

```python
>>> frame = inspect.currentframe()
>>> frame
<frame object at 0x7f1c59c455b8>
>>> dir(frame)
[..., 'f_back', 'f_builtins', 'f_code', 'f_globals', 'f_lasti', 'f_lineno', 'f_locals', 'f_trace']
>>> frame.f_code
<code object <module> at 0x7f1c59b5e5d0, file "<stdin>", line 1>
>>> frame.f_code.co_name
'<module>'
>>> frame.f_lineno
1
>>> frame.f_globals is globals()
True
>>> frame.f_back is None
True
```

If this frame was from a function we could also access the frame it was called
from with `frame.f_back`, the caller of that with `frame.f_back.f_back` and so
on. See [this section](how-python-runs-code#bytecode) if you haven't seen code
objects like `f_code` before.

This example code prints a minimal traceback:

```python
import inspect

def print_stack():
    frame = inspect.currentframe().f_back
    while frame is not None:
        print("file '%s', line %s, in %s" % (frame.f_code.co_filename, frame.f_lineno, frame.f_code.co_name))
        frame = frame.f_back

def thing(): print_stack()
def stuff(): thing()
def thingy(): stuff()
thingy()
```

Example output:

    file 'tbtest.py', line 9, in thing
    file 'tbtest.py', line 10, in stuff
    file 'tbtest.py', line 11, in thingy
    file 'tbtest.py', line 12, in <module>

[traceback]: https://docs.python.org/3/library/traceback.html
[inspect.currentframe()]: https://docs.python.org/3/library/inspect.html#inspect.currentframe
[frame objects]: https://docs.python.org/3/reference/datamodel.html#frame-objects
