[![Build Status](https://travis-ci.org/alehander42/pseudo-python.svg?branch=master)](https://travis-ci.org/alehander42/pseudo-python)
[![MIT License](http://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

#pseudo-python

A restricted Python to idiomatic JavaScript / Ruby / Go / C# translator

[Pseudo](https://github.com/alehander42/pseudo) is a framework for high level code generation: it is used by this compiler to translate a subset of Python to all Pseudo-supported languages

**If you are using Python3.5 and you experience problems with an already installed version of pseudo-python, please upgrade it to `0.2.14` (`pip3 install pseudo-python --upgrade`)**



## Supported subset

Pseudo-Python compiles to `pseudo ast`. 

[Pseudo](https://github.com/alehander42/pseudo) defines a language-independent AST model and an unified standard library.
It can map its own standard library to target language libraries and concepts automatically and it tries to generate readable and idiomatic code.

Pseudo-Python translates a subset of Python to Pseudo AST and then it receives the JS/Ruby/C#/Go backends for free. (+ at least 4-5 backends in the future)

Pseudo was inspired by the need to generate algorithms/code in different languages or portint tools/libraries to a new environment

That's why it can be mapped to a well defined subset of a language

It is meant as a framework consuming ast from parser generators / compilers / various tools and generating snippets / codebases in different target languages

# Plan

![a diagram illustrating the pseudon framework: compilers -> ast -> api translation -> target code](http://i.imgur.com/7ySfy5j.jpg?2)

# Pseudo supports

  * basic types and collections and standard library methods for them
  
  * integer, float, string, boolean
  * lists
  * dicts
  * sets
  * tuples/structs(fixed length heterogeneous lists)
  * fixed size arrays
  * regular expressions

  * functions with normal parameters (no default/keyword/vararg parameters)
  * classes 
    * single inheritance
    * polymorphism
    * no dynamic instance variables
    * basically a constructor + a collection of instance methods, no fancy metaprogramming etc supported

  * exception-based error handling with support for custom exceptions
  (target languages support return-based error handling too)
  
  * io: print/input, file read/write, system and subprocess commands

  * iteration (for-in-range / for-each / iterating over several collections / while)
  * conditionals (if / else if / else)
  * standard math/logical operations

## Installation


```bash
pip install pseudo-python
```

## Usage

```bash
pseudo-python <filename.py> ruby
pseudo-python <filename.py> csharp
``` 
etc for all the supported pseudo targets (javascript, c#, go, ruby and python)

## examples

Each example contains a detailed README and working translations to Python, JS, Ruby, Go and C#, generated by Pseudo

[fibonacci](examples/fib)

[a football results processing command line tool](examples/football)

[a verbal expressions-like library ported to all the target languages](examples/verbal_expressions)


## Error messages

A lot of work has been put into making pseudo-python error messages as clear and helpful as possible: they show the offending snippet of code and 
often they offer suggestions, list possible fixes or right/wrong ways to write something

![Screenshot of error messages](http://i.imgur.com/Et3X9W1.png)

Beware, pseudo and especially pseudo-python are still in early stage, so if there is anything weird going on, don't hesitate to submit an issue

## Type inference

pseudo-python checks if your program is using a valid pseudo-translatable subset of Python, type checks it according to pseudo type rules and then generates a pseudo ast and passes it to pseudo for code generation.


The rules are relatively simple: currently pseudo-python infers everything
from the usage of functions/classes, so has sufficient information when the program is calling/initializing all
of its functions/classes (except for no-arg functions)

Often you don't really need to do that for **all** of them, you just need to do it in a way that can create call graphs covering all of them  (e.g. often you'll have `a` calling `b` calling `x` and you only need to have an `a` invocation in your source)

You can also use type annotations. We are trying to respect existing Python3 type annotation conventions and currently pseudo-python recognizes `int`, `float`, `str`, `bool`, `List[<type>]`, 
`Dict[<key-type>, <value-type>]`, `Tuple[<type>..]`, `Set[<type>]` and `Callable[[<type>..], <type>]`

Beware, you can't just annotate one param, if you provide type annotations for a function/method, pseudo-python expects type hints for all params and a return type

Variables can't change their types, the equivalents for builtin types are
```python
list :  List[@element_type] # generic
dict:   Dictionary[@key_type @value_type] # generic
set:    Set[@element_type] # generic
tuple:  Array[@element_type] # for homogeneous tuples
        Tuple[@element0_type, @element1_type..] # for heterogeneous tuples
int:    Int
float:  Float
int/float: Number
str:    String
bool:   Boolean
```

There are several limitations which will probably be fixed in v0.3

If you initialize a variable/do first call to a function with a collection literal, it should have at least one element(that limitation will be until v0.3)

All attributes used in a class should be initialized in its `__init__`

Other pseudo-tips:

* Homogeneous tuples are converted to `pseudo` fixed length arrays and heterogeneous to `pseudo` tuples. [Pseudo](https://github.com/alehander42/pseudo) analyzes the tuples usage in the code and sometimes it translates them to classes/structs with meaningful names if the target language is `C#` `C++` or `Go` 

* Attributes that aren't called from other classes are translated as `private`, the other ones as `public`. The rule for methods is different:
`_name` ones are only translated as `private`. That can be added as
config option in the future

* Multiple returns values are supported, but they are converted to `array`/`tuple`

* Single inheritance is supported, `pseudo-python` supports polymorphism
but methods in children should accept the same types as their equivalents in the hierarchy (except `__init__`)

The easiest way to play with the type system is to just try several programs: `pseudo-python` errors should be enough to guide you, if not, 
you can always open an issue

## How does Pseudo work?


The implementation goal is to make the definitions of new supported languages  really clear and simple. 

If you dive in, you'll find out
a lot of the code/api transformations are defined using a declarative dsl with rare ocassions 
of edge case handling helpers. 

That has a lot of advantages:

* Less bugs: the core transformation code is really generalized, it's reused as a dsl and its results are well tested

* Easy to comprehend: it almost looks like a config file

* Easy to add support for other languages: I was planning to support just python and c# in the initial version but it is so easy to add support for a language similar to the current supported ones, that I
added support for 4 more.

* Easy to test: there is a simple test dsl too which helps all 
language tests to share input examples [like that](pseudo/tests/test_ruby.py)

However language translation is related to a lot of details and
a lot of little gotchas, tuning and refining some of them took days. Pseudo uses different abstractions to streamline the process and to reuse logic across languages.

```ruby
PSEUDO AST:
   NORMAL CODE     PSEUDO STANDARD LIBRARY INVOCATIONS     
      ||                    ||
      ||                    ||
      ||              API TRANSLATOR
      ||                    ||
      ||                    ||
      ||                    \/
      ||              IDIOMATIC TARGET LANGUAGE 
      ||              STANDARD LIBRARY INVOCATIONS        
      ||                    ||     
      \/                    \/
  STANDARD OR LANGUAGE-SPECIFIC MIDDLEWARES
              e.g.
    name camel_case/snake_case middleware
    convert-tuples-to-classes middleware
    convert-exception-based errors handling
    to return-based error handling middleware
              etc

              ||
              ||
              ||
              ||
  TARGET LANGUAGE CODE GENERATOR

      defined with a dsl aware
      that handles formatting
         automatically
              ||
              ||
              ||
              \/

            OUTPUT
```


## What's the difference between Pseudo and Haxe?

They might seem comparable at a first glance, but they have completely different goals.

Pseudo wants to generate readable code, ideally something that looks like a human wrote it/ported it

Pseudo doesn't use a target language runtime, it uses the target language standard library for everything (except for JS, but even there is uses `lodash` which is pretty popular and standard)

Pseudo's goal is to help with automated translation for cases
like algorithm generation, parser generation, refactoring, porting codebases etc. The fact that you can write compilers targetting Pseudo and receiver translation to many languages for free is just a happy accident


## License

Copyright © 2015 2016 [Alexander Ivanov](https://twitter.com/alehander42)

Distributed under the MIT License.
