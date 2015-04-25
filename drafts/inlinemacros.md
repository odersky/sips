---
layout: sip
title: Inline Definitions and Macros
---

__Martin Odersky__
__Eugene Burmako__

__April 2015__

# Inline Definitions and Macros

## Motivation ##

## Inline Modifiers

We introduce a new reserved word: `inline`. `inline` can be used as a modifier for:

 - concrete value definitions, e.g.

        inline val x = 4

 - concrete methods, e.g.

        inline def square(x: Double) = x * x

The previous rule of regarding a `final val` with no explicit type as
a compile-time constant (inherited from Java) is dropped. Instead we
write such `val`s now as `inline`.

Value and method definitions labeled `inline` are effectively final;
they cannot be overridden.  Inline members also never override other
members. Instead, every inline member becomes an overloaded alternative
of all other members with the same name. Normal overloading resolution
is applied to pick an inline member over some other member.

If an inline definition doesn't explicitly specify its result type,
the result type gets inferred according to the usual rules.

Inline definitions only exist at compile time; no storage is allocated for them in object layout
and no code is generated for them in object method tables.
That means that it is OK to have an inline member that 
has the same type erasure as some other member with the same name.

## Inline Classes

Used as a class modifier, inline makes a class a value class. So

    inline class C(val x: Int)

is equivalent to:

    class C(val x: Int) extends AnyVal

In fact we consider deprecating the second syntax.

Used for an implicit class, inline applies to both the class and the constructor method. So,

    inline implicit class Decorator(val x: T) { ... }

is equivalent to

    inline class Decorator(x: T) { ... }
    inline implicit def Decorator(x: T): Decorator = new Decorator(x)

## Inline Reductions

The compiler will do the following rewritings when encountering certain patterns of code
outside the bodies of inline values and inline methods.
When these patterns of code appear inside the aforementioned constructs, 
they will be ignored by the inliner and processed by the compiler as usual.

1. If `prefix.f[Ts](args1)...(argsN)` refers to a fully applied inline
method, where the prefix `prefix` and all arguments `argsI` for by-value
parameters are side-effect free path expressions or values of inline types, 
replace the expression with the method's right-hand side, where parameter references are replaced 
by corresponding arguments and references to the enclosing `this` are replaced 
by `prefix`. 

   If `prefix` is not a side-effect free path or a value of inline type, lift it out to
   
         val x = prefix; x.f[Ts](args1)...(argsN)
         
   and continue with the rewriting. Do the same for all arguments that passed to
by-value parameters and that are not side-effect free paths. Arguments to by-name
parameters are left alone.

   These transformation are intended to preserve the semantics
of function applications under inlining: A function call should have the same
semantics with respect to side effects independently on whether the function was made `inline` or not.

   After the rewriting is complete, the result of the rewriting is forcibly upcast
to the type of the original expression. This mechanism works exactly the same 
as the mechanism of blackbox macro expansion in Scala 2.10+.

   The rewriting is done on typed trees. Any references from the function
body to its environment will be kept in the rewritten code. Accessors
will be added as needed to ensure that private definitions can be
accessed from the callsite.

   The rewriting is done in the "outside in" style, i.e. calls to inline methods
are expanded before possible calls to inline methods in their prefixes and arguments.
This is different from how the "inside out" style of macro expansion in Scala 2.10+, 
where prefixes and arguments are expanded first. The old style of macro expansion
can, if necessary, be emulated by the new style of inline rewritings.

2. If `prefix.f[Ts](args1)...(argsN)` refers to a partially applied inline
method, an error is raised. Eta expansion of inline methods is prohibited.

3. A conditional expression `if (c) t else e` where `c` has type `inline Boolean`
is replaced by `t` if `c == true` and by `e` otherwise.

   Note: We might also decide to reduce `match` expressions with inline
selectors and inline patterns. While useful, this would be much harder
to spec and implement than conditionals.

4. A field selection of an inline class with an inline argument selects the argument: E.g.,
`new C(42).x` rewrites to `42`.

## Inline Types

