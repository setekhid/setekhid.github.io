---
layout: post
title: "one hour step into erlang"
categories: erlang
author: "Huitse Tai"
---

Another time-limited tutorial to one language study, I wrote these blogs just for somebody like me who already have some basic knowledges of programming and already knew a few of languages to quick start developing.

I structured some basic useful language concepts orderly in these blogs. These concepts are what we need to start writing something, such like a brief history (what scene this language for and some basic concepts and designs), package manager (dependency, installing and utilities), project builder (or automatically builder, usually to solve dependency problem), ide (of course) and coding (what will be ordered as biggest global namespace to the local scope).

Now I give you another one hour to start coding with erlang. Let's do this.[^getting_started-5.4.pdf][^ibm erlang1][^ibm erlang2]

[toc]

  [^getting_started-5.4.pdf]: a quick start tutorial of erlang basic syntaxes http://erlang.org/download/getting_started-5.4.pdf

  [^ibm erlang1]: erlang tutorial part 1 from ibm developer https://www.ibm.com/developerworks/library/os-erlang1/

  [^ibm erlang2]: erlang tutorial part 2 from ibm developer https://www.ibm.com/developerworks/library/os-erlang2/

## 1. overview & utilities

when we talk about erlang, we normally are talking about the language basic syntax and otp platform.

### 1.1. brief history[^erlang wikipedia]

erlang was first appeared at 1986. erlang was designed to improve the development of telephony applications. it was designed with distributed, fault-tolerant and ha in mind.

normally when we call it erlang, we mean the language and its otp framework.

otp was originally an acronym of Open Telecom Platform.[^otp wikipedia] but now it's more like a common application framework. 

otp including a few common components and frameworks for developing distributed applications.

### 1.2. package manager

no more nonsense, use [hex][1] with no question. the rebar3 is using it and rebar3 is under erlang orgnization. we will talk it in next section.

### 1.3. project builder

rebar3 is what I've chosen. see above for the reason.

here is a [basic usage][2]. take a quick view and using `rebar3 new app taste_rebar_app` and `rebar3 new lib taste_rebar_lib` to generate two otp standard projects and take a look of which file you'd like to.

### 1.4. ide

just use [erlide][3].

[here][4] is how to install it. it's just a plugin of eclipse. also a vim key binding for eclipse [vrapper][5].

  [^erlang wikipedia]: wikipedia of erlang https://en.wikipedia.org/wiki/Erlang_(programming_language)

  [^otp wikipedia]: wikipedia of OTP https://en.wikipedia.org/wiki/Open_Telecom_Platform

  [1]: https://hex.pm
  [2]: http://www.rebar3.org/docs/basic-usage
  [3]: http://erlide.org
  [4]: http://erlide.org/articles/eclipse/120_Installing-and-updating.html
  [5]: http://vrapper.sourceforge.net/home/

## 2. coding

### 2.1. module

Each file contains what we call an erlang module. The file which are used to store the module must have the same name as the module but with the extension ".erl".

we using `-module` to declare a module, and `.` end one statement.

```erlang
% file one_hour_tut.erl
-module(one_hour_tut).
```

and the functions within the module need to be exported using `-export`, like below,

```erlang
% export two function lets_go with one argument and another with two arguments
-export([lets_go/1, lets_go/2]).
```

calling the function looks like this `one_hour_tut:lets_go(args).`.

[getting more details][6]

### 2.2. functions

functions normally defined as below,

```erlang
lets_go(all) ->
	"let's do this.";
lets_go(Who) ->
	_Pre = "let's go, ",
	_Pre ++ atom_to_list(Who).
```

> **note**:
> variables in erlang must begin with capital letter or underscore, see [stackoverflow][7], its style is from prolog. and atoms(below) begin with small letters.

the semicolon separate the two pattern matching of this function. comma here is like `let ... in ...` in haskell.

about functions, there are some basic concepts, like most other functional programming language. you can read more on the [reference][8].

### 2.3. fundamental types

erlang has 8 fundamental types[^erlang types], integers, floats, atoms, references, binaries, pids, ports, funs, tuples, lists, maps, strings and records.

some interesting types list below,

* atoms

atoms normally started with lower-case letter, otherwise, it must be put in single quotes. such like `atom1`, `atom2`, `'_atom3'` and `'Atom4'`.

* funs

is the type of functions, you can declare it with key word `fun` like below,

```erlang
Fun1 = fun (X) -> X+1 end.
```

* pids

it's a most important type for erlang, cause erlang is a concurrency language.

pid is the short name of process identifier.

* tuples, maps, lists

like lots of other functional language, all of those same paradigm languages were proud to say they are playing those three types happy.

```erlang
A = {this, is, 1, tuple}.
B = #{this=>is, a=>map}.
C = [this, is, a, list].
```

* strings

strings in erlang are enclosed in double quotes. like `"this is a string"`, different to atoms.

> **note**:
> erlang doesn't have boolean, it use two atoms `true` and `false` as its boolean result.

## 3. distributed erlang

erlang is designed to be distributed as its core concept.

### 3.1. basic

https://learnxinyminutes.com/docs/erlang/

```erlang
% TODO
```

### 3.2. otp

```erlang
% TODO
```

  [6]: http://erlang.org/doc/reference_manual/modules.html
  [7]: http://stackoverflow.com/questions/28324245/why-the-variables-in-erlang-are-designed-to-begin-with-capital-letter
  [8]: http://erlang.org/doc/reference_manual/functions.html

  [^erlang types]: erlang basic types http://erlang.org/doc/reference_manual/data_types.html

## 3. dig deeper

* compare beam vs jvm
* erlang otp mechanism http://learnyousomeerlang.com/what-is-otp and dig much more into otp libraries
* more about concurrency, distributed, service discovery and database
* error handling http://erlang.org/doc/reference_manual/errors.html
* macros and header file http://erlang.org/doc/reference_manual/macros.html
