---
layout: sip
title: Inline and Macros
---

__Martin Odersky__

__April 2015__


## Motivation ##

## Part 1: inine ##

## Inline Modifiers

We introduce a new reserved word: `inline`. `inline` can be used as a modifier for:

 - concrete value definitions, e.g.

        inline val x = 4

 - concrete methods, e.g.

        inline def square(x: Double) = x * x

value and method definitions labeled `inline` are implicitly final; they cannot be overridden.
The previous rule of regarding a `final val` with no explicit type as a compile-time constant
(inherited from Java) is dropped. Instead we write such `val`s now as `inline`.


## Inline Reductions

The compiler will do the following rewritings.

1. If `prefix.f[Ts](args1)...(argsN)` refers to a full applied inline
method, replace the expression with the method's right hand side,
where parameter references are replaced by corresponding arguments and
references to the enclosing `this` are replaced by `prefix`. Note that
the prefix and argument expressions are substituted as they are, so
any side effects might be duplicated, reordered, or omitted, depending on
where expressions are used in the method body.

The rewriting is done on typed trees. Any references from the function body to its
environment will be kept in the rewritten code. It is an error if a reference is
no longer valid at the call site (for instance because it refers to a private member
of the inline method's class).

2. A conditional expression `if (c) t else e` where `c` has type `inline Boolean`
is replaced by `t` if `c == true` and by `e` otherwise.

Note: We might also decide reduce `match` expressions with inline
selectors and inline patterns. While useful, this would be much harder
to spec and implement than conditionals.

## Inline Default Arguments

A default argument for any method may be marked `inline`. In that case,
the argument expression is supplied directly at the call site instead of
being packed in a default method.

Example: Given

     inline def foo(x: Int)(y: Int = x + 1) = ...

Then `foo(0)()` expands to `foo(0)(0 + 1)`

## Inline Types

We introduce `inline T` as a new form of type. Values of type `inline T` are values
of type `T` that are statically known to the compiler. Constant types are subtypes
of inline types, and inline types are subtypes of their underlying types. E.g.

    42  <:  inline Int  <:  Int

    (Float => inline Boolean)  <:  (inline Float) => (inline Boolean)  <:  (inline Float) => Boolean

The usage of inline types is restricted. Inline types can only appear in the following
positions:

 - As parameter or result type of an inline method

        inline def square(x: inline: Int): inline Int = ...

        inline def power(x: Int, n: inline Int): Int = ...

 - As type of an inline val -- in fact it is automatically inferred there, i.e.

        inline val x: T = E

   is regarded as equivalent to

        inline val x: inline T = E

 - As a parameter type of an inline class
 - As the right hand side of an (unparameterized) type alias:

        type IS = inline String

 - As a type parameter on another inline type:

        inline List[inline Int]

        (inline Int, inline Int) => inline Boolean

 - As the type of a type ascription:

       (42: inline Int)

Inline types can specifically not be used as

 - types of abstract methods or fields
 - types of variables
 - types of class parameters
 - type parameters to methods
 - type parameters in `new` expressions or class parents.


class C inline (x: Int, y: Int)


Question: How to achieve the following in a hygienic way:


    class PlusWrapper(x: Int) {
      @inline def ++ (y: Int): Int = plus(x, y)
    }
    @inline def plus(x: inline Int, y: inine Int): inline Int = x + y
    def plus(x: Int, y: Int): Int = x + y

    inline implicit def plusWrapper(x: inline Int) = new PlusWrapper(x)


Without hygiene, we'd have

    x, y: Int

    x ++ y
    -->
    plusWrapper(x).++(y)
    -->
    new PlusWrapper(x).++(y)
    -->
    plus(x, y)
    -->
    x + y


