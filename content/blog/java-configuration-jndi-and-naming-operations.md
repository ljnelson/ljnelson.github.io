---
title: "Java Configuration: JNDI and Naming Operations"
date: 2021-12-30T16:22:37-08:00
draft: false
tags: ['java', 'jndi', 'config']
---
{{< toc >}}

## Background and Rationale

I've written
[previously](https://lairdnelson.wordpress.com/2021/11/07/configuration-and-dependency-acquisition/)
about how Java configuration systems boiled down to their essence are
simply systems for loading Java objects that are described in a
particular way, usually by some kind of name (along with
[qualifiers](https://lairdnelson.wordpress.com/2021/12/07/qualifiers-and-configuration-coordinates-in-configuration/)),
and that such systems have absolutely [nothing to do with dependency
injection](https://lairdnelson.wordpress.com/2021/11/07/configuration-and-dependency-acquisition/).

These ideas are not new.  In the Java enterprise world, they first
showed up in a systematic way in the [Java Naming and Directory
Interface (JNDI)
API](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/module-summary.html).
This API, in turn, is a slight modernization, and idiomatic Java
translation, of the [Object Management Group's Naming Service
Specification](https://www.omg.org/spec/NAM/1.3/PDF), which quietly
stagnated sometime in 2004.

Somewhere along the line, JNDI got (wrongly) pigeonholed as just a
way to access LDAP servers.  I'm going to ignore that entire side of
things (the "directory" part), because that's not what JNDI primarily
is at all.

Then it got demonized because of the `java:` URL scheme and the
`comp/env` naming prefix mandated by the Java EE Platform
Specification to identify a component's environment, but, as we'll
see, the concept underlying this prefix is critical, regardless of how
clunky the actual implementation turned out to be.

Now it is routinely rejected more because of its age than because of
any technical limitations it might have.  Having said that, it most
certainly shows its age, and not in a good way.

So why _would_ we look at all this?  Because to this day it is the
only Java-related configuration-like system that has appropriately
considered most, if not all, the concerns that arise when you talk
about loading Java objects that are qualified in some way into a
class—the very heart of Java-centric configuration use cases.

It's somewhat bizarre to note that the Java-centric configuration
frameworks currently _en vogue_ are so comparatively deficient:

 * MicroProfile Config certainly has not considered these concerns.
   (As I've [written
   before](https://lairdnelson.wordpress.com/2021/10/14/some-of-the-things-i-dont-like-about-microprofile-config/),
   it is non-deterministic and subject to namespace clashes.)

 * Lightbend's TypeSafe Config has not considered these concerns.
   (Names are considered to be absolute.)

 * Spring has not considered these concerns.  (Names are considered to
   be absolute and configured objects are presumed to "belong" to the
   Spring ecosystem.)
   
 * Jakarta Config is stumbling backwards into these concerns without
   realizing they are valid concerns, and is repeating the egregious
   mistakes of MicroProfile Config before it.  (It is
   non-deterministic and subject to namespace clashes and is focusing
   its standardization efforts in irrelevant places.)

I'm sure there are others.

**What if we could extract the age-old lookup-and-namespace-related
concerns that JNDI addresses and re-express them with a modern API?**
That would make for a pretty decent Java-centric configuration API.

To catch a glimpse of these concerns and how they are addressed within
JNDI requires a little bit of squinting.  The JNDI API itself is old
and outdated by today's standards.  It predates generics, for one
thing, and [attempts to modernize it, regardless of how trivial, have
been gently and officially
rebuffed](https://mail.openjdk.java.net/pipermail/core-libs-dev/2021-August/080696.html).
For another, it predates the [`java.util.ServiceLoader`]() convention
of discovering service providers at startup.  Getting past these
anachronisms and others like them can be a little tricky, but the
journey is worth it.

## What's In A (Naming Service) Name?

Let's talk about what a naming service does, and why the terms even
exist, and, as a result, why no one ever thinks of "naming service" in
the same breath as "configuration" (but maybe they should).

First, we'll talk about JNDI's precursor, the [Object Management
Group's Naming Service
Specification](https://www.omg.org/spec/NAM/1.3/PDF).  I know:
CORBA—but bear with me.

### The Object Management Group Naming Service Specification

Let's revisit the 1990s for a moment.  [How do you do, fellow
kids?](https://knowyourmeme.com/memes/how-do-you-do-fellow-kids)

Back in CORBA's day, you could park distributed objects on one
machine, and use another machine to look them up, and the calling
program might be written in an entirely different language from the
serving program.  The sub-programs doing the actual retrieval and
serving were known as ORBs: object request brokers.  As you might
imagine, an object request broker turns a simple local request for a
particular object into a distributed request for that object, and
answers such remote requests as well.

Because CORBA is cross-language, this meant that each ORB had to be
able to park an object under some kind of location that any requestor
could specify in some way, regardless of the languages involved.  All
languages have strings, and CORBA had ironed out the various language-
and machine-related discrepancies among strings, so the way you
identified the object you wanted to retrieve from some other ORB was
to effectively just name it.  So a _naming service_, then, is
something that, when given a string-typed, address-like name of some
kind, gives you back an object bound under that name, suitable for
immediate use in the calling language.  (Usually there was a hazy
presumption that something remote-ish had occurred, e.g. you had
retrieved an object over the network from some other machine
somewhere.  This was because so-called distributed objects were cool.
Now we know better, but the (mostly valid) concepts involved apply
even locally, as we'll see.)

(More importantly for configuration-related purposes, a name's sole
function, really, is to help distinguish one typed object from another
object bearing the same type.)

Getting ORBs from different vendors to cooperate was tough, so there
were a lot of specifications that ended up being written by the
[Object Management Group](https://www.omg.org/).  One of them was a
specification governing hierarchical naming semantics, named,
appropriately enough, the [Naming Service
Specification](https://www.omg.org/spec/NAM/1.3/PDF).

There's nothing that says, inherently, that a naming service has to be
hierarchical.  But hierarchies can be convenient, and the Naming
Service Specification decided to formally specify tree-based naming
semantics instead of flat ones.  The root of a naming service tree is
a _context_; its nodes are _objects_; its branches are _names_.

(A context, of course, is also a kind of object, and again, while
specifying the notion of a child context is not required, that's how
the Naming Service Specification decided to do it.)

(Some of this hierarchical stuff was due to the inherent chattiness of
CORBA itself: you didn't want to traverse deep graphs with remote
calls for each traversal operation!  Better to grab a context and use
it directly to get its stuff.)

Among other things, all of this means that, in the Naming Service
Specification, any given name is always relative to a parent context.
There are no absolute names.  If I give you a name, you don't know
what it means or designates (in the absence of other information).
But if I tell you that it's relative to a particular context, then you
have the tools to be able to _resolve_ the name against that context
and find the object it points to.  A context turns out to be a
namespace (among other things).

(In the world of name-based configuration this concept is absolutely
critical.  It is ignored by all major Java-centric configuration
frameworks for no good reason that I can see.)

In the Naming Service Specification, names have structure.  A sequence
of _components_ within a name forms a _compound name_.  Its individual
components are also known as _simple names_.  Finally, if you have a
compound name with two components (simple names), there is also one
context involved: the one that the first component desginates, and to
which the second component is relative. (To belabor an earlier point,
note that the first component (simple name) is relative to some
context but in this example I haven't told you what it is, so,
standing on its own, this hypothetical compound name is rather
useless.)

The Naming Service Specification also gestures feebly at another type
of identifier that is sort of part of names: _kind_.  It reads, in
part:

> The **kind** attribute adds descriptive power to names in a
syntax-independent way.  Examples of the value of the **kind**
attribute include _c_source_, _object_code_, _executable_,
_postscript_, or “ ”. The naming system does not interpret, assign, or
manage these values in any way. Higher levels of software may make
policies about the use and management of these values. This feature
addresses the needs of applications that use syntactic naming
conventions to distinguish related objects. For example Unix uses
suffixes such as **.c** and **.o**. Applications (such as the C
compiler) depend on these syntactic convention to make name
transformations (for example, to transform **foo.c** to **foo.o**).

(This reads as a kind of primitive precursor to MIME types.  It also
demonstrates, inadvertently, that any system that uses a "name" to
look something up eventually realizes that it _must_ use
[_qualifiers_](https://lairdnelson.wordpress.com/2021/12/07/qualifiers-and-configuration-coordinates-in-configuration/)
to look something up instead.  JAX-RS matrix parameters are another
example of this sort of thing.  There's plenty more to say about this,
but not here.)

### JNDI

JNDI's naming services are a very straightforward translation of the
language-independent terminology of the Naming Service Specification
into relatively idiomatic Java terms.

JNDI, like the Naming Service Specification, has a `Context`, which,
like its namesake, is a locus of
[`Name`s](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/Name.html)
that designate `Object`s (including other `Context`s).  (A `Context`
belongs to a _naming system_, which is a conceptual entity seemingly
introduced by the JNDI specification and not really present in the
Naming Service Specification.)

JNDI's
[`Name`s](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/Name.html)
are like those of the Naming Service Specification, but do not
explicitly include the concept of _simple_ names.  Instead, a JNDI
name is always either a [compound
name](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/CompoundName.html),
even if it has only one component, or a [_composite
name_](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/CompositeName.html),
a term not mentioned in the Naming Service Specification.  (Composite
names were added in a JNDI specification revision, and allow a single
name to automatically span actual naming systems, with sophisticated
parsing semantics.)  Within any given naming system, hierarchical or
flat, you're always working with compound names.

Like the Naming Service Specification, any given JNDI
[`Name`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/Name.html)
is always relative to a parent `Context`.

#### Bootstrapping

JNDI allows an implementor to bootstrap itself.  This is unique, even
today.

Specifically, the
[`InitialContext`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/InitialContext.html)
class is just a `Context` implementation that finds a "real" `Context`
implementation to use, and then delegates all operations to it.  The
JNDI bootstrap mechanism, which predates the `java.util.ServiceLoader`
service provider approach but which seems to have served as its
inspiration, allows custom `InitialContext` implementations to be
supplied from outside of the framework.

This means, among other things, the `Context` to which primordial
`Name`s are relative is highly customizable, even by end users.

In today's Java world, someone would probably have introduced
(needlessly) a builder-style object, or a whole scaffolding framework
for finding the root `Context`.  In JNDI, you just call `new
InitialContext()` and you're done, as it should be.

#### Retrieval Operations

From the looking-things-up perspective, that's about it.  Get your
hands on a
[`Context`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/Context.html),
[ask it for an `Object` under a particular
`Name`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/Context.html#lookup(javax.naming.Name)),
and make use of the resulting `Object`.  Where the `Object` came from,
how it was bound, whether it was synthesized out of other `Object`s,
whether various kinds of links and redirects were followed, whether
the `Object` fronts some other kind of system—none of these things are
your concern.  You named an `Object`, you resolved that name against a
`Context`, and you took delivery of that `Object`.

Of particular note, you probably also cast that `Object` to a type you
were expecting.  More on that later.  Peeking ahead for a moment, a
name always has a conceptual type that the caller, for the most part,
is expecting.

#### Binding Operations

Since JNDI emerged from the CORBA world and was a faithful carryover
of the Naming Service Specification, it also handled the "writing
side" of naming services.  Placing an `Object` under a `Name` in JNDI
is known as _binding_.  For my purposes, I'll be ignoring all binding
operations.  Java EE, in fact, one of the earliest consumers of JNDI,
required that `Context`s exposed to the end user be read-only, and
it's a regrettable fact (in my opinion) that the mutating operations
of JNDI were not split into their own sub-specification.

#### Name-to-`Object` Bindings

For ordinary use cases, a `Name` in JNDI is always conceptually bound
to some kind of `Object`.  This means that a `Name`, regardless of how
many components it might have, also has an effective (Java) type,
namely, that of the `Object` that it references.

JNDI has a few clunky introspective APIs for listing bindings, but for
my purposes I'm going to ignore them, since in configuration use cases
you already know the name of the thing you're looking up.  What's
mainly important here is that a `Name`, when resolved against a
`Context`, is effectively typed by a Java class.

#### Synthesizing Operations

JNDI has a facility, not found in the Naming Service Specification,
that permits some other Java object to synthesize a Java object out of
"raw materials", any of which may be bound in a tree rooted at any
`Context`.  This object doing the synthesizing is called an
[`ObjectFactory`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/spi/ObjectFactory.html).
A JNDI implementation itself can supply `ObjectFactory` instances,
but, more importantly for this article's purposes, so can users and
component providers.

This sounds complicated but is actually fairly simple.

Recall that when you look something up in a JNDI tree rooted under a
`Context`, you are blissfully unaware of the name resolution process
(how the `Object` that is eventually delivered to you came to exist).
Was it put together out of other things?  Was it created out of
nothing?  Was it loaded from disk?  Served over the network?  You
don't know and you don't care.

For certain cases, name resolution is absurdly simple: someone
explicitly bound a textual string into a `Context` under, say, the
name "`hoopy`"; you asked for the `Object` bound under "`hoopy`"; and
lo and behold you got back that very string—perhaps "`frood`"—as a
Java `String` object.  Nothing fancy here.

##### `ObjectFactory`

In other cases, an
[`ObjectFactory`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/spi/ObjectFactory.html)
gets involved.  This happens because the [`Context::lookup`
operation](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/Context.html#lookup(javax.naming.Name))
delegates, in all implementations that I'm aware of, and perhaps
explicitly stated somewhere, to a call to the
[`NamingManager::getObjectInstance`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/spi/NamingManager.html#getObjectInstance(java.lang.Object,javax.naming.Name,javax.naming.Context,java.util.Hashtable))
method, which pulls in `ObjectFactory` instances to actually perform
the creation (or retrieval, or synthesis, or whatever) of a Java
`Object` for a given `Name`.

So backing up from the minutiae for a moment, depending on what
`ObjectFactory` instances are on the classpath and properly
designated, a JNDI implementation may synthesize `Object`s out of thin
air as if they were bound explicitly to various `Name`s.

`ObjectFactory` instances and explicit binding can work together,
too.  An administrator can bind a [`Reference`]() into a JNDI
implementation that explicitly names the `ObjectFactory` to use to
perform the actual name resolution.

###### URL Context Factory

One kind of `ObjectFactory` is one that plays the role of a _URL
context factory_.  These kinds of `ObjectFactory` implementations can
create `Context`s to represent particular URL schemes, such as, most
infamously, `java:`, which is at the heart of the Java EE (and now
Jakarta EE) specifications.

The notorious `java:comp/env` "prefix" is, more specifically, a URL
whose semantics are strictly defined by the Java EE (now Jakarta EE)
Platform Specification.  `java:` causes a URL context factory
specified by the Platform Specification to come into the picture, and
`comp/env` is a compound name, relative to the root `Context` created
by the URL context factory, denoting, by definition, a `Context` with
very specific semantics.  One of its specific semantics is very
interesting to configuration systems, because it disambiguates
component namespaces.  That is, if an EJB refers to a `Frood` stored
under the name `hoopy`, and another EJB refers to a `Towel` stored
under the name `hoopy`, no conflict can occur, so long as `hoopy` is
relative to `java:comp/env` in both cases.

No Java-centric configuration framework that I'm aware of other than
JNDI addresses this extremely important concern.

## Putting It All Together

So backing up to the conceptual level, JNDI provides the ability for
two names to be used by two different components developed in
isolation to refer to two different things, with at least the
_possibility_ that a name clash will not occur (one component's
`java:comp/env` `Context` is not the same `Context` as another
component's `java:comp/env` `Context`).

In addition, it allows for user-supplied factories to handle
individual name resolution operations.  There is a defined search
order for discovering such factories.  You supply a name, the system
searches until it finds something that can make `Object`s for that
name.

JNDI properly separates the lookup use cases (ask for an `Object` of a
particular type by name; receive it) from the binding and
administration use cases (install an `ObjectFactory` that knows how to
make an `Object` of a particular type; bind that `ObjectFactory` to a
particular name), to such a degree that there could have been two
specifications instead of one.

Operations such as conversion and object binding are deliberately left
unspecified, as it should be.  That is, any given `ObjectFactory` may
do whatever it wishes to come up with an `Object` for a given name
resolution request.

At an abstract level, a compound name is a typed path, whose
intermediate nodes may have known types as well.

If we strip some of the terminology and API cruft away we can say the
following:

* In JNDI and configuration systems, there is some kind of root object
  to which notional paths are relative.
    * In JNDI this is a `Context` and the rootiest of all root
      `Context`s is an `InitialContext`, as the name accurately
      reflects.
* In JNDI and configuration systems, you locate an `Object` by
  supplying that root object with a typed path that designates the
  `Object` to retrieve.
    * In JNDI this is a `CompoundName` supplied to the
      [`Context::lookup`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/Context.html#lookup(javax.naming.Name))
      method, and if you need to figure out what the type is, you can
      invoke any of several crufty methods to find the relevant
      `Class`.
* In JNDI and configuration systems, some accessory piece of
  technology that is registered under that typed path is actually
  responsible for making the requested `Object` come into existence.
    * In JNDI, this is an `ObjectFactory` invoked by way of the
      [`NamingManager::getObjectInstance`](https://docs.oracle.com/en/java/javase/17/docs/api/java.naming/javax/naming/spi/NamingManager.html#getObjectInstance(java.lang.Object,javax.naming.Name,javax.naming.Context,java.util.Hashtable))
      method.




