---
layout: sip
title: Inline Definitions and Macros
---

__Martin Odersky__

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
write such `val`s now as `inline`.  Inline values can be defined
anywhere, but inline methods must be members of a class, trait or
object.

Value and method definitions labeled `inline` are implicitly final;
they cannot be overridden.  Inline members also never override other
members. Instead, every inline member becomes an overloaded alternative
of all other members with the same name. Normal overloading resolution
is applied to pick an inline member over some other member.

Inline definitions only exist at compile-time; no storage is allocated for them and
no code is generated for them. That means that it is OK to have an inline member that
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

The compiler will do the following rewritings.

1. If `prefix.f[Ts](args1)...(argsN)` refers to a full applied inline
method, replace the expression with the method's right hand side,
where parameter references are replaced by corresponding arguments and
references to the enclosing `this` are replaced by `prefix`. Note that
the prefix and argument expressions are substituted as they are, so
any side effects might be duplicated, reordered, or omitted, depending on
where expressions are used in the method body.

   The rewriting is done on typed trees. Any references from the function
body to its environment will be kept in the rewritten code. Accessors
will be added as needed to ensure that private definitions can be
accessed from the callsite.

2. A conditional expression `if (c) t else e` where `c` has type `inline Boolean`
is replaced by `t` if `c == true` and by `e` otherwise.

   Note: We might also decide reduce `match` expressions with inline
selectors and inline patterns. While useful, this would be much harder
to spec and implement than conditionals.

Inline expressions are rewritten "outside in", that is, in call by name mode.

3. A field selection of an inline class with an inline argument selects the argument: E.g.,
`new C(42).x` rewrites to `42`.

## Inline Default Arguments

A default argument for any method may be marked `inline`. In that case,
the argument expression is supplied directly at the call site instead of
being packed in a default method.

Example: Given

     def foo(x: Int)(y: Int = inline x + 1) = ...

Then `foo(0)()` expands to `foo(0)(0 + 1)`

## Inline Types

We introduce `inline T` as a new form of type. Values of type `inline T` are values
of type `T` that are statically known to the compiler _at the point where the containing code is expanded_. Constant types are subtypes
of inline types, and inline types are subtypes of their underlying types. E.g.

    42  <:  inline Int  <:  Int

    (Float => inline Boolean)  <:  (inline Float) => (inline Boolean)  <:  (inline Float) => Boolean

The usage of inline types is restricted. Inline types can only appear in the following
positions:

 - As parameter or result type of an inline method:

        inline def square(x: inline: Int): inline Int = ...

        inline def power(x: Int, n: inline Int): Int = ...

 - As parameter of an inline class:

        inline class C(inline val x: Int) { ... }

 - As type of an inline val -- in fact it is automatically inferred there, i.e.

        inline val x: T = E

   is regarded as equivalent to

        inline val x: inline T = E

 - As the right hand side of an (unparameterized) type alias:

        type IS = inline String

 - As a type parameter on another inline type:

        inline List[inline Int]

        (inline Int, inline Int) => inline Boolean

 - As a refinement type of another inline type:

        inline Cell { def elem: inline Int }

 - As the type of a type ascription:

        (42: inline Int)

Inline types can specifically not be used as

 - types of abstract methods or fields
 - types of variables
 - types of class parameters
 - type parameters to methods
 - type parameters in `new` expressions or class parents.

## Inline Overloading Resolution

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

A macro is an expression of the form

    macro { ... }

where `{ ... }` is some block of Scala code. (In fact, `macro` may prefix arbitrary expressions,
but blocks are used most commonly). Macros may only appear in the bodies of inline methods.

Macros can only reference the following things in their environment:

 1. parameters of enclosing inline methods (where the parameter may, or may not be, marked inline),
 2. inline values,
 3. `this` references to classes containing an enclosing inline method as a member
 4. anything that's global,

A macro is said to _close over_ items in (1)-(3).

Inside a macro, the type of closed-over definitions changes. If the definition is of type
`inline T` outside the macro, it is of type `T` inside. If it is a non-inline type `U`, the type in
the macro is `Expr[U]`, the type of expression trees of type `U`. Types with several levels of
`inline` are handled analogously. For instance, a type like

    inline (inline Int, Boolean) => String

would be seen inside a macro as

    (Int, Expr[Boolean]) => Expr[String]

A dual transformation is applied to a macro's result. Normal types are prefixed with `inline` until
a type is of the form `Expr[T]` in which case it is replaced by `T`. So if a macro body returned
a list of type

    List[(String, Expr[T])]

the type of the macro itself would be

    inline List[inline (inline String, T)]

## Whitebox vs Black Box

Normaly, we would require that an inline method has an explicit result type.
This restriction is in place so that we can expand inline functions and macros
in a separate phase after type checking.

We could allow inline functions with missing result type (so called "whitebox macros"),
maybe controlled by a language import such as

    import language.whiteboxMacros

