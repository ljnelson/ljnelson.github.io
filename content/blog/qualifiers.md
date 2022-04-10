---
title: "Qualifiers"
date: 2022-04-10T12:39:47-07:00
draft: false
tags: ['java', 'config']
---
{{< toc >}}

## Qualifiers

In this ongoing series related to Java configuration, I've most
recently covered [what a path is]({{< relref "paths.md" >}}).  I
described what a path consists of, and introduced the notion of
_qualifiers_.

In this post I'll talk a bit more about qualifiers.

## JSR-330 and Jakarta Dependency Injection

I think that good old JSR-330 (now [Jakarta Dependency
Injection](https://jakarta.ee/specifications/dependency-injection/2.0))
was the first to explicitly define the notion of a
[qualifier](https://jakarta.ee/specifications/dependency-injection/2.0/apidocs/jakarta/inject/qualifier).
In Jakarta Dependency Injection's case, a qualifier is necessarily an
annotation, but disregard that for the time being, because it's not at
all important.

The documentation is terse to the point of being almost useless, but
if you look at it long enough you start to see that a
qualifier&mdash;the abstract notion, mind you&mdash;is an immutable
value for an immutable, usually unnamed, implied attribute.  (I've
been [somewhat fascinated by all
this](https://lairdnelson.wordpress.com/2017/01/31/cdi-qualifiers-are-values/)
for quite a while now.)

So in the Jakarta Dependency Injection documentation, the
[`@Leather`](https://jakarta.ee/specifications/dependency-injection/2.0/apidocs/jakarta/inject/qualifier)
annotation represents a value for some sort of unnamed key, say,
`material`.

What's so special about a qualifier being an annotation?  Nothing.
Making a qualifier an annotation is an easy way to create an immutable
key value pair, that's all.

In Jakarta Dependency Injection, a class can be annotated with many
qualifier annotations.  That just means it can have many qualifiers,
i.e. a set of immutable key value pairs.

Together, a type and qualifiers constitute a path [as we've defined
it]({{< relref "paths.md" >}}).  That's interesting.

That is: in Jakarta Dependency Injection, when you ask for a
conformant system to inject an object, you are asking it to supply you
with an object of a particular type and with particular qualifiers.
The actual machinery that does this isn't concerned with injection at
all.  It's just logically making a [path]({{< relref "paths.md" >}})
of sorts out of the information you've supplied (the type and
qualifiers) and satisfying the loading request.

Hmm, this all [sounds familiar]({{< relref "paths.md" >}}), doesn't
it?

## Qualifiers, Suitability and Ambiguity

In fact, the conformant system will frequently not have _exactly_ the
object you were looking for, but it may have one that is _suitable_.
The concepts of qualifiers and suitability are intertwined and we'll
come back to their relationship soon.

At this point we've come to realize a few important things:

 * A qualifier is just an additional facet of a load request.
 * A qualifier doesn't have to be an annotation.
 * Qualifiers have nothing intrinsically to do with dependency
   injection.  It's just that JSR-330 happens to have defined the
   term.
 * In systems that use qualifiers, a request might be fufilled with
   something that is exactly what was requested, or something that was
   suitable for the request.

In a dependency injection system, ambiguity is the enemy.  Sometimes a
load request will exactly identify one object.  In that case, the
dependency injection system satisfies the load request and everything
is great.  Sometimes a load request, however, will identify two
different objects that are both suitable.  The dependency injection in
this case normally fails the request and reports that there was
ambiguity.

In a configuration system, ambiguity is not necessarily the enemy.
Frequently there are suitable values that get ranked or prioritized or
stacked or otherwise arranged so that you always get _something_ if at
all possible.  This is largely because configuration, unlike business
objectspace, describes many different environments.

But apart from the ambiguity angle, **a configuration system is just a
dependency injection system without the injection part**.

Now let's come back to ambiguity and qualifiers and suitability.

Let's say I ask for a `String` under the name `hostname`.  As we've
[seen]({{< relref "paths.md" >}}), a path can be reduced to a set of
qualifiers, so this could be represented in notation I just made up
like this:

```
{ name = hostname, type = java.lang.String }
```

Let's actually refine this.  Let's say that I still ask for a `String`
under the name `hostname`, but somehow the fact that my class that is
doing this is in an application that is running in the test
environment gets captured.  That might look like this:

```
{ name = hostname, type = java.lang.String, env = test }
```

Let's refine this again.  Let's say that I still ask for a `String`
under the name `hostname`, but somehow the fact that my class that is
doing this is in an application that is not only running in the test
environment but also in the west region:

```
{ name = hostname, type = java.lang.String, env = test, region = west }
```

Let's further suppose that the loading system doesn't actually have a
value that exactly matches all this.  Let's say that it can furnish a
value whose `type` is `java.lang.String`, whose `name` is `hostname`
and whose `region` is `west` (note: no `env` setting).  Should this
value "match" or not?  Is this value suitable, in other words?

Or: given a request path of `{ name = hostname, type =
java.lang.String, env = test, region = west }`, does a value path of
`{ name = hostname, type = java.lang.String, region = west }` match?
Is it suitable?

In a dependency injection system like CDI, the answer is no: the
injection point&mdash;the request path&mdash;does not have a
subset of the qualifiers of the proposed value.  Game over.

But there is a sense in configuration land that we _do_ want this
value to be suitable.  It's not _maximally_ suitable, since it doesn't
exactly match all the qualifiers, but it doesn't redefine any of them
(i.e. the `name`, `type` and `region` qualifiers all have the same
values as those of the request).

## Qualifiers and Just Enough Set Theory

If we try to translate this intuitive sense into something a little
more concrete, it becomes clear that, for a given key, a set of
qualifiers that omits that key is _suitable_ for a set of qualifiers
that specifies that key.  But a set of qualifiers that has a different
value for that key is not suitable.

It also turns out that if a set of qualifiers has keys in it that are
not present in a requested set of qualifiers, that's suitable too,
though maybe not as suitable as we'd like.

From this we know that a model of qualifiers will need to be able to
report the _intersection_ between two sets of qualifiers: the key
value pairs they have in common.  And we know that this model will
need to be able to report the _symmetric difference_ between two sets
of qualifiers: the key value pairs that are contained in one but not
the other.

Finally, given that qualifiers are likely to be represented
persistently, a set of qualifiers must have a defined iteration order
so that arbitrary changes in set ordering do not impact suitability
calculations.

## The Model

Putting it together, we can therefore say something like this:

 * A `Qualifiers` is an immutable set of entries.
 * An entry is an immutable pairing of a `Comparable` and an immutable
   object value.
 * A `Qualifiers` can report its size.
 * A `Qualifiers` may be empty.
 * A `Qualifiers` can report its intersection size with respect to
   another `Qualifiers`.
 * A `Qualifiers` can report its symmetric difference size with
   respect to another `Qualifiers`.
 * Because of all this, `Qualifiers` is a [value-based
   class](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/doc-files/ValueBased.html).

We've now sketched what qualifiers look like, and what a path made up
of them looks like.  We've also hinted at ambiguity and suitability.
In the next post, we'll look at the notions of determinism, presence
and absence, which are the last foundational pieces to any
configuration system.
