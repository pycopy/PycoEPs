# StrictMode
Created Saturday 21 November 2020

## Contents

* [Introduction](#introduction)
* [Summary and scope](#summary-and-scope)
* [Definitions](#definitions)
* [Motivation](#motivation)
* [Introduction of import-time and run-time distinction](#introduction-of-import-time-and-run-time-distinction)
* [Level 1: No namespace attributes additions/deletions](#level-1-no-namespace-attributes-additionsdeletions)
* [Level 2: Implicit const namespace slots](#level-2-implicit-const-namespace-slots)
* [Level 3: Explicit const namespace slots](#level-3-explicit-const-namespace-slots)
* [Discussion and implications](#discussion-and-implications)
	* [Dynamic module imports](#dynamic-module-imports)
* [Support infrastructure](#support-infrastructure)
	* [Plugging the globals() hole](#plugging-the-globals-hole)
	* [Startup sequence compatibility between standard and strict mode](#startup-sequence-compatibility-between-standard-and-strict-mode)
	* [Activating strict mode](#activating-strict-mode)
* [Open issues](#open-issues)
* [Implementation](#implementation)
* [Prior art](#prior-art)
* [Trivia](#trivia)

## Introduction

This proposal seeks to introduce opt-in "strict execution mode" for Python language implementations interested in such a feature. Under this mode, some "overdynamic" language features are restricted in a reasonable way. The primary motivation for these restrictions are opportunities for simple(r) to implement runtime optimizations. However, it is also believed that the "strict mode" may also help on its own with clarity and maintenance of large(r) Python codebases.

The "strict mode" is an opt-in alternative execution mode to the standard one. The standard mode is not intended to be replaced by the strict mode. It's up to an application to decide whether it supports strict mode or not (based on requirements of its dependent libraries if course). Any program which can (correctly, without errors) run in the strict mode can also run in the standard mode without any changes in behavior.

## Summary and scope

Proposed are restrictions on modifications, at run-time, to "namespace dictionaries" of classes and modules. The baseline restriction is prohibition of addition of new attributes (and by extension, removals of them). Further extension is introduction of "const" attributes, and prohibition of their redefinition (reassignment) at the run-time. It is believed that the restrictions applied are relatively modest and can be surmounted by the interested Python programmers. Moreover, they should coincide (or being fairly close) with restrictions that some codebases may already place, and overall, would lead to improved structure and maintenance of large (and small) codebases.

This proposal explicitly does not consider attribute dictionaries of objects. It is considered to be the matter of fact that addition of new object attributes at run-time is a widely used feature in Python programs. Besides, the Python language already offers means to control that - object's class `__slots__` attribute. Namely, if it does not contain `__dict__` meta-attribute, then creation of new object attributes (beyond declared in `__slots__`) is prohibited.

## Definitions

Namespace dictionary - a (special) dictionary used to implement attributes of modules and classes. Note that each script in Python is a module ("default" module for scripts is called "__main__"). In this regard, global variables are attributes of a module (and thus stored in module's namespace dict). One way namespace dictionaries differ from normal Python dictionaries is that namespace dicts' keys are always strings. And for many implementation, they are actually interned strings (because we want to lookup attributes efficiently by a single comparison operation of a dictionary key, instead of comparing string content). 

Namespace slot - an entry in a namespace dictionary, consisting of attribute name and its value. This proposal adds extra property of a namespace slot - "constness". This property would make namespace dictionaries even more distinct from normal Python dictionaries, which don't have such property. Implementation wise, a "const" bit would be almost certainly stored in a "key" part of the dictionary entry, because a "value" part can hold an arbitrary Python value, whereas per above, the a "key" part stores only a string, so it would be "easier" to find an extra space for the "const" bit.

Import-time - timespan of program execution where imports of other modules and initialization of current module happens.

Run-time - timespan of program execution when all imports and initialization complete, and a program "runs" per se.

To avoid mixing up the (very specific) meaning of these terms with similarly sounding generic terms (like "runtime"), we write them using hyphens, like we did above.

## Motivation

The underlying motivation for the introduction of the strict mode is to avoid expensive dynamic lookup-by-name during program runtime. Such lookups inherently require operation of *searching*, and searching is inherently slow. A common topic of argument is how much searching is slower, especially how much it affects the overall program performance (of which searching for dynamic resolution of names is just one overhead). This proposal doesn't try to go in such arguments.

And if that sentence did not emphasize it enough, let us elaborate: this proposal avoids that argument completely. The proposal is based on the idea that to not be slow, you should avoid slow operations. Languages which are built with "don't perform slow searching operations for variable lookups", so-called "static languages", for example C/C++, are known to be generally fast. Of course, dynamic vs static variable/attribute lookup is not the only thing which differentiates Python from C/C++. That is why this proposal avoids speculations of how much static lookups would help in the case of Python. Instead, it tries to apply known good practices of avoiding unneeded dynamic lookups, to cross out this fairly simple problem from the problem domain, and instead expose, and let concentrate on, the next level of Python's performance issues.

Note that there is a well-known technique on how to optimize *dynamic* lookups, it's called "caching" (of the lookup results, in the assumption that most of the lookups are fairly static in nature). Oftentimes, caching is "inline" and sometimes even "polymorphic" (so, not just single result is cached, but a few). The problem of this technique is that you still need to handle the case when cache does not contain the actual information. So, you need to anticipate that possibility. Then you need to act properly on it. Then you need to test it. Then you need to maintain it over time. If you have many/complex lookups, then it all multiplies. And then even if you did all that (nobody seems to have done that too completely and too well in the case of Python, for example), all the guards and support code required for fallbacks still takes space in your instruction caches, causing thrashing of it, so it all still runs slower than it could.

This proposal instead looks for a way to add "much more static lookups", as an alternative to proliferation of dense inline caches and associated guards and fallback code.

## Introduction of import-time and run-time distinction

Hey, but everyone loves Python's dynamic typing nature! We love meta-programming, monkey-patching, and some of us even consider one of the darkest days of Python when it lost the ability to do `True = False`.

Fear not! The author of this proposal is like every other Python user loves all the things above! However, this proposal is based on the observation that all these things are mostly useful if done at a particular stage of program lifecycle. Specifically, at the "initialization" time. That's the good time to use meta-programming to generate constants. To monkey-patch other modules and even the language runtime itself, be it redefinition of `True` or `print()`. However, beyond that point, doing all those things are much less useful, because it's much harder to reason about the program behavior and/or test it.

Given that all Python code is structured as modules, the "initialization" time can be defined as "module importing time", or "import-time" for short. Let's trace how it works: the interpreter starts with executing the top-level code (i.e. code not inside any function) in the `__main__` module. It imports other modules, which have their top-level code executed, which imports other modules recursively. When this process completes, the "import-time" is over, and "run-time" begins.

The problem of the currently available execution mode of the Python is that it does not provide means to clearly delineate import-time from run-time. That's because the "import" statements are located at the same top-level codespace as "run" code. There're some conventions to delineate import-time from run-time code, e.g. the well-known `if __name__ == "__main__"` idiom. Indeed, the code under that `if` statement is not supposed to be executed if the module is imported, so it clearly delineates run-time code. However, this distinction is neither universal, not distinctive enough from the point of view of the interpreter (it's just another statement among many in the same codespace).

Effectively, with the current Python execution model, all code is executed at "import-time". This proposal seeks to introduce well-defined distinction between import-time vs run-time. For this, it introduces the `__main__()` function, located in the `__main__` module of the application (from which application execution starts). All top-level code in the `__main__` module is executed to completion, including code in recursively imported modules. This completes the "import-time" phase of the application. After that, the `__main__()` function is executed, which begins the "run-time' phase.

The semantics of attribute *stores* is different between the import-time and run-time. At import-time, all the currently available namespace dictionary operations are available. However, we take a chance to warn about some "suspicious" operations. But at run-time, some attribute store operations are restricted. In the next sections, we discuss 3 progressive levels of restrictions.

To give a wider perspective of the change proposed, it is a case of [Multi-stage Programming](https://en.wikipedia.org/wiki/Multi-stage_programming). Note the linked Wikipedia article is effectively a stub, and heavily biased toward original research of a particular author, in a particular programming environment (a functional language). It is thus full buzzwords intended to "sell" that particular paradigm. The only generic part of it is "Multi-stage programming (MSP) is a variety of metaprogramming in which compilation is divided into a series of intermediate phases".

Note that all discussion in this section applies only to the strict mode. "Standard mode" continues to be available, and none of the changes discussed in this or further sections affect it.

## Level 1: No namespace attributes additions/deletions

The basic idea is to disallow addition of new namespace attributes. The reason why this may be useful is because, as explained above, namespace attributes are stored in dictionaries, which have limited capacity. If the current capacity is exceeded, a dictionary needs to be reallocated with higher capacity and its contents rehashed. If we disallow growing namespace dictionaries, then the slot for a given attribute will be always at the given position. Additionally, for non-moving garbage collectors, the memory address of the slot will never change. That means that instead of dictionary searching operation for a particular attribute, we can look up it once (at the beginning of run-time, when the restriction on addition of new attributes is applied), and store a reference (position, or even direct memory added) to the actual namespace slot instead.

For symmetry, it also makes sense to disallow deletions of namespace attributes. Similar to addition, deletion of entries from dict could lead to its shrinking, and changing of existing slot positions. Even if it does not, "deleted" keys would need to be marked with a special sentinel value, and then there would be one extra guard to access a namespace slot. As was mentioned, one of the motivations of the strict mode is to reduce number of guards required to JIT Python to a reasonable amount. 

## Level 2: Implicit const namespace slots
 
Under level 1, we can avoid lookups-by-searching. But we still need to lookup current value in the (now static) namespace slot. However, many of the objects in the namespace slots are never changed. For example, majority of the functions, classes, methods are never overridden once they are defined. To capitalize on this observation, we introduce concept of "const" namespace slots, and stipulate that some namespace slot definitions implicitly produce const slots. As was mentioned, this is true for modules, classes, and functions (methods are just functions in a class namespace).

During import-time, redefinition of const slot leads to a warning. This would particularly happen when monkey-patching a function/class in other module. Like any warning in Python, it could be disabled (e.g. when a particular case of monkey-patching is part of application "business logic"). During run-time, attempt to assign a value to const namespace slot leads to RuntimeError.

The idea behind const slots is that we can avoid extra memory indirection to access its value. We literally can (at run-time) propagate const slots values to where they are used, which oftentimes lead to the cheapest possible "immediate" values. For example, normally `foo(a, b)` would be executed as: take a value of the `foo` namespace slot, and call that value. If `foo` is a const slot, we can call foo's code directly. Not only that, we also know which exactly `foo` is being called, and so for example can optimize argument preparation/passing sequence.

## Level 3: Explicit const namespace slots

Level 2 introduced concept of implicitly const namespace slots (for functions, classes, modules). Level 3 extends it by allowing to mark any namespace slot as const. This is achieved by using variable annotations:

`my_const: const = 1`

This functionality allows for example to express difference between the following cases:

```
# "foo" is implicitly const name
def foo():
	pass

#"bar" is a variable holding a reference to function, initialized with "foo",
# but can be reassigned later.
bar = foo

# "baz" is an alias of "foo".
baz: const = foo
```


Note that the discussion here concerns namespace dictionaries and their slots. "const" annotation definitely could be applied to local function variables too. But this restriction would be handled in the compiler, and fully at compiled. As this proposal focuses of runtime support for namespace dictionaries, we don't discuss local function variables further.

## Discussion and implications

The baseline implications should be fairly obvious from the text above:

```
def fun():
    pass

# This (defining the function of the same name) will lead to warning
# at import-time.
def fun():
	pass

import mod

# This (assigning to a var in another module) is possible
# both at import-time and run-time.
mod.var = 1

# This will lead to warning at import-time and RuntimeError at runtime.
mod.func = lambda *a, **kw: print("Stubbed out!")
```

Does it mean that you cannot override function behavior at runtime? No, you just would need to do that the way it would be done in a "generic programming language":

```
import mod

org_func = mod.func

def my_func():
    ...

def overriden_func():
    if (flag):
        my_func()
    else:
        org_func()

mod.func = overriden_func
```


Or, using a "function pointer":

```
import mod

# "func_ref" is a variable holding a reference to function.
# You can reassign it at any time.
func_ref = mod.func

def overriden_func():
    # Call function by pointer
    func_ref()

mod.func = overriden_func
```

Of course, if you know Python and read the proposal above thoroughly (so know that semantics at import-time fully matches existing semantics (albeit with a few warnings added), you could short-circuit the above code to below. But that's **not** how you would do it in a generic programming language, so use with care:

```
import mod

# "func_ref" is a variable holding a reference to function.
func_ref = mod.func
# We override a const function slot in another module with a variable.
mod.func = func_ref
# Of course, if you're smart, you can do:
# mod.func = mod.func

# Now we can override mod.func at any time at run-time. (Because it's
# no longer a const function slot, but just a variable.)
def foo():
     mod.func = overriden_func
```

These last example should provide insight at what implicit const namespace slots of the strict mode seek to achieve. Per the legacy Python semantics, all function names are actually variables, holding a reference to function code, which can be reassigned at any time. But reassignment happens maybe in 1% of cases. Strict mode switches the default to: function names are constant references to function code, which is what's used in 99% of cases (and can be capitalized on for various optimizations). If you have a case which fits that 1% of cases (and you don't want to follow the approach of "a generic programming language"), you can reset slot to be a variable again by assigning to it, as a variable (and this assignment can be as simple as identity assignment). 

### Dynamic module imports

By far, the most "grave" issue with the strict mode is that dynamic (at run-time) imports are not supported:

```
def foo():
    # Leads to RuntimeError.
    import mod
```

Let us walk thru why: at run-time, we perform an import. A new module object is created and module top-level code starts executing. What this code would usually do is to create some variables/functions/classes in the module's namespace dictionary, but that's exactly what's prohibited in the strict mode!

At this time, this is considered a forward-looking feature. To perform any non-trivial optimization on dynamic languages, whole-program analysis is required. So, the strict mode is kind of naturally upholds that requirement.

Don't get me wrong - I absolutely love dynamic imports! As an example, the strict mode was implemented in the Pycopy dialect of Python, and of 5 not completely trivial applications written for Pycopy, 5 use dynamic imports. So, it is very important to consider what to do about them.

First of all, if we use an "import" statement with statically known module name, we can move the import to the top level. This is also considered the best practice in many codestyles. A slightly advanced compiler can do that automatically, i.e. it would transform the example above to the equivalent of:

```
import mod as _unique123

def foo():
    mod = _unique123
```

(Such a transform would rely on the fact that there's no "grave" side effects are performed on the module import per se. Actually, one of the reasons to perform an import like above dynamically at runtime is if it's known that import is "heavy", e.g. takes long time, allocates too much memory, performs network connections, etc. But the known best practice is to avoid that, and keep module top-level as simple and side-effect free as possible. If module requires initialization, instead add explicit `init()` function. Again, the strict mode would only motivate people to apply this best practice.)

However, that would not help with 5 of my own cases, because I use truly dynamic imports using `__import__()` or wrapper modules from stdlib. And in at least 2 cases, the module being imported actually depends on command line parameters. Let us consider what to do with them.

Given that imports are not allowed at run-time, there is no other choice than to perform them at import-time. I.e., the code should which performs dynamic imports should be refactored out, to allow it to be called yet at import-time. Then these modules can either be stored in (const) variables, and the rest of code refactored to use them. Or - we don't really need to change the rest of code, because if a module is already imported, it can be re-imported in another scope (including at run-time). To demonstrate that on the existing example: 

```
import mod

def foo():
    # This works, as the module is already imported.
    import mod
    # This works too.
    m = __import__("mod")
```

If which module is loaded depends on command-line arguments, then apparently we need to parse command-line arguments at import-time too. The hardest case is a "set of plugins" designs, where a user can select at runtime an arbitrary module to be loaded. In this case, we would need to e.g. designate a particular directory for plugins, and pre-import all modules from it, in the anticipation that user may later call any of them.

So, while dynamic imports definitely pose a challenge for the strict mode, and require re-architecturing and refactoring of the application initialization sequence, they are not insurmountable. Once a few refactorings are made, it should lead to understanding of how to write applications compatible with the strict mode "from scratch". And it should be mentioned that for highly dynamic application, running in standard Python mode remains a valid option.

There is actually yet another option to tackle dynamic imports problem - to ease restrictions on namespace attribute creation during import time, for the module being imported. It would work like follows: when import processing is started, the namespace dict of the module is marked as allowing additions/deletions (the rest of namespace dicts remain sealed, i.e. a new module cannot add attributes to already loaded modules, or change const attributes). When import of a module is finished, its namespace dict is sealed like the rest. The issue here is that imports can be recursive, so there can be multiple active unsealed namespace dicts, and e.g. "child" module can change namespace of a module it's imported from. But if stack-like sealing policy is applied (as described above), parent module won't be able to patch namespace of a child module once it's imported. Which is different from how normal import-time semantics work. So, maybe it would make sense to employ another, more complex, sealing policy. But all this discussion shows that this would be much more complex handling, with various corner cases to consider. But the whole motivation behind introducing the strict mode is to allow to more easier develop simple JIT engines for Python. In that regard, closed-world whole-program approach implicitly upheld by the original strict mode formulation is quite a simplifying approach, allowing for highest level of optimizations, which yet does not pose insurmountable restrictions on the application functionality. Due to these reasons, extensions described in this paragraph are outside of the strict mode scope as defined in this proposal.

## Support infrastructure

### Plugging the globals() hole

globals() at run-time have obvious means to subvert namespace preservation semantics built so far. At the same time, it's useful at import-time - it's part of Python meta-programming infrastructure. So, there is no talk of banning it completely from strict mode. Instead, it should be updated to apply the same restriction as described above (at run-time: disallow to create new entries and delete existing, disallow to assign to const keys). One way to achieve that is to implement a "proxy dict" object which would enforce needed restrictions. This is an approach similar to how locals() is implemented (except that locals() says: "The contents of this dictionary should not be modified; changes may not affect the values of local and free variables used by the interpreter.", whereas here we say "it's an explicit error to assign to non-assignable slot, etc.")

It should be noted that apparently, locals() behavior should not be modified: it can return module or class namespace dict only when executed at import-time, where we don't apply restrictions on their modification. At run-time, locals() would be run only in function scope (so return real locals, and that is not a namespace dictionary per our definition).

The reference implementation in Pycopy does not currently implement a proxy dict object. Instead, it reuses existing facility to mark (any) dict as "fixed" (i.e. read-only). This applies more restrictions than described above: no modifications are allowed via globals() (even for non-const slots), only "read" operations are allowed. This is in line with the overall intent of the strict mode: to allow all existing Python meta-programming facilities at import-time, but disallow/discourage their usage at run-time. (For example, there may be no "namespace dict" at run-time at all, it all could have been optimized out. Then allowing to use globals() at run-time means such a dict should be maintained solely for its usage.)

### Startup sequence compatibility between standard and strict mode

As discussed above, the strict mode application runs the `__main__()` function. It's a RuntimeError if the main module doesn't define such a function. With that in mind, it would be nice achieve compatibility with standard mode, allowing the same application run under both. A common idiom in Python is something like:

```
if __name__ == "__main__":
    main()
```
 
So, we'd just need to rename the main function from `main()` to `__main__()`. But the code above actually poses bigger problem for the strict mode: it would be executed during import time, and the main function would be also run at import-time, not at run-time. Looking at the above code, it is clear what must be changed: in the strict mode, the name of the main module must be different from the usual `"__main__"`. In the current implementation in the Pycopy dialect it's set to `"__main__s"`. Following are apparent requirements for this name:

* It should start with an underscore, so it was not imported by `from mod import *`.
* It should start with `"__main__"`. There should be a way to run some code at import-time for the main app module (but not if its imported in another module), and this way should be compatible with both standard and strict modes. A natural way is to update the existing idiom `if __name__ == "__main__":` to `if __name__.startswith("__main__"):`.

### Activating strict mode

To activate a strict mode of execution, the primary means should be a command-line option to the interpreter. Assuming the strict mode would get traction, something like `--strict` would be warranted, but before that, an existing `-X` namespace is a good candidate, e.g. `-X strict` (implemented in Pycopy).

Besides that, it would be nice to let application purposely developed to run in strict mode, to activate it. For that, and existing idea with "pragma comments" can be used, similar to existing `# encoding: utf-8` which must appear in the first few lines of the source. `# mode: strict` pragma comment could activate the strict mode, however for simplicity (and while strict mode semantics is not fully fixed), Pycopy implements `#-X strict`.

One could pose a question - why need to activate the strict mode explicitly? Why not look if there's a `__main__()` function is defined, and if so, run the application in the strict mode? Sadly, there is a chicken-and-egg problem due to compatibility with standard mode considerations described above. Specifically, to avoid code  `if __name__ == "__main__" `running, the name of module should be set before starting to run top-level module code, i.e. we need to know that strict mode is needed before that. But we can reliably lookup presense of `__main()__` function only after the top-level code finished its running. After all, it's the top-level code which actually creates that function in Python. Mind that this function can be defined as:

```
globals()["".join("__", "main", "__")] = lambda: call_another()
```

We can workaround that but requiring that the `__main__()` to be defined as `def __main__():` at the AST level, or, much worse, try to add extra interpration to bytecode, trying to catch the execution of the `if __name__ == "__main__"` statement.

But remember that the whole idea of the strict mode is simplicity. Having explicit switch for strict mode is the way to achieve simplicity and explicit control. And the pragma comment to enable the strict mode on the application side should alleviate "production" usage usability concerns.

## Open issues

Currently, there is no clearly designated way to create const slots programmatically (at import-time). This in definitely useful, consider e.g.:

```
for i, n in enumerate(["ONE", "TWO"]):
	globals()[n] = i
	# Would like to mark the created value as const here 
```

This issue can be extended to the general question of "how to apply (variable) annotations programmatically in Python." 

Terminology: yet to be fully established. For example, alternative to "import-time" could be "load-time". "Import-time" is more specific term in the context of Python, and that's why it was chosen. "Load-time" on the other hand is more intuitive antonym of "run-time". "Run-time" is itself not ideal, as it's easy to mix it up with "runtime", which is more general term (effectively, "run-time" is a particular runtime execution discipline, as established by the strict mode). A possible alternative is less clear, perhaps something like execution-time, though that is not without its issues too (execution also happens at "import-time").

## Implementation

The strict mode, as described in this proposal, is implemented in the version 3.4.0 of the "Pycopy" Python dialect, https://github.com/pfalcon/pycopy.

There also was an attempt to prototype strict mode implementation for CPython in pure Python. This prototype code, no workability guarantee, is provided for reference in https://github.com/pfalcon/python-strict-mode . While working on that prototype, it was found that a kind of "Catch-22" applies to CPython: As a background, it is know to contain many overdynamic features, and thus hard to optimize. One would think that these overdynamic features at least allow to *prototype* a solution which allows to limit this overdynamicity, e.g. as described in this proposal. But here is the mentioned catch-22 kicks in: CPython has too much overdynamicity already, but not enough of it, to put this overdynamicity under control ("virtualize" it). Insightful as usual communication with the core dev team is available in [[https://bugs.python.org/issue36220.]] The rejected patch to address that issue is at [[https://github.com/python/cpython/pull/18033.]]

It would be still possible to implement strict mode for CPython, but that would require to "virtualize" much more of its existing execution infrastructure, e.g. handling bytecode compilation in a custom way and/or post-process the code objects. That is much more effortful task, without readily available infrastructure, than the approach initially taken by the prototype (just wrap namespace dictionary in the proxy objects), so further experimentation in that direction was put on hold.

## Prior art

The background of ideas presented in this proposal is anything but new. This background can be decomposed into:

1. Ideas to treat some language "constructs" as possessing implicit const(-like) traits.
2. Explicit support for "constants" (either surface, syntactic-only, or also attempts to implement semantics).

Regarding first category:

There always been ideas on optimizing (clearly inefficient, to anyone scrutinizing them) "global" variables (and function) access. In 2001-2002, that actually reached level of PEPs (in the order of appearance):

* PEP 267 -- Optimized Access to Module Namespaces (2001-05)
* PEP 266 -- Optimizing Global Variable/Attribute Access (2001-08)
* PEP 280 -- Optimizing access to globals (2002-02)

But this prolific activity led to counter-effect, with proposals deadlocking each others, with PEP 280 (written by BDFM!) deferral notice reads: "While this PEP is a nice idea, no-one has yet emerged to do the work of hashing out the differences between this PEP, PEP 266 and PEP 267. Hence, it is being deferred."

Regarding second category:

There were various 3rd-party modules implementing "constants" over time. Many of them were just "syntactic sugars" for defining (global or class-scoped) variables, something which eventually landed as "enum" module in the stdlib. Some actually tried to enforce "constantness" with variable success. Random selection of links:

* https://code.activestate.com/recipes/65207-constants-in-python/
* https://aroberge.blogspot.com/2020/03/true-constants-in-python-part-1.html

Since introduction of annotations in Python, they started to be used, e.g. `Final` annotation, which eventually lands in the stdlib `typing` module. As any other annotation, it is ignored on the *language* level, and internal tools/modules are required to actually enforce/take advantage of it.

The voices were heard that it would be nice to first define, then start to gradually honor more and more a constant annotation on the language level. In particular, it was pointed out that many other languages already have it, and various direct Python competitors (JavaScript, Lua) recently acquired it too:

* https://mail.python.org/archives/list/python-dev@python.org/message/WV2UA4AKXN5PCDCSTWIUHID25QWZTGMS/
* https://mail.python.org/archives/list/python-ideas@python.org/thread/SQTOWJ6U5SGNXOOXZB53KBTK7O5MKMMZ/ 

This proposal rehashed the previous ideas and tries to integrate them into coherent implementation approach with the aims:

* Actually enforce constant property at runtime.
* Don't require const annotations for the most common cases, while introducing support for it.
* Tighten up semantics of the language in the direction already desired and explored by larger codebases.
* Target this feature-set towards optimizing named variables/methods lookup, which would allow to implement simpler JIT techniques (and concentrate on more important performance matter).

## Trivia

This proposal refers to the opposite of the "strict mode" as "standard mode". It may be however considered that that name is not vivid enough. It may be due to pay the tribute to a spiritual instigator of it, and call it "Smalltalk mode". This will at once ring the epiphany of the strict mode: people who want to program Python as if it were Smalltalk, may continue to do so (strict mode is an opt-in feature). The rest of programmers may explore possibilities which the strict mode offers or inspires.
