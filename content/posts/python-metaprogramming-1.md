---
title: "Metaprogramming in Python, part 1"
date: 2019-02-03T17:41:54+01:00
description: "Multiple inheritance, new vs old style classes, MRO, what does type do?"
categories: [ "python", "metaprogramming", "programming", "OOP" ]
tags: ["python", "type", "metaprogramming", "classes", "MRO", "OOP"]
---


# Metaprogramming in Python, part 1



___



Topics covered in this part:

- multiple inheritance
- new vs old classes
- method resolution order (MRO) 



___



Metaprogramming is basically a computer program that write or manipulate other programs (or itself) as their data. In many cases, this allows to get more done in the same amount of time as it would take to write all the code manually, or it gives programs greater flexibility to efficiently handle new situations without recompilation or verbosity.

 

You can see examples for this type of programming in popular python packages like [Flask](https://github.com/pallets/flask/blob/master/flask/wrappers.py), [Django](https://github.com/django/django/blob/master/django/utils/decorators.py) or [SQLAlchemy](https://github.com/sqlalchemy/sqlalchemy/blob/master/lib/sqlalchemy/orm/util.py).

We will discuss ways of metaprogramming in Python and how metaprogramming can simplify certain tasks, but before we should talk a bit about what exactly `type` is doing and how classes work.



___



## Basics



Object oriented programming comes with other types of problems when compared to the functional way. Notable how to connect these objects with each other. As you probably already heard the term inheritance, it's one of the ways. Not all languages but Python do allow multiple inheritance which in itself comes with the [Diamond problem](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem).



![Diamond problem](/posts/diamond.png)



>The "deadly diamond of death" is an ambiguity that arises when two classes B and C inherit from A, and class D inherits from both B and C. If there is a method in A that B and C have overridden, and D does not override it, than which version of the method does D inherit: that of B, or that of C?



Languages have different ways of dealing with this problem.

Different solutions:



* [mixin classes](https://en.wikipedia.org/wiki/Mixin) (Scala, Perl, Python, Smalltalk)

* [trait](https://en.wikipedia.org/wiki/Trait_(computer_programming)) (PHP, Rust)

* [virtual inheritance](https://en.wikipedia.org/wiki/Virtual_inheritance) (C++)

* [interfaces](https://en.wikipedia.org/wiki/Interface_(computing)#In_object-oriented_languages) (Java)

* [protocols](https://www.tutorialspoint.com/objective_c/objective_c_protocols.htm) (Objective-C, Swift)



___



##### Mixin



*Interface with implemented methods.*



A class that contains methods for use by other classes without having to be the parent class of those other classes.

**"being included"** rather than "inherited"



Example in Scala ([source](https://docs.scala-lang.org/tour/mixin-class-composition.html))



```scala
abstract class AbsIterator {
  type T
  def hasNext: Boolean
  def next(): T
}

class StringIterator(s: String) extends AbsIterator {
  type T = Char
  private var i = 0
  def hasNext = i < s.length
  def next() = {
    val ch = s charAt i
    i += 1
    ch
  }
}


trait RichIterator extends AbsIterator {
  def foreach(f: T => Unit): Unit = while (hasNext) f(next())
}

// we combine the functionality of StringIterator and RichIterator
// into a single class
// the new class RichStringIter
// has StringIterator as a superclass and RichIterator as a mixin
object StringIteratorTest extends App {
  class RichStringIter extends StringIterator("Scala") with RichIterator
  val richStringIter = new RichStringIter
  richStringIter foreach println
}
```



(extra: [dependency inversion principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle))



___



##### Trait



Set of methods that can be used to extend the functionality of a class.

Stands somewhere between an interface and mixin. Defines behaviors via full method definition (== includes the body of the methods), this differns a bit when compared to mixins that can also carry state through marker variables (traits usually don't).



A set of methods that implement behavior to a class + require that the class implement the a set of methods that parametrize the provided behaviour.



Example ([source](https://doc.rust-lang.org/rust-by-example/trait.html))

```rust
struct Sheep { naked: bool, name: &'static str }

trait Animal {
    // Static method signature; `Self` refers to the implementor type.
    fn new(name: &'static str) -> Self;
    // Instance method signatures; these will return a string.
    fn name(&self) -> &'static str;
    fn noise(&self) -> &'static str;
    // Traits can provide default method definitions.
    fn talk(&self) {
        println!("{} says {}", self.name(), self.noise());
    }
}

impl Sheep {
    fn is_naked(&self) -> bool {
        self.naked
    }
    fn shear(&mut self) {
        if self.is_naked() {
            // Implementor methods can use the implementor's trait methods.
            println!("{} is already naked...", self.name());
        } else {
            println!("{} gets a haircut!", self.name);
            self.naked = true;
        }
    }
}

// Implement the Animal trait for Sheep.
impl Animal for Sheep {
    // `Self` is the implementor type: `Sheep`.
    fn new(name: &'static str) -> Sheep {
        Sheep { name: name, naked: false }
    }

    fn name(&self) -> &'static str {
        self.name
    }

    fn noise(&self) -> &'static str {
        if self.is_naked() {
            "baaaaah?"
        } else {
            "baaaaah!"
        }
    }

    // Default trait methods can be overridden.
    fn talk(&self) {
        // For example, we can add some quiet contemplation.
        println!("{} pauses briefly... {}", self.name, self.noise());
    }
}


fn main() {
    // Type annotation is necessary in this case.
    let mut dolly: Sheep = Animal::new("Dolly");
    dolly.talk();
    dolly.shear();
    dolly.talk();
}
```



___


##### Interfaces

Since Java 8, the language introduces default methods on interfaces.
Java simply solves the diamond problem by requiring that class D must reimplement the method (the body of which can simply forward the call to one of the super implementations) or it gives you a nice compile error.



Example in Java ([source](https://stackoverflow.com/a/8999131))



```java
interface Animal {
    public void eat();
    public void sleep();   
}

class Lion implements Animal {
    public void eat() {
        // Lion's way to eat
    }
    public void sleep(){
         // Lion's way to sleep
    }
}

class Monkey implements Animal {
    public void eat() {
        // Monkey's way to eat
    }
    public void sleep() {
        // Monkey's way to sleep
    }
}
```


___



## [Type](https://docs.python.org/2/library/types.html)



In a [previous post](https://blog.peterl.io/2019/01/python-decorators/#about-functions) we already mentioned that everything is an object in Python, we also know that classes create objects. 

You might ask at this point if everything is an object (and classes are also objects), what is the root? What/who creates these classes?



Let's check it out

```python
>>> class Foo:
...   pass
...
>>> foo = Foo()
>>> type(foo)
<type 'instance'>
``` 
dive deeper then

```python
from inspect import *
>>>isclass(Foo)
True
>>>isclass(foo)
False
>>>isclass(type(foo))
True
>>>isinstance(foo, Foo)
True
>>>isinstance(Foo, type)
True
>>>type(type(type(Foo)))
<type 'type'>
>>>isclass(type(type(type(Foo))))
True
```



Everything is an object in Python, and they are all either instances of classes or instances of metaclasses, except for `type`


`type` is itself a class, and it is its own type. It's a metaclass. A metaclass instantiate and defines behavior for a class just like a class instantiate and defines behavior for an instance.


`type` is the built-in metaclass Python uses. To change the behavior of classes in Python (like the behavior of Foo), we can define a custom metaclass by inheriting the type metaclass. Metaclasses are *a* way to do metaprogramming in Python.


It's equal to write

```python
class Foo:
  pass
```


```python
Foo = type('Foo', (), {})
```

spice things up

```python
>>> Foo = type('Foo', (object,), {})
>>> Bar = type('Bar', (Foo,), dict(name='FooBar',))
>>> type(Foo)
<type 'type'>
>>> vars(Foo)
dict_proxy({'__dict__': <attribute '__dict__' of 'Foo' objects>, '__module__': '__main__', '__weakref__': <attribute '__weakref__' of 'Foo' objects>, '__doc__': None})
>>> Foo.name
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: type object 'Foo' has no attribute 'name'
>>> Bar.name
'FooBar'
>>> Bar.__dict__
dict_proxy({'__module__': '__main__', 'name': 'FooBar', '__doc__': None})
```


we've just created a new class using type.


___



A metaclass is the class of a class. Basically a metaclass defines how a class behaves. A metaclass is most commonly used as a class-factory allowing you to do extra-things when creating a class. You can also define how common dunder methods (`__add__`, `__iter__`, `__getattr__`) will behave in your derived classes.



Now you can see how classes are created under the hood let's talk about how Python solves the diamond problem.



## Method resolution order


- almost like in Perl
- the order of inheritance affects the class semantics
- different in old-style classes <--> new-style classes


##### old-style


```python
class Foo():
	pass
```

##### new-style

```python
class Foo(object):
	pass

# in Python3+ `object` implicitly added even if you left it
```

The resolution order differs in old-style classes, using a [pre-order depth-first search](https://en.wikipedia.org/wiki/Tree_traversal#Pre-order_(NLR)).

```python
#!/usr/bin/python

class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass
```

Check the mro.

```python
>>> D.mro()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: class D has no attribute 'mro'
>>> D.__mro__
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: class D has no attribute '__mro__'
```

Looks like we can't check the mro like this, try a with different example.

```python
>>> class A: i = 0
...
>>> class B(A): pass
...
>>> class C(A): i = 2
...
>>> class D(B, C): pass
...
>>> class E(C, B): pass
...
>>> assert D().i == 0
>>>
>>> assert E().i == 2
```

with old-style classes the MRO is `D > B > A > C`.

Check it with new-style classes too, now we should be able access it with `__mro__`

```python
#!/usr/bin/python

class A(object): pass
class B(A): pass
class C(A): pass
class D(B, C): pass

>>> D.mro()
[<class '__main__.D'>,
 <class '__main__.B'>,
 <class '__main__.C'>,
 <class '__main__.A'>,
 <type 'object'>]
```



Let's visualize it a bit more with some variables.



```
>>> class A(object): i = 0
...
>>> class B(A): pass
...
>>> class C(A): i = 2
...
>>> class D(B, C): pass
...
>>> class E(C, B): pass
...
>>> # with old-style this was 0
>>> assert D().i == 2  
>>>
>>> assert E().i == 2
```



The difference is pretty obvious. Rather than exhausting the parent's parent chain it continues with the childer first.



With new-style classes the MRO is `D > B > C > A`

This new method resolution order use a [post-order depth-first search](https://en.wikipedia.org/wiki/Tree_traversal#Post-order_(LRN)) or more precisely [C3 linerization](https://en.wikipedia.org/wiki/C3_linearization).



Worth to mention that attribute lookup still faster than with new-style classes.


```python
class A():
    def __init__(self):
        self.a = 5

class B(object):
    def __init__(self):
        self.a = 5

if __name__ == '__main__':
    import timeit
    print(timeit.timeit("a.a", setup="from __main__ import a"))
    print(timeit.timeit("b.a", setup="from __main__ import b"))

>>> 0.036071062088
>>> 0.047596931457
```


In Python3+ you wouldn't notice this, object is the default.



## Summary



We covered quite a bit of topics in this post. Next we'll continue with mixin examples in Python, dunder methods and finally start examining some metaprogramming recipes, examples.



Multiple inheritance give us powerful patterns that we can use daily to solve recurring problems or even apply *behaviours* to classes dynamically runtime.  

These concepts allow us developers to work with some nice abstractions on how the language's basic building blocks behave.



I find these topics very fascinating. There were few times when thinking outside the box and applying one single metaclass solved a problem what looked unsolvable without the need of refactoring huge parts of the existing code base. 



Thanks for reading.

Happy coding! :)

___



## Further resources



* [Python new vs classic style object](https://wiki.python.org/moin/NewClassVsClassicClass)
* [Multiple inheritance](https://stackoverflow.com/a/226056)
* [Mixins](https://stackoverflow.com/a/547714)
* [Python Data Model](https://docs.python.org/3/reference/datamodel.html)
* [Python Data Model examples](https://pybit.es/python-data-model.html)
* [Class vs type in Python](https://stackoverflow.com/a/35959047)
* [Python Dunder methods deep dive](https://dbader.org/blog/python-dunder-methods)
* [Abusing dunder methods](https://github.com/vakila/dunders)
* [VIDEO - Metaclasses in Python](https://www.youtube.com/watch?v=ZrUIRSVv1gw)
* [VIDEO - Metaprogramming in Python](https://www.youtube.com/watch?v=Adr_QuDZxuM)
* [VIDEO - Python OOP Tutorial](https://www.youtube.com/watch?v=ZDa-Z5JzLYM&list=PL-osiE80TeTsqhIuOqKhwlXsIBIdSeYtc)
* [VIDEO - SOLID Principles of Object Oriented and Agile Design](https://www.youtube.com/watch?v=TMuno5RZNeE)


