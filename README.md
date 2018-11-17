Query/Response
==============

**A messaging protocol for evolutionary architectures.**

_Sometime around 2015 I came across a presentation with [Fred George][1010],
 discussing the [Challenges In Implementing Microservices][1020]. It's a great
 talk, filled with a lot of experience and relevance, also today if you ask me.
 Experience comes from learning through failures, and at this point in time I
 was well aware of the problems that come with services using REST-ful APIs to
 call other services. I had already seen how latency and availability got out
 of hand, as calls from service A to B were really depending on service B
 calling C._

_In his talk George lands at the question "Synchronous or Asynchronous?" and
 proceeds to describe, what he calls, the "Needs Pattern". Service A would,
 instead of calling service B, publish a request notification, and service B
 would be able to pick it up and send back a response. After hearing this I
 began to think a lot about the effects of moving to an asynchronous way of
 connecting services. There was clearly a lot more here than just decoupling
 services from each other temporally, and from knowing about each other._

_The Query/Response pattern, that I arrived at, challenges developers to think
 much more about responsibility and autonomy, in design and architecture. It
 gives very few guarantees, which will force decisions about service objectives
 much earlier in a design and development process. It literally turns things
 around, which we will see._

  [1010]: https://twitter.com/fgeorge52
  [1020]: https://youtu.be/yPf5MfOZPY0