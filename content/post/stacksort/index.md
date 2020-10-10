---
title: "StackSort"
date: 2020-10-10T10:28:16-04:00
tags: ["demo"]
draft: true
---

{{< warning >}}
Running arbitrary untrusted code is a bad idea, use your head.
{{< /warning >}}

## Background

This project was inspired by the XKCD comic [Ineffective Sorts](https://xkcd.com/1185/) and the similar project by Gregory Koberger, [stacksort](https://gkoberger.github.io/stacksort/).

{{< blockquote author="Randall Munroe" >}}StackSort connects to StackOverflow, searches for 'sort a list', and downloads and runs code snippets until the list is sorted.{{< /blockquote >}}

I wanted to be able to run download and run code from StackOverflow the same way I would use any normal Python package.

{{< code >}}
from stacksort import quicksort
.
.
.
{{< /code >}}

## Hooking the Import System

Code can by found [here](https://github.com/buckley-w-david/stacksort/blob/master/stacksort/_meta/injector.py) (`StackSortFinder` and `StackSortLoader`) and [here](https://github.com/buckley-w-david/stacksort/blob/master/stacksort/__init__.py).

Most of what I needed I figured out by reading [this blog post](https://dev.to/dangerontheranger/dependency-injection-with-import-hooks-in-python-3-5hap), and the rest of some trial and error. It's actually not that difficult to hook the import system, it came down to defining two very simple classes.

{{< code >}}
class StackSortFinder(importlib.abc.MetaPathFinder):
    _COMMON_PREFIX = "stacksort."

    def __init__(self, loader):
        self._loader = loader

    def find_spec(self, fullname, path, target=None):
        if fullname.startswith(self._COMMON_PREFIX):
            name = fullname[len(self._COMMON_PREFIX):]
            return self._gen_spec(name)

    def _gen_spec(self, fullname):
        return importlib.machinery.ModuleSpec(fullname, self._loader)
{{< /code >}}
<!--_ VIM doesn't realize that the underscores in there don't mean it should italicise-->

`find_spec` needs to return a [`ModuleSpec`](https://docs.python.org/3/library/importlib.html#importlib.machinery.ModuleSpec) to intercept the import (Not returning anything means the import is not intercepted).

`from stacksort import quicksort` will generate a `fullname` of `stacksort.quicksort`, so we check for and strip off the prefix `stacksort.`, return the `ModuleSpec`.


{{< code >}}
class StackSortLoader(importlib.abc.Loader):
    def create_module(self, spec):
        return StackOverflowStuff(spec.name)

    def exec_module(self, module):
        pass
{{< /code >}}

The loader needs two methods, `create_module` and `exec_module`. We're not doing anything that requires the `exec_module` method so it's a noop.

`create_module` returns *something* that will do the actual work of fetching and compiling StackOverflow code. This is the object that the user gets from importing. We can use `spec.name` to access the name of what the user was trying to import so we can use that to search for the right code.


{{< code >}}
from stacksort._meta import injector
import sys

loader = injector.StackSortLoader()
finder = injector.StackSortFinder(loader)
sys.meta_path.append(finder)
{{< /code >}}

After those are defined it's a matter of instantiating both classes and adding our finder to `sys.meta_path`.

Viola! The import system has been hooked.

By putting the above code chunk in the package's `__init__.py` file, it runs on package import, before python tries to import anything else. This means that there's no explicit setup step that an importer needs to run, the import hooking Just Works™.

## Fetching Code

[Code can be found here](https://github.com/buckley-w-david/stacksort/blob/master/stacksort/_meta/stackoverflow/find.py)

This part was nothing novel, after a half-second of searching I found [stackapi](https://stackapi.readthedocs.io/en/latest/), which let me leverage the [StackExchange API](https://api.stackexchange.com/).

The main thrust is the following:
1. Search of questions tagged with 'python' and 'sorting', and use whatever the user was trying to import ("quicksort" for instance) as the mysterious "q" parameter.  
The "q" paramter is "a free form text parameter, will match all question properties based on an undocumented algorithm".
2. Fetch the answers to the questions returned from that search.
3. loop through those answers, extracting and returning code blocks.

Aside from that is some further bookkeeping involved: 
 - Fetching more questions/answers if no appropriate ones were found.
 - if a `safety_date` has been provided (The default is that the date I put it up on GitHub is used), don't accept any questions/answers that were posted (or edited) after that date. This is a cue I took from Gregory Koberger's implementation that improves safety a bit since it prevents anyone from specifically targetting this library with malicious code since nobody knew about it before now.
 - The function `yield`s the code blocks as it extracts them. This pattern makes it easy to consume all the code blocks by iterating over the generator returned by calling the function. 
 - A friend of mine suggested having the ability to randomize the order of results in some way so that it's not as likely you'll hit the same answer every time. Support for this isn't really fleshed out by there is an option to shuffle the answers.
 - The StackExchange API has a neat concept of filters to request only specific fields. I'm using that to significanly cut down on the amount of data that needs to be downloaded.

## Dynamic Compilation and Execution

The fun part!

Code can be found [here](https://github.com/buckley-w-david/stacksort/blob/master/stacksort/_meta/injector.py) (StackSortRunner) and [here](https://github.com/buckley-w-david/stacksort/blob/master/stacksort/_meta/compile.py)

From my previous project trying working on [automatic SQL query parameterization]({{< ref "/post/string-interpolation-and-sql" >}} "String Interpolation and SQL"), I had some experience with dynamic code generation, so I generally knew what I was doing and didn't need to go read blog posts to figure it out.

There are a bunch of small details and hueristics I needed to come up with, but at a high level this is what happens:

1. The loader from our import hooking returns a `StackSortRunner`.
2. `StackSortRunner` has a `__call__` method that iterates over code blocks return from the fetching section.
3. The code block is sent to the enigmatic `compile.compile_sorter` function, this returns a callable object.
4. The returned object is called, the unsorted list is passed to it as an argument.
5. If none of that caused an exception, a reference to that "working" object is stored, and the result of its execution is returned. Anytime the `StackSortRunner` is called in the future that "working" version will be used without performing another search.

All of the complexity here is in `compile.compile_sorter`. 
