---
title: "Determinism, Presence and Absence"
date: 2022-04-10T14:08:24-07:00
draft: false
tags: ['java', 'config']
---
{{< toc >}}

## Determinism, Presence and Absence

In this ongoing series related to Java configuration I've most
recently covered [what qualifiers are]({{< relref "qualifiers.md"
>}}).  They are strongly related to [what paths are]({{< relref
"paths.md" >}}).  A path picks out some possible objects in
objectspace.  Those objects may or may not exist.  Whether they are
known to exist, and for how long, is the subject of this post.

## Determinism

Determinism is, very loosely speaking: if you put the same stuff in
twice, you'll get the same result twice.

What does "same" mean?  For most purposes, "same" can simply mean
"indistinguishable".

If I have a sane implementation of `public final int add(final int a,
final int b)`, then if you call this method 47 times, supplying it
with `1` and `1` respectively each time, you'll get `2`, each time.

Why does this matter in configuration land?  It's kind of a
Schrödinger's cat situation.

Let's say you have a `Supplier<? extends String>` in your hand and you're about to call
its `get()` method.  Will it return a value?

Will it throw an exception?

You don't know until you try it.  So OK, let's say you call it.  Boom:
it throws an exception.

If you call it again, will it throw an exception again?  Will it be an
exception that is indistinguishable from the prior one?  Or will you
get a value this time?

You don't know; you have to try it.  So OK, let's say you call it
again.  This time it returns "Hello, world!".

This supplier is non-deterministic.  There's no way to know what it is
going to do.  To learn what it's going to do, you could read the
documentation, which might or might not be correct.  Perhaps the
documentation mentions that it could throw a `NoSuchElementException`
at any point for any reason.  Now you know it's non-deterministic.  It
would be kind of nice if this information were actually available
programmatically.

Suppose this time you have a different `Supplier<? extends String>` in
your hand.  Suppose you call it 347 times and it returns "Hello,
world!" each of those times.  Is it deterministic?  Maybe; you still
don't know.

Determinism is critical in configuration land because it helps you
understand when a piece of configuration might change (and I haven't
even _begun_ to dive into the fetid swamp that is mutable
configuration) and when it is guaranteed (to the extent possible) to
be constant.

## Presence and Absence

Some values in configuration land are _present_.  If I ask a loading
system for a type of [qualified]({{<relref "qualifiers.md" >}})
object, the loading system might return me a suitable value.  That
value is _present_: I asked for it, and I got it.  (If I ask a loading
system for it again, will I get the same object back?  That's
determinism, an orthoganal concept.)

Some values in configuration land are _absent_.  If I ask a loading
system for a type of [qualified]({{<relref "qualifiers.md" >}})
object, the loading system might _not_ return me a suitable value.
That value is _absent_: I asked for it, and I didn't get anything.
Again, determinism and absence are orthogonal.

Additionally: **presence and absence have nothing to do with `null`.**
Some present values can be `null` and it's entirely possible to
represent absence with an object.

Let's go back to the `Supplier` example.  If you were able to query
the `Supplier` to see what sorts of machinery it hides, you could see
whether you might have to call `get()` one time or many times, and you
could tell whether, if it returns `null`, that represents the presence
of a value explicitly set to `null` or the absence of a value
altogether.

If you could also query the supplier to find out whether the presence
or absence it reports is permanent or transient, then you would also
know whether it was deterministic.

In practice, these orthogonal concepts can be folded together into,
say, an `enum` whose values might be:

 * `ABSENT`: describes permanent (and hence deterministic) absence.
   In the `Supplier` example, there really isn't any point to calling
   the `get()` method.  If we say that absence is indicated by the
   throwing of an exception, for example, then any time you call
   `get()` you are guaranteed it will throw an exception.
 
 * `PRESENT`: describes permanent (and hence deterministic) presence.
   In the `Supplier` example, you can call the `get()` method once and
   be confident that you'll receive the one value it will forever
   return.  That value, of course, might be `null`.

 * `DETERMINISTIC`: describes permanent absence or presence, but you
   don't know which until you try.  In the `Supplier` example, maybe
   `get()` will return a value (which may be `null`) and maybe it will
   throw an exception.  Whatever it does, it will do for every
   subsequent invocation.  That's useful.
   
 * `NON_DETERMINISTIC`: who knows what will happen.  This is the
   default mode of `Supplier`s everywhere.
   
