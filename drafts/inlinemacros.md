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
outside the bodies of inline methods.
When these patterns of code appear inside inline methods,
they will be ignored by the inliner and processed by the compiler as usual.

1. If `prefix.f[Ts](args1)...(argsN)` refers to a fully applied inline
method, hoist the prefix and the arguments into temporary variables
and then replace the expression the method's right-hand side,
where parameter references are replaced by references to temporary variables
created for the corresponding arguments and references to the enclosing `this`
are replaced by references to the temporary variable created for the prefix.

    val x$prefix = prefix
    val x$1 = arg1
    ...
    val x$M = argM
    <f's body with parameter and this references replaced with x$'s>

   This hoisting is intended to preserve the semantics
of function applications under inlining: A function call should have the same
semantics with respect to side effects independently on whether the function was made `inline` or not.

   The rewriting is done on typed trees. Any references from the function
body to its environment will be kept in the rewritten code.

  If the result of the rewriting references private/protected definitions
to the class that defines the inline method, these references will be changed
to use accessors generated automatically by the compiler. To ensure that the rewriting
works in the separate compilation setting, it is critical for the compiler to generate
the accessors in advance. We currently think that it is feasible, because it is always
possible to detect these references by analyzing the bodies of inline methods.

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

    inline ((Float => inline Boolean))  <:  inline ((inline Float) => (inline Boolean)) <:  inline ((inline Float) => Boolean)

The usage of inline types is restricted. Inline types can only appear in the following
positions:

 - As parameter or result type of an inline method:

        inline def square(x: inline Int): inline Int = ...

        inline def power(x: Int, n: inline Int): Int = ...

 - As parameter type of an inline class:

        inline class C(val x: inline Int) { ... }

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

## Inline Overloading Resolution (to be discussed)

We have stated that the rewriting is done on typed trees and that references of
the original tree are kept after the rewrite. There is one exception to this: References to
overloaded methods are re-computed after the rewrite. So a different alternative might
be chosen if the argument types after the rewrite are more specific than the method
parameter types. So, compile-time overloading resolution is essentially multi-method dispatch.
Here is a scenario where this can be used to support constant folding inline infix methods.

The task here is to define `++` as an infix operator for inline Ints, which should
give an inline result if and only if its two operands are inline values. Here's how it can be achieved:

    inline implicit class PlusWrapper(val x: Int) {
      inline def ++ (y: Int): Int = plus(this.x, y)
    }
    inline def plus(x: inline Int, y: inline Int): inline Int = x + y
    def plus(x: Int, y: Int): Int = x + y

Let's assume we have

    a, b: inline Int

Then we rewrite `a ++ b` as follows:

    a ++ b
    -->
    PlusWrapper(a).++(b)
    -->
    new PlusWrapper(a).++(b)
    -->
    plus(new PlusWrapper(a).x, b)
    -->
    plus(a, b)
    -->
    a + b: inline Int

Note that the call to `plus` in `PlusWrapper` resolves statically to the non-inline method `plus`.
Yet in the last-but-one expression of the above rewrite sequence it is rewritten to refer to the
inline method `plus` because at that point both arguments are known to be inline values.

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
the compiler expands macro expressions outside the bodies of inline methods.
When macro expressions appear inside inline methods,
they will be ignored by the macro expander and processed by the compiler as usual.

A macro expression is expanded by evaluating its body and replacing the original expression
with an expression that represents the result of the evaluation.
The implementation is responsible for instantiating a `scala.meta.macros.Context` necessary for macro bodies
to evaluate and for converting between its internal representation for program elements and representations
defined in `scala.meta`, such as `scala.meta.Term` and `scala.meta.Type`.
