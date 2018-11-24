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
  * [Inversion of flow](#inversion-of-flow)
- [Specification](#specification)
  - [Intent](#intent)
  - [Components and Collaborators](#components-and-collaborators)
      + [Publisher](#publisher)
      + [Consumer](#consumer)
      + [Query](#query)
      + [Response](#response)
      + [Address](#address)
  - [Methods and Actions](#methods-and-actions)
      + [Prepare Address](#prepare-address)
      + [Publish Query](#publish-query)
      + [Consume Query](#consume-query)
      + [Publish Response](#publish-response)
      + [Consume Response](#consume-response)
- [The example revisited](#the-example-revisited)
  * [A better library protocol](#a-better-library-protocol)
  * [Top-3 books have stars](#top3-books-have-stars)
  * [One of each flavour](#one-of-each-flavour)
  * [Out with the old](#out-with-the-old)
- [Query/Response Maturity Model](#queryresponse-maturity-model)
  * [Level 0 - Purgatory](#level-0---purgatory)


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
communication could be used. The examples are only pseudo-code and plain-text
data, to keep things simple.

### Any good sci-fi books out there?

Let's publish a query.

    query: books.sci-fi
    reply-to: library/books.sci-fi#42

The structure above captures all the basic components that a query should
communicate. The term `books.sci-fi` expresses the published _need_, and we
can easily understand that it's a _request_ for science fiction books.

_The dot-notation is not at all required, the query can use any syntax that
fits the platform or programming language._

The query has an address where responses should be sent back to:
`library/books.sci-fi#42`. This is really important, not only in order to
receive responses, but also to avoid coupling the sender to the query. We
don't need to state who's publishing the query. The `reply-to` is just an
address, a location or _mailbox_ that can be used for replies.

The address is only for this particular query, and it is made to be unique.
In this example `library/books.sci-fi#42` describes a topic `library`, and
then the unique mailbox or queue for the query with a hash-code
`books.sci-fi#42`.

### The current top-3 books

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

### The Asimov collection

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

Considering all this, we need to remember [Postel's Law][5010]. Information
should be liberally handled (interpreted), but publishing should be done
more conservatively. As a consumer of responses we just can't have a
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

_To really think and reason about who's controlling the write operation, can
 be a very powerful concept in my view. And arguably, the further away we
 can push this authority from the actual, internal act of writing, the less
 we need to think about the complexity of both collaborators at once. This is
 of course the essence of messaging. We could still achieve this with the REST
 endpoint, but I would say that it is a lot harder to avoid thinking about
 the effect of the returned response from the POST request. Even if it is
 empty. We are caught in a lock-step or imperative model._

  [5010]: https://en.wikipedia.org/wiki/Robustness_principle

### No book lovers out there?

Let's rewind the scenario a bit. Let's say we've just published the query,
but no responses arrive. What should we do?

This is not a flaw in the design, but a specific part of the Query/Response
pattern. It is always up to the consumer of responses (the one that sent
the query), to decide _how long_ it will continue to read, or wait for any to
arrive at all. The pattern does not force this or make any promises.

There might be responses. There may be none, a single one or a huge amount.
This is by design, and it forces us to think about important questions, early
in development. Fallback values, proper defaults, circuit-breakers and how
to deal with a flood of responses.

_The most commonly asked question, by developers new to the Query/Response
 pattern, is: "But what if there are no responses, what do I show the user?".
 Exactly! Plan for that. This is something that should be considered early
 in design and development. There might very well be a response, eventually,
 but how long do you let the user wait for a result?_

### Reprise, surprise

Back to our original scenario. We've received both the top-3, as well as
a collection of Asimov books. And we're still open for more responses to the
published address.

    response: library/books.sci-fi#42
    body:
      "Neuromancer"
      "Snow Crash"
      "I, Robot"

Hey, what's this! We now received the same response and body payload, as
before. This is still not a problem, and it's not a flaw in the pattern. It
is not possible to avoid multiple responses, even from the same publisher. As
a consumer, we have to be ready to handle it. There is nothing wrong with this
response at all.

_The consumer must handle this, and can't keep the entries in a simple list. If
 we did, it would contain several duplicate entries. It would be enough to use
 a set instead, so any duplicate entries would only be kept once._

### So, what's in the library?

Let's see what we have.

    query: library.sci-fi
    reply-to: bookshelf/library.sci-fi#1337

A new query is published and we understand the `query` term to mean that
there's an _interest_ in knowing what books are in the library. A successful
scenario could arrive at the following response being consumed.

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

> A user whishes to view a list of science fiction books through the
> `Bookshelf` service, which needs to call the `Library` for the list. The
> `Library` service aggregates all sci-fi books by calls to 2 configured
> services: `Top-3` and `Authors`. Only after both service calls return, can
> the `Library` respond to the `Bookshelf` and the user is presented with
> a list of sci-fi books.

In this type of system, not only are the calls aggregated in the total time,
effectively forcing the user to wait until all calls return, but also to the
availability of each service. This accumulates at the point of the user,
making it highly probable that viewing the list of books will fail.

_There are many ways to work towards better and more resilient solutions, also
 in the synchronous solution. I'm not trying to say that it is the wrong
 model. The point I'm trying to make, is the very different way of thinking
 that the Query/Response pattern forces us into from the start. Availability,
 fallbacks, resilience and strict timeouts are called out as key-concepts._

_I hope this illustrates what's possible using this approach and that I've
 sparked at least som interest in the Query/Response pattern. Later I will
 extend on some of the features and caveats._

Specification
-------------

I'd like to describe the Query/Response pattern in a more formal but not
too strict way, since it's not in any way some type of _standard_ or
_protocol_. This is a pattern derived from the general idea of expressing a 
_need_ or _demand_, as previously told. It is shaped here, into a specific
version, or flavour, in the **Query/Response pattern**. It simply contains
my recommendations and suggestions on rules or principles to follow.

Please, take what you like, leave the rest, and extend as you seem fit.

Use of the keywords: "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" are intended
to follow the definitions of [RFC 2119][3010].

  [3010]: https://www.ietf.org/rfc/rfc2119.txt

### Intent

The Query/Response pattern aims to describe a model for information sharing
in a distributed system. It does so by using strong decoupling of system
actors and establishing asynchronous message-based high-level data exchange,
as the only means of communication.

The following specifications tries to provide a set of rules and guides, 
which can be used as an authoritative source for developer, implementing the
pattern.

### Components and Collaborators

| Name         | Type        | Description                         |
|--------------|-------------|-------------------------------------|
| `Query`      | message     | Very small, published notification. |
| `Response`   | message     | Carries information as payload.     |
| `Address`    | location    | Reference to "a mailbox"            |
| `Publisher`  | actor       | Initiates _publish_ method calls.   |
| `Consumer`   | actor       | Accepts _consume_ method calls.     |

#### `Query`

A notification that expresses a specific _need_ or _whish_, which can be
fulfilled by a response, published to a specified return address. The query
MUST state its _need_ or _whish_ in an interpretable way. It may use any
suitable syntax, semantics or language. Most commonly a simple string or term
is used, similar to a message subject, -name or an event _routing-key_. A
query MUST specify an address for responses, which SHOULD be _appropriate_
for the stated query and, technically _available_, as the query is created.

_I very much recommend creating queries with expressions or terms from a
 domain specific, or ubiquitous language. This allows for broader understanding
 and involvement of stakeholders. Keeping queries human readable makes sense.
 It's often desirable to use structured terms, with semantics, such as
 filters or parameters. This is quite common and not at all bad._

#### `Response`

A notification, published, as a response to a query, optionally carrying an
information- or data-payload. A response MUST NOT be sent without an intent to
_answer_ a specific query (use event notifications for that). The response
MUST be sent to the address of the query it responds to, without manipulating
it. A response SHOULD carry an appropriate information- or data-payload, with
the intent to answer the query it responds to. Note that this is not a strict
requirement. Responses SHOULD be sent within an appropriate time frame of
seeing a query.

_In most cases it's desirable to publish a response as quick as possible,
 after consuming a query._

#### `Address`

Describes and designates an addressable _location_ with the capability to
receive and handle responses. Typically a messaging _mailbox_ or a queue. The
address MUST NOT describe a system actor or collaborator, but instead ensure
decoupling between a publisher and a consumer.

_In messaging or broker based systems, the address is typically a routing key,
 topic or a queue-name._

#### `Publisher`

An actor that initiates the publishing of a notification, either a query or
a response depending on its current role. The publisher MUST NOT be responsible
for the arrival of any published information. Publishers MUST NOT know any
consumers.

> NOTE: The concrete _interpolated_ roles `Query-Publisher` and
> `Response-Publisher`, does not have to be bound to a single or unique actor.

_It is open for the implementation of the Query/Response pattern to solve or
 choose how it ensures delivery of messages, e.g. using a broker- or queue-
 based messaging system or some other solution for asynchronous communication._

#### `Consumer`

An actor that willingly yields to the consumption of notifications, from some
external source, either a response or a query depending on its current role.
Consumers MUST NOT know any publishers.

> NOTE: The concrete _interpolated_ roles `Query-Consumer` and
> `Response-Consumer`, does not have to be bound to a single or unique actor.

### Methods and Actions

_Nothing in the Query/Response pattern is synchronous, or based on the notion
 of guaranteed delivery (or only-once semantics). The following structured
 step-by-step description is only for documentation purposes, and does not,
 in any way, define a sequence which can be relied upon._

#### Prepare `Address`

Before publishing a query, the query publisher SHOULD ensure that an
appropriate address, specified for the query, can be handled.

_Implementations are free to use a best-effort approach. It may be that the
only option is to use short-lived or temporary resources, which may or may
not fail to be allocated. Therefore there's no strict requirement to ensure
that the address can be handled._

#### Publish `Query`

The query publisher can, at any time, choose to publish a query. No ACK or
NACK will be provided and the query publisher MUST NOT assume that the query
has been consumed, or that a response will be returned at this time. The
publisher SHOULD consider the case where the query is lost, examine options
to detect and repair this, if possible; _timeouts, retries or fallbacks are
perhaps options to investigate_.

#### Consume `Query`

A query consumer, that is willingly listening for queries, may at any time
receive, and choose to handle a query. Consuming queries is an unbound
operation. The consumer SHOULD handle queries with an intent to provide a
response, or ignore the query. A consumer MAY decide to publish none, one or
any number of responses to the query - it is optional. A consumer MAY at any
time choose to stop listening for queries.

_Please note that the Query/Response pattern does not protect against
query consumers with harmful intent. Implementations should consider issues
like security, encryption and trust as extensions to it._

#### Publish `Response`

A response publisher MUST use the provided address of the query it responds to,
when publishing responses. No ACK or NACK will be provided and the publisher
MUST NOT assume that the response has been delivered, arrived properly or
consumed.

#### Consume `Response`

A response consumer, listening for responses at a previously created address,
MAY at any time receive one or several responses - or not at all. Consuming
responses is an unbounded operation. Any received response MAY have a payload
or body of information. The consumer SHOULD assert and validate any
transferred information with great care. A consumer MAY at any time choose to
stop listening for responses.

The example revisited
---------------------

Let's examine one of the most powerful aspects of using the Query/Response
pattern. If we think back to our initial example we published a query for
books in the sci-fi genre.

    query: books.sci-fi
    reply-to: library/books.sci-fi#42

We also learned that responses may come from different sources, with different
payloads and we are responsible for dealing with validation and duplicates etc.

The query in this example uses only some minimal semantics to express the
genre of books requested, the term `sci-fi`. This is part of a contract from
our domain, together with rules on how any result payload should be presented.
The list of strings within quotes are not by accident, it is also by design.

The Query/Response pattern does not enforce any structural rules for query,
address or response syntax. This must come from designers and developers. _I
would suggest, using [Domain Driven Design][8010] to leverage the power of
a ubiquitous language in the queries_.

All this together puts us in a position to allow change and evolution in our
system.

  [8010]: https://en.wikipedia.org/wiki/Domain-driven_design

### A better library protocol

We have agreed on supporting _stars_ for book ratings, and different teams
scramble to their stations to extend for the new feature.

We saw earlier that data returned was formed as a list of quoted strings, and
the contract for parsing was: "first quoted string per line is book title".

    body:
      "Neuromancer"

That rule and the capability to extend it, made it possible to agree on a new
optional format: "trailing key-values are properties". For example:

    body:
      "Neuromancer" isbn:9780307969958 stars:4

This is great. Let's get to work.

### Top-3 books have stars

    query: books.sci-fi
    reply-to: library/books.sci-fi#77

At a later time a new query for science fiction books is published. Now, we
still must not assume anything about the service or collaborator publishing
the query. It may be that we have a new service running in our system, not yet
live, or an updated version of the first one - we don't need to know.

    response: library/books.sci-fi#77
    body:
      "Neuromancer" stars:3
      "Snow Crash" stars:5
      "I, Robot" stars:4

The first response looks great, it's using the new extended protocol and
provides star-ratings with the top-3 sci-fi book list.

### One of each flavour

Another response is consumed:

    response: library/books.sci-fi#77
    body:
      "I, Robot"
      "The Gods Themselves"
      "Pebble in the Sky"

Oh, ok seems that we've received a response with only Asimov books again, and
sadly no stars. Luckily the protocol rules allows us to still use the response
if we choose to.

    response: library/books.sci-fi#77
    body:
      "I, Robot" stars:2
      "The Gods Themselves"
      "Pebble in the Sky" stars:5

And what is this now. We've consumed yet another response and it appears to be
the Asimov list again, but this time with star-ratings, but only for a few
titles.

This is quite normal and shows us a really important and valuable aspect of
the Query/Response pattern. If we would pull the curtain back a bit, it could
be reasonable to assume that the publisher of Asimov books now exists in 2
distinct versions. One supports the new updated format, and has a couple of
star-ratings set. The other appears to be the _older_ version.

We have effectively seen how response publishers can evolve, and even exist
side-by-side, if care is taken to design a suitable payload protocol.

_The backward compatibility of the payload format is not at all required in the
 Query/Response pattern. Implementations could use version tags or classifiers
 to check for compatibility at the consumer side._

The key point here is, the consumer is still responsible for asserting the
usefulness and value of the response information. Parsing, validating or
checking for version compatibility is required.

### Out with the old

Let's jump forward and say that at some later time, the query for sci-fi books
is published again.

    query: books.sci-fi
    reply-to: library/books.sci-fi#88

And this time, the only consumed response with Asimov books is the following:

    response: library/books.sci-fi#88
    body:
      "I, Robot" stars:3
      "The Gods Themselves" stars:3
      "Pebble in the Sky" stars:5

We can almost certainly conclude that the original version of the Asimov
book service has been shut down.

Again we can see how the Query/Response pattern helps in coping with a natural
evolution of the system. Services can be added, removed or upgraded at any
time.

Query/Response Maturity Model
-----------------------------

Just like with the [Richardson Maturity Model][9010], I've identified an
evolution of maturity around the acceptance, use and implementation of
Query/Response. It describes the benefits, opportunities and also
complexities, pretty well.

  [9010]: https://martinfowler.com/articles/richardsonMaturityModel.html

### Level 0 - Purgatory

All communication and exchange is bound to fixed, configured, service end-
points. Synchronous blocking calls exchange information based on formats
declared in project Wiki-pages or Word-documents. Changes typically
require system wide, synchronized, upgrades. This lead to development dropping
in velocity, as each module or team will find it hard or impossible to
act independently of each other.

### Level 1

Using the Query/Response pattern for the first time often leads to healthy
temporal decoupling pretty quick. But with a lot of code still written with
a synchronous model in mind, the data exchange tend to look a bit like _sync_.
Already at this level teams and modules gain a lot in the capability to move
independently. Releases and deployment is practically not a tangle any more,
although the view on evolutionary data-structures or protocols for data, may
lag behind and still be Wiki/Document-based.

