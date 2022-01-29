---
title: "Service Loader Requests"
date: 2022-01-28T14:52:27-08:00
draft: false
tags: ['java', 'config']
---
{{< toc >}}

## What Would a `ServiceLoader` That Could Take Requests Look Like?

In this ongoing series related to Java configuration, I've covered
[JNDI]({{< relref "java-configuration-jndi-and-naming-operations" >}})
and [Jakarta RESTful Web Services]({{< relref
"java-configuration-jaxrs-as-a-configuration-system" >}}).  Both
involve loading Java `Object`s from potentially many providers with
disambiguation algorithms built in, and both permit the application
assembler to work around naming clashes.  Of course, neither is a Java
configuration system.

Let's look at some of the super-concepts these two technologies have
in common.

### Objectspace

The first is what I'll call, somewhat pretentiously, _objectspace_.
This is the notional space of all possible Java objects (and
primitive types) in all possible Java virtual machines in all possible
worlds.  Objectspace is big.

You can pick out an individual object from objectspace, let's say, by
supplying some _loading system_ with an _address_.  (This sounds like
pointers!  Indeed, that's part of the analogy I'm going for.)  Give
the system an address, it hands you back the corresponding object.  By
definition, if you provide a precise enough address there will be
exactly one object it picks out, never zero.  Simple.  What could
possibly go wrong?

To get an object from objectspace according to the rules as I've laid
them out so far, you have to be _remarkably_ precise.  You'd better be
able to distinguish objects by type, certainly, and whatever the
addressing scheme looks like, it better take type into account.  But
you can't stop there.

### Ambiguity

If we _do_ stop there, we have to address ambiguity.  If I ask for
what I hope is the only `String` in all of objectspace by asking a
hypothetical loading system for the one true object that has the type
`String.class`, the loading system will laugh in my face.  There are
many `String`s in objectspace.

OK, OK; I might instead want to ask this hypothetical loading system
for a very particular `String`, namely the one in objectspace under
the System property key `java.home`.  But even this isn't specific
enough, since, remember, objectspace encompasses all Java objects in
the universe.  There are many Java homes.  The one that identifies
_my_ Java installation is just one of them!  And yet I get a single
`String` back, so, Laird, your analogy sucks.

### Addressing

