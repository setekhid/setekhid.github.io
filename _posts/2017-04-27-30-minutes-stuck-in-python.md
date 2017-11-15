---
layout: post
title: "30 minutes stuck in python"
categories: python
author: "Huitse Tai"
---

This article is not intended to explain any details about python or its utilities, only for whom wanna start coding quickly.

While this sentence was being written, I only read some codes from some python projects. And I knew some basic knowledges and experiences about programming, like how to build a project, some other programming languages (java, scala, haskell, c/c++, nodejs etc.).

Now my problem is I wanna contribute to [dcos building scripts](github.com/dcos/dcos), so here we are,

[toc]

## 1. preparing

### 1.1. language versions and [brief history](https://en.wikipedia.org/wiki/Python_(programming_language)#History)

python project was started by [Guido van Rossum](https://en.wikipedia.org/wiki/Guido_van_Rossum) at 1989. 

python2 was released on 16 October 2000, and the latest minor version is 2.7, which was planning to be killed at 2020.

python3 was released on 3 December 2008, and the latest minor version is 3.6. python3 has very bad backwards-compatible, didn't know the reason, considering the limited time, we leave this funny question here.

We chosen python3, because it's the newest and major release, and using the elder version may cost us some time to repeat developing basic utilities, not good for new guys.

### 1.2. package manager (pip)

The most popular and the only solution has been told by google is [pip](https://en.wikipedia.org/wiki/Pip_(package_manager)).

You may wanna create a clean seperate python environment, [virtualenv](https://virtualenv.pypa.io/en/stable/) could help. Anyway I was planning docker.

So hurry up!

### 1.3. dependency manager

There are two popular solutions I knew for now, [setup.py](https://github.com/pypa/setuptools) and [requirements.txt](https://pip.pypa.io/en/stable/reference/pip_install/#requirements-file-format) by pip.

Note: above comparing may cause some wrong understanding, these two utilities are not intended to solve same problem. You may wanna do some google of ["setup.py vs requirements.txt"](https://caremad.io/posts/2013/07/setup-vs-requirement/).

Anyway, for now, I only need some where I can write my dependencies down and a tool to handle it automatically. Both can do.

I said I wanna use docker to handle runtime environment, [python:3.6-onbuild](https://hub.docker.com/_/python/) seems smooth for it. And [`pip install -r requirements.txt`](https://github.com/docker-library/python/blob/7eca63adca38729424a9bab957f006f5caad870f/3.6/onbuild/Dockerfile#L13) is how it handle onbuild event.

So carry requirements.txt and go to next!

### 1.4. ide (vim with python-mode)

[python-mode](https://github.com/python-mode/python-mode) is what I was planning to use. It's a vim plugin, you'd better to use some other plugin install it, like [pathogen](https://github.com/tpope/vim-pathogen).

And there are [a few configurations](https://github.com/setekhid/_vim/blob/master/_vimrc#L68) to make us convenient.

You may also wanna reinstall vim to match python-mode's requirements.

```bash
brew install vim --with-python3
```

### 1.5. entry point (main function)

python is a script programming language, so it is playing like a script, each code directly written on file scope will be executed immediately. Such like definition of functions, variables or classes, or calling to one function, statements.

So the file which you feed to interpreter is like the main function in c.

But we still have some rules, (I'm not sure actually), like nodejs I guess we should write the definition on the file scope, nothing more, and use

```python
if __name__ == "__main__":
	main()
```

as main function.

`__name__` above means the module name which importing or running current module. [`__main__`](https://docs.python.org/3/library/__main__.html) is the scope in which top-level code interpreting.

More details, you should search python module system.

Briefly when you imported module A into module B, module A would be put into module B's symbol table, the scope A which is the global scope of module A would be under the module B's global scope. In scope A's opinion, module B is its module name.

(Above is just my guess, LOL, Enjoy it!)

So anyway next!

## 2. coding

### 2.1. packages

First, there is a concept named searching path. python has a definition named [sys.path](https://docs.python.org/3/library/sys.html#sys.path) as its [library searching path](https://docs.python.org/3/tutorial/modules.html#the-module-search-path).

Shortly, input script's location directory will be the library searching path. python will search packages and modules from searching path while importing something into activiting module.

[packages](https://docs.python.org/3/tutorial/modules.html#packages) in python can contain other packages or module files. The package name normally is the directory name which contains package's contents. And a package must contain a file named `__init__.py` on its directory directly. python will treat this file's directory as a package and will execute this file when the package was first imported.

A possible structure may look like below:

```directory
pkg01/
	__init__.py
	mod01.py
	pkg02/
		__init__.py
	pkg03/
		__init__.py
		mod02.py
		mod03.py
```

There is a variable [`__all__`](https://docs.python.org/3/tutorial/modules.html#importing-from-a-package) defines what can be imported when you run `from pkg01.pkg03 import *`.

### 2.2. modules

[modules](https://docs.python.org/3/tutorial/modules.html#modules) is a smaller scope in python than packages.

A module is a file containing Python definitions and statements. The file name is the module name with the suffix `.py` appended. A module files contains statements including calling and definitions.

All the code, calling and definitions on the module file scope will be executed immediatly when this module file was being imported the first time.

Pretending we have `mod03.py` as below,

```python
def hello():
	print("Hello, World!")
```

And you can import a package, a module or a definition into current scope, like below,

```python
import pkg01.pkg03.mod03
pkg01.pkg03.mod03.hello() # so now you can call like this

from pkg01.pkg03 import mod03
mod03.hello() # or like this
```

And in `mod02.py`, you can do,

```python
import mod03 # same like the first import, because they are in the same parent scope
mod03.hello()

from mod03 import hello # of course, you can do this outside this package as well
hello()
```

### 2.3. classes

python is a dynamic script language, it's [class definition](https://docs.python.org/3/tutorial/classes.html#class-definition-syntax) looks like below,

```python
class class_name(parent_class01, parent_class02):
	"""docstring of this class which will be assigned to __doc__"""

	def __init__(self, arg1, arg2):
		# this method is optional, only called when this class initializing.
		parent_class01.__init__(self, arg1)
		parent_class02.__init__(self, arg2)
		<statement01>
		...

	<statement02>
	<statement03>
	...
```

You can place any legal code in class body, imaging what you can do a minute. And normally you should only put definitions in here.

Parent classes inheriting to specified in brackets behind. And you should mannually call their initializing methods if necessary. python will left-depth search a method when you invoke one, which means a method will be searching recursively like below,

```
derived_obj.demo_method()
if calling failed
	base1_obj.demo_method()
	if calling failed
		base2_obj.demo_method()
		...
```

The instantiation of above class should be like ```obj_name = class_name(var1, var2)```, ```__init__``` method will be invoked automatically. In this example, it will call the init method of its two parent classes orderly.

**Notice**, the example below,

```python
class different_definitions:
	"""there are two different variables, to be clear which object they belong to"""

	var1 = "belong to class object"
	
	def __init__(self):
		self.var2 = "belong to each instances created from this class"
```

```var1``` belongs to class object ```different_definitions```, and ```var2``` belongs to instance object ```haha``` when you run ```haha = different_definitions()```.

And if you need define some private variables in class, [dig deeper](https://docs.python.org/3/tutorial/classes.html#private-variables).

### 2.4. functions & definitions

python function can be defined like below,

```python
def func01(arg1, arg2):
	"""some complains or nagging maybe"""
	<statement01>
	<statement02>
	...
	return result
```

The first statement of the function body can optionally be a string literal as this function's [docstring](https://docs.python.org/3/tutorial/controlflow.html#tut-docstrings).

That will be enough for me, but you may wanna [dig more](https://docs.python.org/3/tutorial/controlflow.html#defining-functions).

### 2.5. exception handling

see [here](https://docs.python.org/3/tutorial/errors.html).

This blog costs me about three days to write down. But I expect you can read it in 30 minutes and get started with python project. The last chapter has been dismissed, cause it's not useful for me for now.

Now, go ahead, write something, google and stackoverflow will be your best teacher.

## 3. dig deeper

* syntaxes and keywords
* dynamic features
* [namespace mechanisms](https://docs.python.org/3/tutorial/classes.html#python-scopes-and-namespaces)
* metaprogramming and metaobjects programming paradigm
* cycle-detecting garbage collector
* [python interpreter](github.com/python/cpython)
* [jit interpreter](pypy.org)
* differences between python3 and python2

## 4. edited

### 26/04/2017

DO NOT use vim plugin [python-mode](https://github.com/klen/python-mode), if you are a new guy. Instead, using [PyCharm](https://www.jetbrains.com/pycharm/) as your IDE, I suggest.

python-mode will break macvim and will report you lots of useless errors and warnings of your code. DO NOT use it if you have another choise.
