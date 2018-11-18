WIP: Query/Response
===================

**A messaging protocol for evolutionary architectures.**

_Sometime around 2015 I came across a presentation with [Fred George][1010],
 discussing the [Challenges In Implementing Microservices][1020]. It's a great
 talk, filled with a lot of experience and relevance, also today if you ask me.
 Experience comes from learning through failures, and at this point in time I
 was well aware of the problems that come with services using REST-ful APIs to
 call other services. I had already seen how latency could get out of hand and
 availability get lost, as calls from service A to B were really depending on
 service B calling service C. It was a mess._

_In his talk George lands at the question "Synchronous or Asynchronous?" and
 proceeds to describe, what he calls, the "Needs Pattern". Service A would,
 instead of calling service B, publish a request notification, and service B
 would be able to pick it up and send back a response. After hearing this I
 began to think a lot about the effects of moving to an asynchronous way of
 communicating with services. There was clearly a lot more here than just
 decoupling services from each other temporally, and from knowing about each
 others locations._

_The Query/Response pattern, that I arrived at, challenges developers to think
 much more about responsibility and autonomy, in design and architecture. It
 gives very few guarantees (almost none actually), which will force decisions
 about service objectives and resilience at a much earlier stage in the design
 and development process. It literally turns things around, which we will see._

  [1010]: https://twitter.com/fgeorge52
  [1020]: https://youtu.be/yPf5MfOZPY0

A simple example
----------------

Let's dive in and get to know the Query/Response pattern from a more practical
point of view first. The protocol is first and foremost _message based_ but any
asynchronous communication implementation can, in theory, be used. I'll present
everything using pseudo-code and stupid-simple data structures only.

### Any good sci-fi books out there?

    query: books.sci-fi
    reply-to: library/books.sci-fi#42

The structure above captures all the basic components that a Query should
communicate. The term `books.sci-fi` expresses the published _need_, and in
our example we can quite easily understand that it's about books in the 
genre or category science fiction.

_The dot-notation is not at all required, the Query can use any syntax that
fits the platform or programming language._

In the published Query we've made sure to add an _address_ or designation as
to where responses should be directed `library/books.sci-fi#42`. This is
important, not only in order for information to arrive at the right place,
but we also want to avoid coupling the sender to the Query. We don't need to
state who's publishing the Query. The `reply-to` _address_, therefore, is
simply just that: an address, a place, a location or _mailbox_ that we can
send responses to.

In our example we've provided the address `library/books.sci-fi#42`. We have
made it unique for our specific query, with the label or key (overly
simplified here). The term `books.sci-fi#42` is only used for responses to
our specific Query. Furthermore we are not required to _keep_ or guarantee
the existence of the given address for any specified time, more about that
later.

#### The current Top-3 books

    response: library/books.sci-fi#42
    body:
      Neuromancer
      Snow Crash
      I, Robot

We're in luck, we've got a response! The information above represents a
Response to the Query we published. It is directed back to the address from
the Query, and carries a body or payload of information which may be of
interest to the Query publisher.

The response does not have to convey its publisher, which allows us to reason
about exchange of information without the notion of _"A sends a query to B,
which in turn sends a responds to A"_. We are ensuring decoupling of the
collaborators by having the pattern allow for an _optional_ Response to the
provided _address_, rather than a required reply to the _sender_. More about
later.

#### The Asimov collection

Now, since our Query is published as a notification, and since we're not bound
to a single reply, we can simply keep on consuming any Response sent back
to the provided address.

    response: library/books.sci-fi#42
    body:
      I, Robot
      The Gods Themselves
      Pebble in the Sky

In this payload we receive a list of book titles which all have the same
author. One of the entries was already in the first Response we received.
This is of course not a problem, and it shows us something important. The
consumer of a Response will have to assert the value of the received
information, and decide how to handle it.

