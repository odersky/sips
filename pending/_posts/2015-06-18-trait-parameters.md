---
layout: sip
title: Trait Parameters
---

__Martin Odersky__

__first submitted 18 June 2015__

## Motivation ##

We would like to allow parameters to traits. These replace early definitions, which are complicated and hard to get right.

## Syntax ##

The syntax already allows this. Excerpting from Dotty's SyntaxSummary.txt (the one for Scala 2 is analogous):

    TmplDef ::=  ([`case'] `class' | `trait') ClassDef
    ClassDef ::=  id [ClsTypeParamClause] [ConstrMods] ClsParamClauses TemplateOpt
    TemplateOpt ::=  [`extends' Template | [nl] TemplateBody]
    Template ::=  ConstrApps [TemplateBody] | TemplateBody
    ConstrApps ::=  ConstrApp {`with' ConstrApp}
    ConstrApp  ::=  AnnotType {ArgumentExprs}

## Initialization Order ##

Parent traits can now be introduced as a type or as a constructor which can take arguments. The order of initialization of traits is unaffected by parameter passing - as always, traits are initialized in linearization order.

## Restrictions ##

The following rules ensure that every parameterized trait is passed an argument list exactly when it is initialized:

1. Only classes can pass arguments to parent traits. Traits themselves can pass arguments no neither classes nor traits.

2. If a class `C` implements a parameterized trait `T`, and its superclass does not, then `T` must appear as a parent trait of `C` with arguments. By contrast, if the superclass of `C` also implements `T`, then `C` may not pass arguments to `T`.

For example, assume the declarations

    trait T(x: A)
    trait U extends T

`U` may not pass arguments to `T`. On the other hand, a class implementing `U` must ensure that `T` obtains arguments for its parameters. So the following would be illegal:

    class C extends U

We have to add the trait `T` as a direct parent of `C`. This can be done in one of two ways:

    class C extends T(e) with U
    class C extends U with T(e)

Both class definitions have the same linearization. `T` is in each case initialized before `U` since `T` is inherited by `U`.

The arguments to a trait are in each case evaluated immediately before the trait initializer is run (except for call-by-name arguments, which are always evaluated on demand).

This means that in the example above the expression `e` is evaluated before the initializer of either `T` or `U` is run. On the other hand, assuming the declarations

    trait V(x2: B)
    class D extends T(e1) with V(e2)

the evaluation order would be `e1`, initializer of `T`, `e2`, initializer of `V`.

## Interaction with Other Features ##

### Modifiers

Trait parameters accept the same modifiers as class parameters, with the same meaning. 

### Context bounds

Context bounds on type parameters of traits are allowed and map to implicit evidence arguments as usual.

### Implicit parameters

Traits allow implicit arguments either directly, or indirectly by expansion from a context bound. Implicit
arguments to such parameters are passed whenever the resulting program would be otherwise illegal. This means:

 - If a trait takes normal parameters before implicit parameters, and arguments for normal
   parameters are passed when inheriting the trait, but arguments for implicit parameters are missing,
   these will be provided through implicit search as usual.

 - If a trait `T` takes only implicit parameters and the trait is inherited from a class `C` but `C`'s superclass
   does not derive from `T`, then implicit arguments are provided.

## See Also ##