Not so fast!  When I ask for this hypothetical loading system to give
me the `java.home` `String`, I'm actually supplying plenty of other
implicit addressing information in my request, whether I know I'm
doing this or not.  Specifically, I'm _actually_ asking for the only
`String` in all of objectspace that has the type `String.class`, that
is indexed under the System property key `java.home`, and on this JVM,
running on this machine, on this architecture, in this universe, and
so on and so forth.  That _may_ be enough to pick out the `String` I
want (and normally is in non-hypothetical, non-abstract,
non-[architecture
astronaut](https://www.joelonsoftware.com/2001/04/21/dont-let-architecture-astronauts-scare-you/)
cases).

So types and JVM-wide names seem to be the bare minimum to pick out an
object from all of objectspace.  Sounds easy—but that's not quite
right either.

Let's say instead that I would like this loading system to fetch me
the one true `String` that can be found in some hazy way "under" the
name `preferredHost` (I made this up so that it's not some key defined
by Java itself).  Let's further wave our hands about possible name
collisions: let's pretend that no other developer in the entire
universe could possibly ever be writing a class like mine, and no
other developer in the entire universe could possibly ever mean
anything semantically different from what I mean when I say
`preferredHost`.  (In practice, of course, these assumptions are
completely ridiculous.  Bear with me; we'll get there.)

But if my application is running in `test`, in some sort of hazy way,
then there may very well be an ambiguity here.  `preferredHost` and
`String.class` no longer uniquely identify a `String` in objectspace.
Maybe there is another string value indexed under `preferredHost` in
objectspace (that identifies the `production` `preferredHost` for
example).  Oh, shoot, I guess really what I was asking for all along
was the one true `String` indexed under the explicit name
`preferredHost` and the implicit name `test`.  (Note that in no way
was I asking for the explicit name `test.preferredHost`, nor did I
have some kind of fallback in mind.)  As a component developer, I of
course didn't know what application my component was running in, so it
didn't occur to me to check for this case.  Oops.

Objectspace is _big_.  Like, [you just won't believe how vastly,
hugely, mind-bogglingly big it
is](https://www.goodreads.com/quotes/14434-space-is-big-you-just-won-t-believe-how-vastly-hugely).
Picking out a single `Object` in it is damn near impossible.

Instead, typically what we do, while blissfully not being aware of it,
is: we supply _some_ addressing information, and then rely on some
common case where we were expecting one thing, and there was only one
thing that happened to match our imprecise addressing information, and
we asked our loading system for it, and it responded, and we got only
one thing, not 27, and it never occurs to us that 27 of those things
is actually a very real possibility.  Then the bugs roll in.

**Good loading system APIs recognize that no addressing system you can
come up with will ever pick out an object from objectspace without
some kind of further disambiguation.**

#### `ServiceLoader` Addressing

Let's look at
[`java.util.ServiceLoader`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/ServiceLoader.html)
as an example.  This class deliberately embraces ambiguity.  The only
kind of addressing information you can give it is type.  Ask a
`ServiceLoader` for the thing corresponding to `SomeService.class`,
and it will hand you back an `Iterable<SomeService>`.  Anyone can put
a service provider file on the classpath (discoverable as a classpath
resource at `META-INF/services/com.foo.SomeService`, let's say), and
its entries will be picked up.  So great, how do you filter all the
various `SomeService` implementors?  That, dear reader, is up to you.

The upshot is: `ServiceLoader`, like many loading systems, accounts
for the almost completely necessary ambiguity of objectspace in its
API design.  Ask for a thing of a type, get back many things of that
type.  Your problem to solve.

#### Jakarta RESTful Web Services Addressing

Let's consider Jakarta RESTful Web Services.  Let's pretend that
Jakarta RESTful Web Services is, among other things, a way to (again,
partially) address into objectspace.  After all, with a resource
method you can say, effectively, "this method will return a
`SomeService` if the addressing information matches the path `a/b/c`
and the media type `application/json`.".  So if you ask _this_ loading
system for a `SomeService.class` object, providing no path information
and no media type information, the ambiguity is reduced somewhat: this
resource method I described will not "fire" and it, at least, will not
be responsible for returning a `SomeService`.  Some other resource
method might.  If more than one "fires", then there is an algorithm
that somewhat arbitrarily picks one.

#### JNDI Addressing

Let's consider JNDI.  Let's pretend that JNDI is, among other things,
a way to (again, partially) address into objectspace.  After all, with
a
[`javax.naming.spi.ObjectFactory`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/spi/ObjectFactory.html)
you can say, effectively, "this `ObjectFactory` will return a
`SomeService` if the addressing information matches the [compound
name](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/CompoundName.html)
`a/b/c` and an expected
[`environment`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/spi/ObjectFactory.html#getObjectInstance(java.lang.Object,javax.naming.Name,javax.naming.Context,java.util.Hashtable))."
So if you ask _this_ loading system for a `SomeService.class` object,
providing no compound name information and no environment information,
the ambiguity is reduced somewhat: this `ObjectFactory` I described
will not "fire" and it, at least, will not be responsible for
returning a `SomeService`.  Some other `ObjectFactory` might.  If more
than one "fires", then there is an algorithm that somewhat arbitrarily
picks one.

(You of course see the similarity.)

Each of these loading systems differs, of course, in what its
"loaders" are permitted to do (resource methods and `ObjectFactory`
instances have to play by the rules of their specifications).  And
each loading system permits more or less addressing information to
"ride along" with the general request for a typed Java `Object` from
objectspace, to help pare down the possible matching objects to a
manageable number.  But they're very similar when we look at them as
object loading systems that accept reasonably fine-grained addressing
information that identifies many things, but hopefully not lots of
things.

Finally, each of these systems is not a Java configuration system!
But you know what is a configuration system, of a sort?
`java.util.ServiceLoader`.  It is a configuration system that just so
happens to have punted the problem of resolving ambiguity to the end
user.  It is also designed to satisfy only one of several
configuration-related use cases, which it does very well.  What if we
augmented `ServiceLoader` with the ability to receive some kind of
addressing information that would allow it to avoid ambiguity in more
cases than it does right now?  What would we need to add?

### Loaders, Paths and Qualifiers

First, let's talk terminology.  A configuration system doesn't just
load services, so we'll drop that word.  So `ServiceLoader` will
become `Loader` for this discussion, and "service provider" will
become "provider", and so on.

Next, we need a good word for our addressing information.  We can look
to JNDI and Jakarta RESTful web services here.  In JNDI, the
addressing information is termed a _name_.  In Jakarta RESTful Web
Services, the addressing information is termed a _path_ (since of
course it is concerned with URIs).  I like _path_ better than _name_
for addressing information because it implies a type (a filesystem
path yields a file, for example; a Jakarta RESTful Web Services path
yields an object of a particular media type) and can either imply a
tree structure or not, depending on how it's used.  It's also about
finding your way to a destination.  So we'll go with path.

The kind of path we'll talk about is similar to Jakarta RESTful Web
Services' path: it is sparse.  That is, it's not the case that every
element in a path necessarily identifies an object at that location
(in direct contrast with JNDI, where every atomic name in a compound
name identifies a context).  In this regard, a path is like a pointer
or a probe: ultimately what matters is what it picks out at the end,
not the intermediate objects that may exist along the way.

Our path will also need to have the ability to pass additional
addressing information.  JNDI has [environment
properties](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/Context.html#:~:text=provider%20may%20not.-,Environment%20Properties,-JNDI%20applications%20need).
Jakarta RESTful Web Services provides resource methods with the
ability to inspect headers, query string parameters, path parameters
and matrix parameters.  Each of these facilities is a way to further
_qualify_ a request.  We'll consequently follow in the [terminological
footsteps of systems like
CDI](https://jakarta.ee/specifications/cdi/3.0/jakarta-cdi-spec-3.0.html#qualifiers)
and call this additional addressing information _qualifiers_.

**Putting it all together, a _path_ is a _typed_ and _qualified_
ordered sequence of _elements_ that identifies zero or more objects in
objectspace.**

**A _loader_ is a component that dereferences a path.**

If you look at things this way, then a `java.util.ServiceLoader` is a
loader that dereferences very particular kinds of paths, namely those
that have a type, zero elements and no qualifiers.

## What's Next?

We're on our way, but we're not there yet.  What lies at the end of
any path is not necessarily an object, but may be nothing or many
objects.  Also, since in the real world the objectspace we're dealing
with at any given time in any given system is actually fairly small
and sparsely populated, we might want a strategy for what to do when a
path doesn't actually identify anything but something else might be
_suitable_ for it.

Additionally, even if a path _does_ identify something, it may come
and go over time, being _temporarily_ or _permanently_ _absent_ or
_present_.  What our loader loads may therefore need to be not the
object itself, but some kind of dereferencer that is pinned to the
path that yielded it.

There is also the notion of an application's _environment_ to be
concerned with.  An application's environment is the portion of
objectspace it inhabits (the machine it lives on, the locale it's in,
various other automatic and implicit coordinates, and the
human-authored configuration that is suitable for it), which includes
parts that a _component_ of that application, asking for an object to
be loaded, is unaware of, but that are necessary to include with the
addressing information.  Other environments within objectspace may or
may not be suitable for a loader to consider when trying to
dereference a path.

The journey continues; stay tuned.

