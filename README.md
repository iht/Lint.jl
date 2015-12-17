# Lint.jl

[![Package Evaluator](http://pkg.julialang.org/badges/Lint_release.svg)](http://pkg.julialang.org/?pkg=Lint&ver=0.3)
[![Build Status](https://travis-ci.org/tonyhffong/Lint.jl.svg?branch=master)](https://travis-ci.org/tonyhffong/Lint.jl)
[![Coverage Status](https://img.shields.io/coveralls/tonyhffong/Lint.jl.svg)](https://coveralls.io/r/tonyhffong/Lint.jl)

## Introduction

Lint.jl is a tool to hunt for imperfections and dodgy structures that could be improved.

## Installation
```julia
Pkg.add("Lint")
```

## Usage
```julia
using Lint
lintpkg("PackageName")
lintfile("your_file.jl")
lintstr("code")
```
It will follow any `include` statements.

The output is of the following form:
```
filename.jl:Line CODE variable: message
```
`filename.jl` is the file that contains the problem.
`Line` is the line number relative to the start of the file.
`CODE` identifies the problem and gives an indication of severity.
Starts with letter for the severity `E`:`ERROR`, `W`:`WARN` or `I`:`INFO` then has 3 numbers identifying the error.
`variable` is the variable causing the error.
`message` is an explanation of the error.

If your package always lints clean, you may want to keep it that way in a test:
```julia
using Test
using Lint
@test isempty(lintpkg("MyPackage", returnMsgs=true))
```

## What can it find?
* simple deadcode detection (e.g if true/false)
* simple premature-return deadcode detection
* Bitwise `&`, `|` being used in a Bool context. Suggest `&&` and `||`
* Declared but unused variables or arguments. Overruled by adding the no-op statement `@lintpragma("Ignore unused [variable name]")`
  just before the warning line.
* Using an undefined variable
* Duplicate key as in `[:a=>1, :b=>2, :a=>3]`
* Mixed types in uniform dictionary `[:a=>1, :b=>""]` (ERROR)
* Uniform types in mixed dictionary `{:a=>1, :b=>2}` (performance INFO)
* Exporting non-existing symbols (However, it cannot catch symbols generated by macros)
* Exporting the same symbol more than once
* Name overlap between a variable and a lambda argument
* Implicitly-local (local keyword not provided, but neither is global)
  variable assignment that looks like it should be a global variable
  binding, if the name is long enough
* Assignment in an if-predicate, as a potential confusion with `==`
* warn `length()` being used as Bool, suggest `!isempty()`
* Out-of-scope local variable name being reused again inside the same code block. (legal but frowned upon)
* Function arguments being Container on non-leaf type e.g. `f(x::Array{Number,1})`. Suggest `f{T<:Number}(x::Array{T,1})`
* Concatenation of strings using `+`. It also catches common string functions, e.g. `string(...) + replace(...)`
* Iteration over an apparent dictionary using only one variable instead of (k,v) tuple
* Incorrect ADT usage in function definition, e.g. `f{Int}(x::Int)`, `f{T<:Int64}(x::T)`, `f{Int<:Real}(x::Int)`
* Suspicious range literals e.g. `10:1` when it probably should be `10:-1:1`
* Understandable but actually non-existent constructors e.g. String(), Symbol()
* Redefining mathematical constants, such as `e = 2.718`
* Illegal `using` statement inside a function definition
* Repeated function arguments e.g. `f(x,y,x)`
* Wrong usage of ellipsis in function arguments e.g. `f(x, y... , z...)`
* Wrong usage of default value in function arguments e.g. `f(x=1, y)`
* Named arguments without default value e.g. `f(x; y, q=1)`
* Code extending deprecated functions, as defined in deprecated.jl (Base)
* Mispelled constructor function name (when it calls `new(...)` inside)
* Constructor forgetting to return the constructed object
* calling new() with the number of arguments different from the number of fields.
* Rudimentary type instability warnings e.g. `a = 1` then followed by `a = 2.0`. Overruled using a no-op statement
  `@lintpragma("Ignore unstable type variable [variable name]")` just before the warning.
* Incompatible type assertion and assignment e.g. `a::Int = 1.0`
* Incompatible tuple assignment sizes e.g. `(a,b) = (1,2,3)`
* Flatten behavior of nested vcat e.g. `[[1,2],[3,4]]`
* Loop over a single number. e.g. `for i=1 end`
* More indices than dimensions in an array lookup
* Look up a dictionary with the wrong key type, if the key's type can be inferred.
* assign to a variable that happens to have the same name as the host function.
* Inform upcoming 0.3->0.4 changes in Dictionary and Array construction
* Warn [1 :end], or [end -1]
* If the first statement in a true branch of an if-statement is proven to be a boolean expression,
  warn for potentially missing `||` or `&&`
* A type field with undeclared type or array type without dimension (performance INFO)
* Constant global without assigning a value to it.
* Staged function: catch one forgetting that the arguments are all types. Inform unused arguments in both the
  staged function's logic as well as the quoted code.
* Catch silly curly bracket misuse/abuse e.g. Dict{:Symbol, Any}
* Understands and checks basic `@doc`
* Catch assigning `a.b = something` when `a` is proven to be of an immutable type, e.g. `Complex`
* Follow VERSION if-statements to properly lint based on version-appropriate standard

## @lintpragma: steering Lint-time behavior
You can insert @lintpragma to suppress or generate messages. At runtime, @lintpragma is a no-op, so it gives
no performance penalty.
However, at lint-time, these pragmas steer the lint behavior. Module designers do not even have to import
the macro from Lint.jl in their module, as long as they just create an empty macro like this, early in their
module scripts:
```julia
macro lintpragma(s)
end
```

Lint message suppression (do not include the square brackets)
* `@lintpragma("Ignore unused [variable name]")`. Works for unused arguments also.
* `@lintpragma("Ignore unstable type variable [variable name]")`. Ignore type instability warnings.
* `@lintpragma("Ignore deprecated [function name]")`
* `@lintpragma("Ignore undefined module [module name]")`. Useful to support Julia packages across 
    different Julia releases.
* `@lintpragma("Ignore untyped field [field name]")`.
* `@lintpragma("Ignore dimensionless array field [field name]")`. Useful if we really want to store 
    arrays with uncertain/runtime-calculated dimension
* `@lintpragma("Ignore use of undeclared variable [variable name]")`. Useful when using macros to
  generate symbols on the fly.
* `@lintpragma("Ignore incompatible type comparison")`. Useful to silence deliberately different-type comparison

Lint message generation (do not include the square brackets)
* `@lintpragma("Info type [expression]")`. Generate the best guess type of the expression during lint-time.
* `@lintpragma("Info me [any text]")`. An alternative to-do.
* `@lintpragma("Warn me [any text]")`. Remind yourself this code isn't done yet.
* `@lintpragma("Error me [any text]")`. Remind yourself this code is wrong.

The macro also supports lint-time terminal output that generates no Lint message:
* `@lintpragma("Print type [expression]")`. Just print out the type
* `@lintpragma("Print me [any text]")`. Lint-time printing

Useful version tracing tool
* `@lintpragma("Info version [version]")`. lint-time version reachability test

## VERSION branch

As julia evolves, some coding style that is an error becomes valid (and vice versa). It is common to use
VERSION if-statements to implement cross-version packages. As long as the if statement is simple,
Lint can pick them up and suppress version-dependent errors that are not reachable in the current version.

Examples:
```julia
# lint won't complaint about missing `Base.Dates` in 0.3 or missing `Dates` in 0.4
if VERSION < v"0.4-"
    using Dates
else
    using Base.Dates
end
```

```julia
# this passes lint in 0.3 but it generates an INFO in 0.4
s = symbol("end")
```

```julia
# this is an error 0.3 but it passes in 0.4
s = Symbol("end")
```

```julia
# this will lint clean cross versions
if VERSION < v"0.4-"
    s = symbol("end")
else
    s = Symbol("end")
end
```

You can directly test for version reachability by inserting lint-pragmas
like so
```julia
if VERSION >= v"0.4-"
    @lintpragma("Info version 0.3")
    @lintpragma("Info version 0.4.0-dev+1833")
else
    @lintpragma("Info version 0.3")
    @lintpragma("Info version 0.4.0-dev+1833")
end
```
You will see line-by-line reachability in your output. See test/versions.jl
for further examples.

## Current false positives
* Because macros can generate new symbols on the fly. Lint will have a hard time dealing
with that. To help Lint and to reduce noise, module designers can add a
`lint_helper` function to their module.

## Module specific lint helper(WIP)
Key info about adding a `lint_helper` function in your module
* You don't need to export this function. Lint will find it.
* It must return true if an expression is part of your module's
  enabling expression (most likely a macrocall). Return false otherwise
  so that Lint can give other modules a go at it. Note that
  if you always returning true in your code you will break Lint.
* `lint_helper` takes two argument, an `Expr` instance and a context.
  - if you find an issue in your expression, call `Lint.msg(ctx, code, variable, "explanation")`
* typical structure looks like this
```julia
function lint_helper(ex::Expr, ctx)
    if ex.head == :macrocall
        if ex.args[1] == symbol("@fancy_macro1")
            # your own checking code
            return true
        elseif ex.args[1]== symbol("@fancy_macro2")
            # more checking code
            return true
        end
    end
    return false
end
```

## Advanced lint helper interface information
* the context argument `ctx` has a field called `data` typed `Dict{Symbol,Any}`.
 Access it using `ctx.callstack[end].data`.
 Use it to store anything, although you should be careful of colliding
 name space with other modules' `lint_helper`. It is particularly useful
 for storing current lint context, such as when a certain macro is only allowed
 inside another macro.
* If your macro generates new local variables, call this:
```julia
ctx.callstack[end].localvars[end][varsymbol] = ctx.line
```
* If your macro generates new free variables (not bound to a block scope), call this:
```julia
ctx.callstack[end].localvars[1][varsymbol] = ctx.line
```
* If your macro generates new functions,
```julia
push!(ctx.callstack[end].functions, funcsymbol)
```
* If your macro generates new types,
```julia
push!(ctx.callstack[end].types, roottypesymbol)
```
You just need to put in the root symbol for a parametric type, for example
`:A` for `A{T<:Any}`.

## Lintserver feature

Make Julia start listening on a given port and return lint messages to requests on that connection.
This feature is useful when you want to lint julia code in a non julia environment (e.g. an IDE like Sublime Text).

The protocol for the server is:
1. The file path followed by a new line
2. The number of bytes of code being sent followed by a new line
3. The actual code

The server will respond with the messages produced when linting the code followed by a empty line (i.e. if it linted cleanly it will respond with a single newline).

Launch the server:
```julia
using Lint
lintserver(2222)
```

Connect and send requests:
```julia
socket = connect(2222)

println(socket, "none") # filename

str = """
test = "Hello" + "World"
"""

println(socket, sizeof(str)) # bytes of code
println(socket, str) # code

response = ""
line = ""
while line != "\n"
    response *= line
    line = readline(socket)
end

@assert response == "none:1 E422 : string uses * to concatenate\n"
```

Note that the first request might take some time because the linting functions are being imported and compiled.
