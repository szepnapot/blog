---
title: "Python decorators"
date: 2019-01-20T11:41:54+01:00
description: "Introduction to decorators though examples"
categories: [ "python", "metaprogramming", "programming" ]
tags: ["python", "decorators", "metaprogramming", "functions", "closures"]
---

# Python decorators


### About functions

I'm sure you've already heard something like "Python's functions are first-class objects" or "Everything is an object in Python".
What exactly a first-class object and how this is important regarding decorators?

Functions are **first-class objects**, this simply means that you can *assign them to variables*, *store them in data structures*, *pass them as arguments* to other functions or even *return them as values* from functions.

This concept is the base of some advanced language features in Python, like *lambdas, decorators, metaprogramming*.

These concepts can take a while before they sink in, thats not a problem.    It takes time, but rather rewarding once you've get the "aha" moment.
For demonstration purposes we use this simple `greet` function.

```python
def greet(name):
    return f"Hello {str.capitalize(name)} :)"

print(greet("pEtER"))

# outputs: "Hello Peter :)"
```

Things like strings, lists, modules, functions, classes are **all objects**.
The `greet` function is an object so we can assign it to another variable.

```python
welcome = greet
```

Note that this assignment **doesn't** **call** the function, just points to it (reference).

```python
print(welcome("peter"))

# outputs: "Hello Peter :)"
```