In this sense [Postel's Law][5010] applies, information should be liberally
handled (interpreted), but publishing information should better be done with
strict rules. As a consumer of replies we can never be certain of a strict
conformity of the payload information - the model of decoupling has a price,
and this is it.

_But is a published REST endpoint, handling POST requests, that much better
 then? I would argue that we still have the same requirements. Being able to
 handle requests liberally. We must always convert and validate, with great
 care. But we are coupling the client and server to each other and, what is
 perhaps even worse, we're actually asking the client to control the
 imperative writing of information in the server._

_To think about who's controlling the write operation, is a tremendously
 powerful concept, in my view. And, arguably, the further away we can push
 this from the actual act of writing, the less we need to think about the
 complexity of both collaborators at once. This is of course the essence
 of messaging. We could still achieve this with the REST endpoint, but it
 is a lot harder to avoid thinking about the effect of the returned response
 from the POST request. Even if it is empty or `void`._

  [5010]: https://en.wikipedia.org/wiki/Robustness_principle

#### No book lovers out there?

Let's rewind the scenario a bit. Let's say we've just published the Query,
and no responses arrive at once. What should we do?

This is not a flaw in the design, but a specific part of the Query/Response
pattern. It is always up to the _consumer of responses_ (the one that sent
the Query to begin with), to decide how long it will continue to wait for,
or read, responses. The protocol gives no guarantees at all.

There may be responses. There might be none, just a single one or a huge
amount. This is by design, and it forces us to think about important
questions, early in development. Fallback values, proper defaults, circuit-
breakers and how to deal with a flood of responses.

_The most commonly asked question from developers new to the Query/Response
 pattern is: "But what if there's no response, I have to return something to
 the user?". Exactly, plan for that, this is something that should be
 considered early in design and development. There might very well be a
 Response, eventually, but how long is it then ok to have the user wait for
 a result?_

#### Reprise, surprise

Back to our original scenario. We've received both the top-3, as well as
a collection of Asimov books. And we're still open for more responses to the
published address.

    response: library/books.sci-fi#42
    body:
      Neuromancer
      Snow Crash
      I, Robot

Hey, what's this! We now receive the same response and body payload, as
before. This is still not a problem, nor a failure in the protocol. It is
not possible to deter multiple responses, even from the same Publisher, and
we as a consumer must be ready to handle it. There is nothing wrong with this
Response at all.

_Now since we've seen duplicate entries in different Responses and complete
 duplication of full Responses, we can easily understand that a consumer
 implementation cannot just keep a list. It would suffice to use a set, and
 any duplicates will only be kept in one entry._

#### So, what's in the library?

Now let's see what we have.

    query: library.sci-fi
    reply-to: bookshelf/library.sci-fi#1337

A new Query is published and we understand the `query` term to mean that
there's a _need_ or interest in knowing what books are in the library. A
successful scenario could arrive at the following Response being received.

    response: bookshelf/library.sci-fi#1337
    body:
      Neuromancer
      Snow Crash
      I, Robot
      The Gods Themselves
      Pebble in the Sky

Just as expected.

### Inversion of flow

What we've seen in this example scenario is actually an inversion of what
could have been implemented as a tightly coupled, chained set of synchronous
service calls:

> A User whishes to view a list of science fiction books through the
> `Bookshelf` service, which needs to call the `Library` for the list. The
> `Library` service aggregates all sci-fi books by calls to 2 configured
> services: `Top-3` and `Asimov`. Only after both service calls return, can
> the `Library` respond the user `Bookshelf` and the User is presented with
> the list of sci-fi books.

In this type of system, not only are the calls aggregated in time, effectively
forcing the user to wait until all calls return, but also to the availability
and fault-tolerance of each service. This accumulates at the point of the user,
making it highly probable that viewing the list of books will fail.

_There are many ways to work towards better and more resilient solutions, with
 the synchronous solution as a starting point. I'm not trying to say that it is
 the wrong model. The point I'm trying to make, is the very different way of
 thinking the Query/Response pattern forces us into from the start._

