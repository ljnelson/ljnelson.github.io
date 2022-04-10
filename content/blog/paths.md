---
title: "Paths"
date: 2022-04-10T10:52:18-07:00
draft: false
tags: ['java', 'config']
---
{{< toc >}}

# Paths

In this ongoing series related to Java configuration, I've most
recently covered [what it might look like if you had a `ServiceLoader`
that could take requests]({{< relref "service-loader-requests.md"
>}}).  I introduced the notion of a _path_, a sparsely populated
pointer of sorts to objects in objectspace.  Paths are not new, of
course, and are present in everything from filesystems to JNDI to
Jakarta RESTful Web Services.

In this post I'd like to dive a little deeper into how a path might be
put together.

A path is used for addressing, so we want it to be as specific as we
can possibly get it.  What does that mean?

Many times when we think about paths we think of them as if they were
simply names.  Their elements, in this model, are simple strings, and
they are separated by some kind of a separator, and that's it.

This isn't nearly enough for a configuration system addressing
scheme.  Weirdly, several different configuration systems take this
approach, and it is exactly _because_ they take this approach that
they are deficient.

Let's look back at JNDI again.  While JNDI paths consist of name
elements in a sequence, each element also has an
[environment](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/Context.html#getEnvironment()).
An environment in this system is just a bunch of key value pairs that
"ride with" the path element (named `Context` in JNDI).

Or let's look at Jakarta RESTful Web Services again.  While paths
there also consist of elements in a sequence, there are also headers
and matrix parameters that might logically belong to each element in
the sequence.  More key value pairs, if you squint in the right way.

Finally, if you look at all paths in a certain way, you can see that
at their absolute heart they are _nothing more than key value pairs
all the way down_.  They aren't _represented_ this way, of course, but
fundamentally a JNDI `Context` with an environment that contains `a =
b` and that is identified by a name of `a/b/c` is really just a
collection of key value pairs that consists of `{a = b, name =
a/b/c}`.

This insight is kind of important.  It means that when a loader
receives a request to load an object, it is, at some fundamental
level, receiving an immutable set of key value pairs describing the
request.  Once you see this you can't unsee it, and
names-with-separators addressing schemes just don't measure up.

It is nevertheless true that some of these key value pairs are often
more important to the human reader than others.  In the case of a JNDI
`Context`, the name itself is quite important since you always have to
have one and the way it is represented indicates the depth of a
hierarchy that might or might not be present.  Similarly, in Jakarta
RESTful Web Services, the length of a path often tells you something,
just by looking at it, about the specificity of the request.  So
representing the value of a hypothetical `name` key with a value that
is a sequence of names makes some sense.

(Names, as frequently mentioned on this blog, always have implicit
namespaces, so it's important that paths' names be able to be
_transliterated_: if you, a class developer, decide that your path is
going to have a name of `a/b`, and I, an application assembler, know
that `a/b` is already spoken for by another class developer, then I
have to have a way to reconcile this name clash.)

Similarly, there are plenty of implicit key value pairs that are part
of a path even when the path builder doesn't supply them explicitly.
For example, the current `Locale` or operating system might very well
be a key whose value might help a loader find an appropriate object.
Frequently these sorts of situational parameter values are not
explicitly supplied by a user, but are understood nevertheless to be
part of the loading request.  Making these of secondary concern seems
like a good thing to do.

Finally, paths in all systems that use them have an implicit or
explicit _type_.  A filesystem path ends in a type
(e.g. `java.io.File` or `java.nio.file.Path`).  A JNDI `Name` ends in
a type, but Java lacked the syntax to fully describe it; you can see
the footprints in the
[`NameClassPair`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/NameClassPair.html)
class.  A Jakarta RESTful Web Services path (of type `GET` (oh look,
another key value pair that rides with the request!)) terminates in an
entity of a particular type.  The type is, of course, just another key
value pair, logically speaking (`type = java.nio.file.Path`), but is
also a key value pair that is borne by all paths.  So it might make
sense to represent it explicitly and elevate its primacy.

When we put this all together we get something like this:

 * A `Path` is a sequence of `Path.Element`s.
 * A `Path` has `Qualifiers` that are the key value pairs that qualify
   it.
 * A `Path`'s final `Path.Element` has a type, which is also the type
   of the `Path`.
   * Other `Path.Element`s need not have a type.
 * Each `Path.Element` has a name.
   * A `Path.Element` must be able to be transliterated without
     changing source code.
 * Each `Path.Element` has `Qualifiers` that qualify it and no other
   `Path.Element`.
 * Any given `Path.Element` may or may not have a type, but a
   `Path.Element` that ends a `Path` must have a type.
 * A `Path`'s `Qualifiers` includes qualifiers that qualify the path
   as a whole as well as the appropriately-scoped `Qualifiers` from
   each of its `Path.Element`s.
   
 Note that each `Path.Element` in the model described above does not
 have a type (paths can be sparsely populated).  So this does not
 imply any kind of hierarchy, nor should it.
 
 I've quietly introduced the notion of a `Qualifiers`, which is
 basically a read-only set of immutable key value pairs with a defined
 iteration order.  A `Qualifiers` may be empty.
 
 Again, just for the thought experiment, it's worth remembering that a
 `Path` could be fully represented as a `Map` with required `name` and
 `type` keys that have non-`null` values.  But for the reasons listed
 above it's better to represent it as its own thing.
 
 Since I'm proposing to build `Path`s around `Qualifiers`, the next
 post will dive into them a little more deeply.