[top](#TableOfContents)

___

### We can also store functions in data structures.

```python
import base64

functions = [greet,
			 welcome,
			 lambda s: base64.b64encode(b'f{s}'),
			 str.lower]
print(functions)

# outputs:
# [<function greet at 0x103385e18>,
# <function greet at 0x103385e18>,
# <function <lambda> at 0x102bab378>,
# <method 'lower' of 'str' objects>]

```

We can do whatever we want with the objects inside the list.
Let's loop through and call them.

```python
for func in functions:
    print(func, func("test"))

# outputs:
# <function greet at 0x101db1e18> Hello Test :)
# <function greet at 0x101db1e18> Hello Test :)
# <function <lambda> at 0x1033ac378> b'ZntzfQ=='
# <method 'lower' of 'str' objects> test
```

Or call them directly.

```python
print(functions[2]("Hello"))
print(functions[-1]("Peter"))

# outputs:
# b'ZntzfQ=='
# peter
```

[top](#TableOfContents)

___

### Functions can be passed to other functions as is

```python
def about(func):
	return func("my name is Peter")

print(about(greet))

# outputs:
# "Hello My name is peter :)"
```

Of course you can make other functions which generate a different output. This allows to abstract away and pass around **behavior** in programs.

Functions that can accept other functions as arguments are also called **higher-order functions**. They are the base **building blocks of functional programming**.

Note that Python misses a few key functional thing like *tail recursion*, *automatic currying*, *lazy lists*, but the built in `map` function is a great example to show what is a higher-order function.

```python
print(list(map(about, functions)))

# outputs:
# ['Hello My name is peter :)',
#  'Hello My name is peter :)',
#  b'ZntzfQ==',
#  'my name is peter']
```

[top](#TableOfContents)

___

### Functions can be defined inside functions

Often called nested/inner functions. (*funception*)

```python
def log(text):
    def prettify(t):
        return f"[*] - {t}"
    return prettify(text)

print(log('hy'))

# outputs: "[*] - hy"
```

The **inner function is enclosed**, we cannot access it from outside.

```python
log.prettify
# AttributeError: 'function' object has no attribute 'prettify'
```

Take a closer look with the `dis` module.

```python
import dis

codeobj = log.__code__
print(dis.dis(codeobj))

# outputs:
# 21           0 LOAD_CONST               1 (<code object prettify at 0x103b60e40, ...)
#              2 LOAD_CONST               2 ('log.<locals>.prettify')
#              4 MAKE_FUNCTION            0
#              6 STORE_FAST               1 (prettify)
#
# 23           8 LOAD_FAST                1 (prettify)
#             10 LOAD_FAST                0 (text)
#             12 CALL_FUNCTION            1
#             14 RETURN_VALUE
# None
```

I hope that you can see that this is a pretty awesome stuff.
Function can not only **accept** **behaviors** via arguments but they can also **return behaviors**.

[top](#TableOfContents)

___

### Closures

So functions can contain inner functions, even return them. This enables us for example to hide details or capture local state.

```python
def make_sound(text, volume):
    def yell():
        return f"{text.upper()} !!!"

    def normal():
        return f"{str.capitalize(text)}."

    def whisper():
        return f"{text.lower()}..."

    if volume > 0.5:
        return yell()
    elif volume < 0.5:
        return whisper()
    return normal()
```

Notice that the **inner functions didn't have parameters**, yet they are still able to access the parent's parameter. Functions like this are called closures.
A closure remembers the values from its *enclosing lexical scope*, even when the program flow is no longer in that scope. In practice this means that **functions** not only can return behavior but they **are also able to pre-configure those behaviors.**


Here is a simple example where a closure might be more preferable than defining a class and making objects.

```python
def make_multiplier_of(n):
    def multiplier(x):
        return x * n
    return multiplier

times5 = make_multiplier_of(5)
print(times5(4))

# outputs: 20
# another way to write this

times5and4 = make_multiplier_of(5)(4)
print(times5and4)
# outputs: 20
```

We can still find our inner function, just use a dunder called `__closure__`.
All function objects have a `__closure__` attribute that returns a tuple of cell objects if it is a closure function. Let's check `times5`.

```python
mul = make_multiplier_of(5)
print(mul.__closure__)
# (<cell at 0x101dac6d8: int object at 0x100980a00>,)
```
The cell object has the attribute cell_contents which stores the closed value.

```
print(mul.__closure__[0].cell_contents)
# 5
```

[top](#TableOfContents)

___

### Composition of Decorators

Function decorators are simply **wrappers to existing functions**.
They **alter the code execution before or after the wrapped function**. in other words, it's a callable that takes a callable and returns a callable.
Putting the ideas mentioned above together, we can build a simple decorator which wraps a string output of another function by a `strong` HTML tag.

```python
def get_text(name):
    return f"Lorem ninja ipsum dolor sit amet {name}"

def strong_decorate(func):
    def wrapper(name):
        return f"<strong>{func(name)}</strong>"
    return wrapper


get_strong_text = strong_decorate(get_text)
print(get_strong_text("Peter"))
# outputs: "<strong>Lorem ninja ipsum dolor sit amet Peter</strong>"
```

Our first decorator.
A function that takes another function as an argument, generates a new function, augmenting the work of the original function, and returning the generated function so we can use it anywhere. Simple as that.

There is a nice syntatic sugar for this in Python, using the `@` symbol.

[top](#TableOfContents)

___

### Decorator syntax

Python makes creating and using decorators a bit cleaner and nicer for the programmer through some syntactic sugar, prepending the function to be decorated with `@[function to decorate with]`.

In our case to decorate `get_text` we could use a shortcut.

```python
get_strong_text = strong_decorate(get_text)
```

is equivalent to:

```python
@strong_decorate
def get_text(name):
    return f"Lorem ninja ipsum dolor sit amet {name}"
```

Let's create a few other decorators and combine them.

```python
from lxml import etree, html


def strong(func):
    def wrapper(name):
        return f"<strong>{func(name)}</strong>"
    return wrapper


def div(func):
    def wrapper(name):
        return f"<div>{func(name)}</div>"
    return wrapper


def p(func):
    def wrapper(name):
        return f"<p>{func(name)}</p>"
    return wrapper


def prettify_html(func):
    def wrapper(name):
        document_root = html.fromstring(func(name))
        return etree.tostring(document_root, encoding='unicode', pretty_print=True)
    return wrapper


@prettify_html
@div
@p
@strong
def get_text(name):
    return f"Lorem ninja ipsum dolor sit amet {name}"

print(get_text("Peter"))

# outputs:
# <div>
#   <p>
#     <strong>Lorem ninja ipsum dolor sit amet Peter</strong>
#   </p>
# </div>
```

With the basic approach (without Python's syntatic sugar):

```python
decorated_get_text = prettify_html(div(p(strong(get_text))))
```

One important thing is that **the order of our applied decorators matters**. The order in which our decorators got applied is **from bottom to top**, decorator stacking. Note that if you stack many decorators it will eventually have an effect on performance, they keep adding nested function calls to the stack. In practice this is not a big problem, but worth keeping an eye on this, don't abuse the power of decorators.

[top](#TableOfContents)

___

### Deal with arguments

In the examples above we only had one argument `name` and we did know it beforehand. How would you deal with arbitary number of arguments in the decorator, and also forward the arguments to the input function?

If you try to apply one of the decorators from above to a function that takes arguments, it will not work correctly.

Python already has a feature for this scenarios. `*args` and `**kwargs` are exactly for this use case. Create a simple proxy decorator.

```python
def proxy(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

- the `wrapper` closure definition collect all positional and keyword arguments via the `*` and `**` operators, and stores them in variables (args, kwargs)
- the `wrapper` closure then forwards the arguments to the original input function `func` using the "unpacking" operators (`*` and `**`)

Expand this proxy decorator and create a bit more useful debug decorator from it.

```python
def debug(func):
    def wrapper(*args, **kwargs):
        print(f'[*] "{func.__name__}" called with: args={args} | kwargs={kwargs}')
        return func(*args, **kwargs)
    return wrapper

@debug
def greet(name):
    return f"Hy {name}!"

@debug
def complex_greet(name, base_sentence="Hy", punctuation="!"):
    return f"{base_sentence.capitalize()} {name}{punctuation}"

print(greet("Peter"))
# outputs:
# [*] "greet" called with: args=('Peter',) | kwargs={}
# Hy Peter!

print(complex_greet("Peter", base_sentence="Hello", punctuation="."))
# outputs:
# [*] "complex_greet" called with: args=('Peter',) | kwargs={'base_sentence': 'Hello', 'punctuation': '.'}
# Hello Peter.
```

As you can see from this "toy" example, inside our wrapper function we print out some info of our wrapped function before returning it. Decorating any function now with `debug` will print the arguments passed to the decorated function.

With a little extra tweaking we can create a basic debug "logger" for our pet projects:

```python
def debug(func):
    def wrapper(*args, **kwargs):
        if os.getenv("ENV", "dev").lower() == "dev":
            print(f'[*] "{func.__name__}" called with: args={args} | kwargs={kwargs}')
        return func(*args, **kwargs)
    return wrapper
```

[top](#TableOfContents)

___

### Debugging decorators

One thing we haven't talked about is what happens if the decorated function fails, or how can we access the decorated function's metadata like name, docstrings and parameter list?

```python
@debug
def complex_greet(name, base_sentence="Hy", punctuation="!"):
    """
    :param name: name to greet (str)
    :param base_sentence: sentence to greet with (str)
    :param punctuation: punctuation to end the greeting (str)
    :return: formatted greet sentence
    """
    return f"{base_sentence.capitalize()} {name}{punctuation}"

print(complex_greet.__name__)  # "wrapper"
print(complex_greet.__doc__)   # "None"
```

Not good, we lost important info about the wrapped function. This makes debugging decorated functions more challenging, because important **metadata which helps us understand stacktraces are missing**. So how to copy these you ask?

Luckily Python's standard library has an answer for this, `functools.wraps`.

You can use this decorator inside your decorators to copy over the lost metadata from the undecorated function to the decorator closure.

Let's pimp our `debug` decorator.

```python
from functools import wraps

def debug(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f'[*] "{func.__name__}" called with: args={args} | kwargs={kwargs}')
        return func(*args, **kwargs)
    return wrapper

print(complex_greet.__name__)  # "complex_greet"
print(complex_greet.__doc__)
#	:param name: name to greet (str)
#   :param base_sentence: sentence to greet with (str)
#   :param punctuation: punctuation to end the greeting (str)
#   :return: formatted greet sentence
```

As you can see it doesn't take much time to use `wraps` and it can save you from debugging headaches.

[top](#TableOfContents)

___

### Useful built-in decorators

- [@abc.abstractmethod](https://docs.python.org/3.7/library/abc.html#abc.abstractmethod) A decorator indicating
abstract methods.
- [@abc.abstractproperty](https://docs.python.org/3.7/library/abc.html#abc.abstractproperty) A subclass of the built-in
`property()`, indicating an abstract property.
- [@asyncio.coroutine](https://docs.python.org/3.7/library/asyncio-task.html#asyncio.coroutine) Decorator to mark
generator-based coroutines.
- [@atexit.register](https://docs.python.org/2.7/library/atexit.html#atexit.register) Register func as a function to
be executed at termination.
- [@classmethod](https://docs.python.org/3.7/library/functions.html#classmethod) Return a class method for function.
- [@contextlib.contextmanager](https://docs.python.org/3.7/library/contextlib.html#contextlib.contextmanager) Define a
factory function for with statement context managers, without needing to create a class or separate `__enter__()` and
`__exit__()` methods.
- [@functools.lru_cache](https://docs.python.org/3.7/library/functools.html#functools.lru_cache) Decorator to wrap a
function with a memoizing callable that saves up to the maxsize most recent calls. It can save time when an expensive
or I/O bound function is periodically called with the same arguments.
- [@functools.singledispatch](https://docs.python.org/3.7/library/functools.html#functools.singledispatch) Transforms
a function into a single-dispatch generic function.
- [@functools.total_ordering](https://docs.python.org/3.7/library/functools.html#functools.total_ordering) Given a
class defining one or more rich comparison ordering methods, this class decorator supplies the rest.
- [@functools.wraps](https://docs.python.org/3.7/library/functools.html#functools.wraps) This is a convenience function
for invoking update_wrapper() as a function decorator when defining a wrapper function.
- [@property](https://docs.python.org/3.7/library/functions.html#property) Return a property attribute.
- [@staticmethod](https://docs.python.org/3.7/library/functions.html#staticmethod) Return a static method for function.
- [@types.coroutine](https://docs.python.org/3.7/library/types.html#types.coroutine) This function transforms a
generator function into a coroutine function which returns a generator-based coroutine.
- [@unittest.mock.patch](https://docs.python.org/3.7/library/unittest.mock.html#unittest.mock.patch) Acts as a function
decorator, class decorator or a context manager. Inside the body of the function or with statement, the target is
patched with a new object. When the function/with statement exits the patch is undone.
- [@unittest.mock.patch.dict](https://docs.python.org/3.7/library/unittest.mock.html#unittest.mock.patch.dict)
Patch a dictionary, or dictionary like object, and restore the dictionary to its original state after the test.
- [@unittest.mock.patch.multiple](https://docs.python.org/3.7/library/unittest.mock.html#unittest.mock.patch.multiple)
Perform multiple patches in a single call.
- [@unittest.mock.patch.object](https://docs.python.org/3.7/library/unittest.mock.html#unittest.mock.patch.object)
Patch the named member (attribute) on an object (target) with a mock object.

[top](#TableOfContents)

___

### Summary

- everything in Python is an object, including functions
- functions can be nested
- functions can capture and carry some of the parent function's state --> closures
- decorators define reusable building blocks
- decorators can modify a callable's behavior
- '@' is just a syntax sugar
- use `functools.wraps`
- do not overuse decorators

[top](#TableOfContents)

___

In the following posts, I want to follow this path and introduce metaprogramming in Python through examples. Thanks for reading, hope you learned something new.
