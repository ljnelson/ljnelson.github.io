---
title: "Java Configuration: JAX-RS as a Configuration System"
date: 2022-01-01T12:45:33-08:00
draft: false
tags: ['java', 'jaxrs', 'config']
---
{{< toc >}}

## Background and Rationale

I've [written previously]({{< relref
"java-configuration-jndi-and-naming-operations" >}}) about looking at
JNDI through a Java-centric configuration system design lens.  Here
I'll do something similar with the JAX-RS specification (now known as
{deep breath} [Jakarta RESTful Web
Services](https://jakarta.ee/specifications/restful-ws/3.0/), or,
hopefully soon, simply Jakarta REST, which is how I'll refer to it in
this article).

## Jakarta REST as a Configuration System?!

Hear me out.

First, I'm not _actually_ proposing that if you want a configuration
framework in your Java program you should grab a Jakarta REST
implementation and go to town.

But I _am_ looking at it as primarily a _Java_ framework, and not as
something that is web-oriented.  After all, one of its founding goals
was to make it easy to use plain old Java objects (POJOs) to model
representational state transfers.  Who really cares if there is a
network involved or not?

## The Foundation

Jakarta REST is built atop _resource classes_.  While you should of
course consult [the
specification](https://jakarta.ee/specifications/restful-ws/3.0/jakarta-restful-ws-spec-3.0.html#resource-classes)
for the official definition of resource classes (and anything else I'm
going to wave my hands about in this article), the general gist is: a
resource class is a POJO class with some specifically shaped methods
in it, and annotated in a particular way.

For retrieval purposes (`GET`), which is all I'm interested in,
methods that do retrievals obviously need to have a return type.  The
return type can be Jakarta-REST-specific
([`Response`](https://jakarta.ee/specifications/restful-ws/3.0/apidocs/jakarta/ws/rs/core/response)),
or can be a POJO designed by the resource class designer.

The annotations on resource classes and their methods help them
declare what _paths_ they respond to, and what types of objects those
methods supply in response.

When you put this all together, you have a subsystem that receives a
typed path, finds a relevant supplier (a resource method), caches that
fact, and then uses that supplier to serve up an `Object` of some kind
that corresponds to that path (and other qualifiers).

## Configuration Concerns

When you look at Jakarta REST this way, it starts to look an awful lot
like [some of the JNDI concepts I've written about previously]({{<
relref "java-configuration-jndi-and-naming-operations" >}}).

In both cases there is a lookup operation, with name-like structures
identifying the thing to retrieve.  In both cases (although in JNDI
it's a pain in the neck) you can qualify your lookup.  In both cases
an _application assembler_ can declare explicitly how bundles of
components should be combined into an application in such a way that
naming conflicts do not occur.  In both cases, _how_ a resulting
`Object` is put together, or found, or synthesized, is completely
transparent to the caller and is deliberately unspecified.

Jakarta REST also features a wealth of additional qualifiers that ride
along with every request.  You have the path, of course, but you also
have headers (key/value pairs), MIME types, and request-level content
negotiation strategies.  If you look at these coarsely enough, they're
just qualifiers further picking out the `Object` that is being
requested.

One of the nice things about the lookup request format that Jakarta
REST uses is that in a `path/with/many/components` there is no
presumption that each component in the path designates a retrievable
resource.  (JNDI, by contrast, basically requires that a `Context`
exist at each juncture.)  This allows for sparse graphs of resources,
dynamic subresources, and all sorts of other interesting bits that end
up being directly relevant to configuration systems.

Another nice thing about the lookup request format is that the
incoming name-like structure (the request) consists not just of the
name-like thing (the path) plus its qualifiers (the headers and matrix
parameters and everything else) but also the _type_ of the object
being requested (expressed as a MIME type).  In my [earlier JNDI
article]({{< relref "java-configuration-jndi-and-naming-operations"
>}}), I noted that configuration systems involve a _typed path_ at the
heart of configuration lookup.  JNDI sort of lets you get there, but
it is awful and clunky to do.  Jakarta REST makes it reasonably easy:
by the time a resource method gets invoked, you know that MIME type
matching has already occurred according to a well-specified algorithm,
so you know that the resource method in question is equipped to
service the request.

## Disambiguation

Thankfully, the designers of Jakarta REST (well, JAX-RS, in this case)
also realized the namespace issues that always show up when you talk
about someone assembling components together into an application, and
provided for their solution.

In JNDI, namespace issues are somewhat moot, because names are always
relative to a `Context`: there's no such thing as an absolute name.
The same is not true in Jakarta REST, but the application assembler
can explicitly designate an `Application` implementation that says for
certain which Java classes are to be considered resource classes, and
which are not.  This allows two resource methods, for example, from
two different sources, annotated with the same `@Path` annotation, to
coexist: the application assembler can choose just one, can wrap the
other, or any of a variety of other strategies at assembly time to
resolve the ambiguity.

(It's worth noting that no Java-centric configuration system that I'm
aware of lets you do this fundamental disambiguation operation at
assembly time.  That's really odd.)

## Suitability

Probably the most interesting feature of Jakarta REST when looked at
through a Java-centric configuration lens is its built-in notion of
suitability.

A resource method is more or less suitable for a given request as
specified by an [exceedingly well-defined
algorithm](https://jakarta.ee/specifications/restful-ws/3.0/jakarta-restful-ws-spec-3.0.html#mapping_requests_to_java_methods).
For my purposes, the exact steps of the algorithm are unimportant.
The fact that it exists and is defined in terms of _application-level_
concerns, rather than _component-level_ concerns is what is important.
If this algorithm completes and there are somehow still two or more
candidate resource methods for a given request, then an error is
thrown.  This means that resource method selection is _deterministic_:
if you supply the same inputs, you get the same outputs every time.

Recognizing the difference between application-level concerns and
component-level concerns is critical for this kind of determinism,
because components are often developed in isolation from one another,
so sharing things like namespaces and numberspaces and pathspaces and
all the rest can be difficult.  So, for example, defining the matching
algorithm in terms of a global set of MIME types means that there can
be no name clashes between types: `application/octet-stream` means
what it means, regardless of which component uses it.

Contrast this with another popular but extraordinarily misguided
strategy of labeling some component somewhere with a numeric priority
and believing erroneously that you have somehow solved the ambiguity
problem.  Instead, you've just punted it: If component _A_ and
component _B_ are developed in isolation, and both have independently
decided to declare that they are of priority `10`, that is still a
problem the application assembler has to solve, but unless there is
yet _another_ mechanism for her to disambiguiate _this_ ambiguity, it
can lead to a non-deterministic state of affairs.  We see this, [as
I've noted earlier, in MicroProfile
Config](https://lairdnelson.wordpress.com/2021/10/14/some-of-the-things-i-dont-like-about-microprofile-config/).
(Interestingly, Jakarta REST [_did_ walk into this trap in terms of
providers](https://jakarta.ee/specifications/restful-ws/3.0/jakarta-restful-ws-spec-3.0.html#provider_priorities)
and other accessory entities, but at least they give the application
assembler a welcome "out" since she can always write an `Application`
class to make things more explicit.)

## Conclusion

Jakarta REST is a specification for web services, yes, but it is also
a specification for acquiring Java `Object`s given path-like requests,
where the potential suppliers of such `Object`s can be more or less
suitable for any given request.  This lines up pretty well with the
requirements of a Java-centric configuration system.  There are
lessons to be learned here that can be applied to the design of a
"real" Java-centric configuration system.

