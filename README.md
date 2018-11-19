WIP: Query/Response
===================

**A messaging pattern for autonomous services and evolutionary architectures.**

Table of Contents
-----------------

- [Foreword](#foreword)
- [A simple example](#a-simple-example)
  * [Any good sci-fi books out there?](#any-good-sci-fi-books-out-there)
  * [The current top-3 books](#the-current-top-3-books)
  * [The Asimov collection](#the-asimov-collection)
  * [No book lovers out there?](#no-book-lovers-out-there)
  * [Reprise, surprise](#reprise-surprise)
  * [So what's in the library](#so-whats-in-the-library)
- [Inversion of flow](#inversion-of-flow)
- [Specification](#specification)
  - [Intent](#intent)
  - [Components and Collaborators](#components-and-collaborators)
      + [Publisher](#publisher)
      + [Consumer](#consumer)
      + [Query](#query)
      + [Response](#response)
      + [Address](#address)
  - [Methods and Actions](#methods-and-actions)

Foreword
--------

_Sometime around 2015 I came across a presentation with [Fred George][1010],
 about the [Challenges in Implementing Microservices][1020]. It's a great
 talk, with lots of _lessons-learned_ that are really relevant; still are
 today, if you ask me. Experience comes from learning through failures, and at
 this point in time I was well aware of the problems that come with services
 using REST-ful APIs to call other services. I had already seen how latencies
 could spike and availability get lost, as calls from service A to B were
 actually depending on service B calling service C. It was a mess._

_In his talk George lands at the question "Synchronous or Asynchronous?" and
 proceeds to describe, what he calls, the "Needs Pattern". Service A would,
 instead of calling service B, publish a request, and service B would be able
 to pick it up and send back a response. After hearing this I began to think
 a lot about the effects of moving to an asynchronous way of communication
 between services. There was clearly a lot more here than just decoupling
 service endpoints and call latencies. Something more profound._

_The **Query/Response pattern**, that I arrived at, challenges developers to
 really think hard about responsibility and autonomy in architecture and
 design. It gives very few guarantees (almost none actually), which will force
 decisions about [Service Level Objectives (SLA)][1030], as well as
 resilience and availability, at a much earlier stage in the design and
 development process. It literally turns things around, which we will see._

  [1010]: https://twitter.com/fgeorge52
  [1020]: https://youtu.be/yPf5MfOZPY0
  [1030]: https://en.wikipedia.org/wiki/Service-level_objective

A simple example
----------------

Let's learn about the Query/Response pattern by walking through a small
fictional example (no pun intended). The technical context is _messaging_ and
hints at some type of broker-based setup - in theory though, any asynchronous
communication could be used. Any example data are just plain-text, to keep it
simple.

### Any good sci-fi books out there?

Let's publish a query.

    query: books.sci-fi
    reply-to: library/books.sci-fi#42

The structure above captures all the basic components that a query should
communicate. The term `books.sci-fi` expresses the published _need_, and we
can easily understand that it's a _request_ for science fiction books.

_The dot-notation is not at all required, the query can use any syntax that
fits the platform or programming language._

The query has an address where responses should be sent to:
`library/books.sci-fi#42`. This is really important, not only in order to
receive responses, but also to avoid coupling the sender to the query. We
don't need to state who's publishing the query. The `reply-to` is just an
address, a location or _mailbox_ that can be used for replies.

The address is only for this particular query, and it is made to be unique.
In this example `library/books.sci-fi#42` describes a topic `library`, and
then the unique mailbox or queue for the query with a hash-code
`books.sci-fi#42`.

#### The current top-3 books

    response: library/books.sci-fi#42
    body:
      "Neuromancer"
      "Snow Crash"
      "I, Robot"

We're in luck. We got a response! The information above represents a response
to the query we published. It's sent to the address from the query, and carries
a body or payload of information which may be of interest to us.

The response does not have to say who it's from. This allows us to think about
exchange of information, without the notion of: _"A sends a request to B,
which responds to A"_. We are making sure that the services are decoupled from
each other, by letting the response be an _optional_ message, sent to the 
_address_ instead of a reply to the _sender_. More about this later.

#### The Asimov collection

Since our query was published as a notification, we're not bound to a single
reply. We can keep on consuming any number of responses that are sent to the
address we published.

    response: library/books.sci-fi#42
    body:
      "I, Robot"
      "The Gods Themselves"
      "Pebble in the Sky"

In this response we received a list of book titles which all have the same
author. The previous was a list with popular books. This reply even has one
entry which was already in the first response we received.

This is of course not a problem, and it shows us a couple of important things.
Responses may come from different sources and contexts. This means that the
consumer of a response will have to assert the value or _usefulness_ of the
received information, and decide how to handle it.

_The structure of a response should of course conform to some common, agreed
 upon, format or data-shape. More on this later._

This is where we can understand that [Postel's Law][5010] applies. Information
should be liberally handled (interpreted), but publishing should be done with
more care and strictness. As a consumer of responses we just can't have a
guarantee that the information received is valid, well formed or not malicious.
We have to consume, convert and validate with great care. The decoupling in
the Query/Response patter has a price, and this is one part of it.

_But is a published REST-endpoint, for POST requests, that much better? I
 would argue that we still have the same requirements. To be able to handle
 requests liberally, we have to convert and validate, with great care. But
 we are coupling the client and server to each other and, what is perhaps
 even worse, we're actually allowing the client to control the writing of
 information inside the server. We have at least surrendered to that model
 of thinking. The POST is a write operation!_

_To really think, and reason, about who's controlling a write operation, can
 be a very powerful concept, in my view. And, arguably, the further away we
 can push this authority from the actual, internal, act of writing, the less
 we need to think about the complexity of both collaborators at once. This is
 of course the essence of messaging. We could still achieve this with the REST
 endpoint, but it would say that it is a lot harder to avoid thinking about
 the effect of the returned response from the POST request. Even if it is
 empty. We are caught in a lock-step or imperative model._

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
      "Neuromancer"
      "Snow Crash"
      "I, Robot"

Hey, what's this! We now receive the same response and body payload, as
before. This is still not a problem, nor a failure in the protocol. It is
not possible to deter multiple responses, even from the same Publisher, and
we as a consumer must be ready to handle it. There is nothing wrong with this
Response at all.

_Now since we've seen duplicate entries in different responses and complete
 duplication of full a Response, we can easily understand that a consumer
 implementation cannot just keep a list. It would suffice to use a set, and
 any duplicate entries will only be kept once._

#### So, what's in the library?

Now let's see what we have.

    query: library.sci-fi
    reply-to: bookshelf/library.sci-fi#1337

A new Query is published and we understand the `query` term to mean that
there's a _need_ or interest in knowing what books are in the library. A
successful scenario could arrive at the following Response being received.

    response: bookshelf/library.sci-fi#1337
    body:
      "Neuromancer"
      "Snow Crash"
      "I, Robot"
      "The Gods Themselves"
      "Pebble in the Sky"

Just as expected.

### Inversion of flow

What we've seen in this example scenario is actually an inversion of what
could have been implemented as a tightly coupled, chained set of synchronous
service calls:

> A User whishes to view a list of science fiction books through the
> `Bookshelf` service, which needs to call the `Library` for the list. The
> `Library` service aggregates all sci-fi books by calls to 2 configured
> services: `Top-3` and `Asimov`. Only after both service calls return, can
> the `Library` respond to the `Bookshelf` and the User is presented with
> a list of sci-fi books.

In this type of system, not only are the calls aggregated in time, effectively
forcing the user to wait until all calls return, but also to the availability
and fault-tolerance of each service. This accumulates at the point of the user,
making it highly probable that viewing the list of books will fail.

_There are many ways to work towards better and more resilient solutions, with
 the synchronous solution as a starting point. I'm not trying to say that it is
 the wrong model. The point I'm trying to make, is the very different way of
 thinking that the Query/Response pattern forces us into from the start._

Specification
-------------

We will attempt to describe the Query/Response pattern in a more formal way,
but without any claim of conformance to other standards or rules.

This is a pattern derived from the much greater, or wider, idea of expressing
a _need_ or _demand_, as previously told. It is shaped here, into our specific
version, in the **Query/Response** protocol. This is just our particular
_flavour_ of inversion of flow and asynchronous message based communication.

Please, take what you like, leave the rest, and extend as you seem fit.

_We try to adhere to [RFC 2119][3010] when using the keywords: "MUST",
 "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
 "RECOMMENDED",  "MAY", and "OPTIONAL"._

### Intent

The Query/Response pattern, described here, aims to support communication
and information sharing, in distributed systems. It does so by firmly trying
to decouple system actors, by establishing an asynchronous, message based, high-level, exchange protocol.

We currently only strive to describe a shared structure, and a sequence of
communication, to provide a small set of rules, which can be authoritative
to implementors and developers using the pattern.

### Components and Collaborators

#### `Publisher`

The active component, service or collaborator, with the capacity to publish or
broadcast a notification or message, for other to _see_.

We like to avoid binding the Query/Response pattern to any specific transport
layer, but some form of _messaging_ SHOULD be available to the Publisher.

A Publisher MUST NOT be required to know about any Consumers (see below).

There are two types of publishers, the Query- and the Response-Publisher.

#### `Consumer`

An active component, service or collaborator, with the capacity to, on its own
initiative, read notifications or messages of its choice. The consumer SHALL
willingly _subscribe_ to these notifications.

A Consumer MUST NOT be required to know about any Publishers.

There are naturally also two types of consumers, the Query- and the
Response-Consumer.

#### `Query`

A message or notification that expresses a specific _need_ or _whish_, which
can be fulfilled through a later, at any time, published Response. No strict
syntax is specified, rather the context, platform, transport layer and/or
programming language, should guide users to a suitable format.

Most typically a string or term is used, closely related to event names or
message names (_routing key_).

Expressions or terms within a domain specific, or ubiquitous language, are
commonly used to communicate the intent of the Query. It is not uncommon to
use dot-notation or some syntax with brevity, like: `product.images.4123`
or `scores.today.highscores`. This is an example of structure and parameters,
an API built into the Query - this is an accepted pattern and style within
the Query/Response pattern itself.

The Query MUST specify an Address for responses, which SHOULD be appropriate
for consumption.

#### `Response`

A message or notification that is published as a response to a previously
published Query. The Response MUST NOT be sent without a distinct Query being
seen (use normal event notifications for that instead). The Response is not
strictly bound to the time frame of the Query. Nevertheless, it is not
recommended to publish a Response outside of the time-window, or cadence, that
the given problem domain has.

For example, a Response within shipping and logistics may still be of interest
after several minutes of seeing a Query. In online-trading, responses older
than seconds, from the time of the Query, are of no value at all.

The Response MUST be designated to the Address of the Query it responds to,
the correlation must be specific to comparability or equality. In most cases
though, this will be implicit in the transport method. For example messaging
over [AMQP][3050] uses a _routing key_, which is then given in the Query, as
a `reply-to` property. A Response sent to that Address is then either routed
correctly or dropped by the broker.

The Response SHOULD provide a body of information, such as a payload of data.
Please note that this is not a strict requirement, and the Consumer is always
required to assert the received information, for validity and usefulness.

#### `Address`

Describes and designates a _context_, most commonly manifested by a point of
delivery, a mailbox, queue or place to send notifications or messages.

The Address MUST NOT describe a system actor or collaborator, but rather a
resource or function that silently accepts information. The Address SHOULD
act as a decoupling device between a Publisher and a Consumer.

Commonly, in messaging or broker based systems, an Address is typically a
routing key, topic or queue-name.

  [3010]: https://www.ietf.org/rfc/rfc2119.txt
  [3050]: https://www.rabbitmq.com/specification.html

### Methods and Actions

#### `Query-Publisher` prepares an `Address`

If there's an intent to publish a Query, the Query-Publisher SHOULD ensure
that there are suitable resources allocated to accommodate a specific
Address for any responses to the Query.

It is left to the implementation to decide on the temporal binding between
the _intent_ and the creation of resources. It may be that the only option is
to use short-lived or temporary resources, which may or may not fail to be
allocated. Therefore there's no strict requirement to ensure that the Address
can be handled. It is assumed that a _best effort_ is made.

#### `Query-Publisher` publishes a `Query`

At any time can any Query-Publisher choose to express a _need_ and publish
a Query. No ACK or NACK will be provided and the Query-Publisher MUST NOT
assume that the Query has been consumed, or that any Response will be
returned, at this time.

The Query-Publisher SHOULD entertain the case where the Query is lost, examine
options to detect and repair this, if possible. Timeouts, retries or fallbacks
are perhaps options to investigate.

#### `Query-Consumer` consumes a `Query`

A Query-Consumer, that is willingly listening for queries, may at any time
receive, and choose to handle, a Query.

The Query-Consumer SHOULD handle Queries with an intent to _do what is right_.
In most cases this would mean that Query-Consumers handle queries which they
are capable of providing responses for.

_Please note that the Query/Response protocol does not protect against
Query-Consumers with harmful intent. Implementations should consider issues
like security, encryption and trust as extensions to it._

A Query-Consumer MAY decide, after handling the Query, to publish none, one
or any number of responses - it is optional.

#### `Response-Publisher` publishes a `Response`

The Response-Publisher MUST use the provided Address of the Query when
publishing responses. No ACK or NACK will be provided and the
Response-Publisher MUST NOT assume that the Response has been consumed, at
this time.

#### `Response-Consumer` consumes a `Response`

A Response-Consumer that is listening for responses, at a previously created
Address MAY at any time receive one or several responses. It MAY never receive
a Response at that Address.

A received Response MAY have a payload or body of information, as well as any
header-elements, META-data or other properties which may be specific to the
transport layer. The Response-Consumer SHOULD assert and validate any
transferred information with great care.