We introduce `inline T` as a new form of type. Values of type `inline T` are values
of type `T` that are statically known to the compiler _at the point where the containing code is expanded_. Constant types are subtypes
of inline types, and inline types are subtypes of their underlying types. E.g.

    42  <:  inline Int  <:  Int

    (Float => inline Boolean)  <:  inline ((inline Float) => (inline Boolean)) <:  inline ((inline Float) => Boolean)

The usage of inline types is restricted. Inline types can only appear in the following
positions:

 - As parameter or result type of an inline method:

        inline def square(x: inline Int): inline Int = ...

        inline def power(x: Int, n: inline Int): Int = ...

 - As parameter type of an inline class:

        inline class C(inline val x: Int) { ... }

 - As type of an inline val -- in fact it is automatically inferred there, i.e.

        inline val x: T = E

   is regarded as equivalent to

        inline val x: inline T = E


 - As the right hand side of an (unparameterized) type alias:

        type IS = inline String

 - As a type parameter on another inline type:

        inline List[inline Int]

        inline ((inline Int, inline Int) => inline Boolean)

 - As a refinement type of another inline type:

        inline Cell { def elem: inline Int }

 - As the type of a type ascription:

        (42: inline Int)

Inline types can specifically not be used as

 - types of abstract methods or fields
 - types of vars
 - types of parameters to non-inline classes
 - type parameters to methods
 - type parameters in `new` expressions or class parents.

## Macros

A macro expression is an expression of the form

    macro { ... }

where `{ ... }` is some block of Scala code, called macro body. (In fact, `macro` may prefix arbitrary expressions,
but blocks are used most commonly). 

In a macro expression, `macro` stands for a function 
`def macro[T](body: implicit scala.meta.macros.Context => scala.meta.Term[T]): T = ???` declared in `scala.Predef`.
Description of the functionality exposed by the `scala.meta` metaprogramming library 
is outside of the scope of this proposal.

Macro expressions can appear both in the bodies of inline methods
(then their expansion is going to be deferred until these methods expand) and in normal code
(in that case, expansion will take place immediately at the place where the macro expression is written).

Macro bodies can only reference the following names in their environment:

 1. type parameters of enclosing inline methods and classes/traits containing an enclosing inline method as a member,
 2. terms parameters of enclosing inline methods (where the parameter may, or may not be, marked inline),
 3. `this` references to classes/traits containing an enclosing inline method as a member,
 4. inline values,
 5. inline methods,
 6. anything that's global (i.e. a class/trait/object without an outer reference).

Names referenced in macro bodies undergo certain changes. References to inline and global definitions are left untouched,
but local terms of type `T` are transformed into terms of type `scala.meta.Term[T]`, 
and local types `U` are transformed into terms of type `scala.meta.Type`. In other words, 
definitions that are statically available outside macro bodies remain available in macro bodies,
whereas term and type parameters of enclosing inline methods become available as their representations.

## Macro expansion

During typechecking, the compiler treats macro expressions, i.e. invocations of `Predef.macro`,
as normal method calls, typechecking macro bodies and performing type inference if necessary, but nothing else.
In this proposal, macro expansion works differently from macro expansion in Scala 2.10+, 
where macro applications are expanded immediately after being processed by the typechecker.

An important consequence is that macro expressions cannot refine their types during expansion.
This makes it impossible to use the proposed macro system to implement [whitebox macros](http://docs.scala-lang.org/overviews/macros/blackbox-whitebox.html) from Scala 2.10+,
but we are planning to eventually provide alternative ways of enabling the most important whitebox functionality.

At some point in the compilation pipeline, after typechecking is complete for the entirety of the program, 
the compiler expands macro expressions outside the bodies of inline values and inline methods.
When macro expressions code appear inside the aforementioned constructs, 
they will be ignored by the macro expander and processed by the compiler as usual.

A macro expression is expanded by evaluating its body and replacing the original expression
with an expression that represents the result of the evaluation. 
The implementation is responsible for instantiating a `scala.meta.macros.Context` necessary for macro bodies
to evaluate and for converting between its internal representation for program elements and representations
defined in `scala.meta`, such as `scala.meta.Term` and `scala.meta.Type`.

After the expansion is complete, the result of the expansion is forcibly upcast
to the type of the original expression. This mechanism works exactly the same 
as the mechanism of blackbox macro expansion in Scala 2.10+.