Some configuration systems clumsily stumble around these concepts,
some with better results than others:

 * MicroProfile Config and good old System properties are the worst
   here, since they conflate the concept of `null` and, in some cases,
   empty strings, with value absence.  Oops.  At least you can test if
   the System properties contain a key to discover absence
   vs. presence of a given value.
   
 * Lightbend's Typesafe Config tries very hard to do this but still
   falls down a little bit.  You can [dig into the
   `hasPathOrNull(String)`
   javadoc](https://lightbend.github.io/config/latest/api/com/typesafe/config/Config.html#hasPathOrNull-java.lang.String-)
   to see the gymnastics, and don't forget to consider that although
   it calls itself an immutable configuration system there's also
   [`invalidateCaches()`](https://lightbend.github.io/config/latest/api/com/typesafe/config/ConfigFactory.html#invalidateCaches--)
   to pay attention to.  So much for determinism!
   
 * JNDI at least knows that `null` is a valid value, so defines the
   [`NameNotFoundException`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/NameNotFoundException.html)
   type, so you can distinguish between absence and presence.  In
   theory, a `Context`'s `environment` could contain this information,
   but it is not standardized.
   
 * Jakarta RESTful Web Services is obviously an HTTP-centric
   specification and so can take advantage of the various HTTP caching
   directives.
   
## Introducing `OptionalSupplier`

To solve this problem, let's introduce `OptionalSupplier`.

Wait, you say, how come we can't just use `java.util.Optional`?
Because `Optional` is designed for `Stream` operations that cannot or
should not return `null`, and so deliberately equates `null` with
"emptiness", a concept similar to, but not equal to, absence.

Nevertheless, some of the fluent methods on `Optional` are quite
useful.  So what if we took the spirit of `java.util.Optional` and
applied it to a `Supplier`, with explicit rules on how `get()`
behaves, allowing for the possibility of `null` as a valid value?

Then you could do things like this:

```java {linenos=false}
switch (optionalSupplier.determinism()) {
case ABSENT:
  // You're never going to get a value; use a default
  // value instead maybe. Or you could throw an exception.
  // Or you could go ahead and call get() and let *it*
  // throw an exception to indicate absence. Here we
  // use a default value.
  greeting = "Hello, world!";
  break;
case PRESENT:
  // You only have to call get() once, and it won't throw.
  // It will always return the same value.  You could use
  // this information later to decide what to do about
  // caching and such.  Note the lack of the try/catch block.
  greeting = optionalSupplier.get();
  break;
case DETERMINISTIC:
  // Whatever get() does, it will always do, but we don't
  // know what it will do.  Guard appropriately.  With this
  // information you might be able to decide caching semantics
  // or abort early.
  try {
    greeting = optionalSupplier.get();
  } catch (final NoSuchElementException permanentlyAbsent) {
    // forever absent; use a default value, maybe?
    greeting = "Hello, world!";
  }
  break;
case NON_DETERMINISTIC:
  // We don't know what get() is going to do when it is called.
  // Guard appropriately, and now you know that greeting shouldn't
  // be cached, or, if you want to cache it, that a new value for
  // it might be returned by a get() invocation at any time.
  try {
    greeting = optionalSupplier.get();
  } catch (final NoSuchElementException absentForNow) {
    // Absent, but optionalSupplier.get() might return a
    // present value later.  For this example we just use a
    // default value.
    greeting = "Hello, world!";
  }
  break;
default:
  throw new AssertionError("Impossible enum constant: " + optionalSupplier.determinism());
}
```

Or you could do some fluentish stuff like this:

```java {linenos=false}
// This looks just like Optional.orElse(), but of course at the
// end of all this greeting may very well be null.  Absence is
// not java.util.Optional emptiness.
String greeting = optionalSupplier.orElse("Hello, World!");
// This assertion, in other words, might fail:
// assert greeting != null
```

So now we have notions of what a path is, what qualifiers are, and
whether any given value in configuration land is permanently or
transiently present or absent.  There's just one last foundational
piece to this whole thing, and that's subtyping and Java type
assignability semantics&mdash;coming up next.

