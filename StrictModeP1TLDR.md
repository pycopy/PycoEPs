# StrictModeTLDR
Created Monday 30 November 2020

## Abstract

This proposal seeks to define "constant namespace slot" (variables/functions/methods) for Python, including both implicit and explicit (user-annotated) slots. Further, the proposal seeks to define necessary changes to language semantics to actually uphold the constant property (i.e. inability to redefine). To preserve as much as possible the benefits of Python's dynamic nature and metaprogramming it enables, the proposal introduces 2 distinct execution modes: "Import-time", which preserves full existing Python execution semantics (albeit with some warnings to point at "problem spots"), and "Run-time", where constant property of the designated name slots (including implicitly constant nameslots, like functions/methods) is enforced (attempts to assign to such slots lead to runtime exception). Overall, the new execution variant is named "strict mode". 

It is believed that these measures would allow to develop more structured and more robust Python code, especially in big codebases. At the same time, the underlying motivation for the strict mode is to ease the development of runtime optimizers for Python, in particular JIT compilers.

Support infrastructure for the strict mode is defined: means to enable it, compatibility measures with standard execution mode.

The strict mode is intended to be an opt-in feature, not replacing the default Python execution mode. 

Below is an example of strict mode program with comments explaining various features:

```
import mod


# Leads to a warning: replacing (monkey-patching) a constant slot (function) with a variable.
mod.func1 = 1

# Leads to a warning: replacing (monkey-patching) a constant slot (function).
mod.func2 = lambda: None

# Way to define a constant.
my_cnst: const = 1

# Leads to a warning: replacing (monkey-patching) a constant slot.
my_cnst: const = 2

glb1 = 100


def fun():
    # Imports are not allowed at run-time
    import mod2
    # But you can re-import module previously imported at import-time.
    import mod

    global my_cnst
    # RuntimeError
    my_cnst = 3

    # RuntimeError
    mod.func2 = lambda x: 1

    global glb1, new
    # RuntimeError: Cannot create new global nameslots at runtime.
    new = 1
    # Nor can delete existing
    del glb1

    # Cheats don't work
    globals()["new"] = 1


# Leads to a warning: replacing (monkey-patching) a constant slot (function).
def fun():
    pass


# fun_var is a variable storing a reference to a function (can store ref
# to another func).
fun_var = fun

# fun2 is an alias of fun
fun2: const = fun


# Run-time execution starts with this function. This clearly delineates
# import-time from run-time: a module top-level code is executed at
# import-time (including import statements, which execute top-level code
# of other modules recursively). When that is complete, strict mode
# interpreter switches to run-time mode (restrictions enabled) and
# executes __main__().
def __main__():
    fun()


# This statement is not executed when program runs in strict mode.
# It is executed when it is run in normal mode, and allow to have
# the same startup sequence (execution of __main__()) for both cases.
if __name__ == "__main__":
    __main__()
```
