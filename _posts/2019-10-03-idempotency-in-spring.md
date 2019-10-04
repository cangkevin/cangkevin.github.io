---
layout: post
title: "Implementing Idempotency in a Spring Application"
author: Kevin Cang
excerpt_separator: <!--preview-->
tags: ['Idempotency', 'Spring']
---
Idempotency is an incredibly powerful concept both in theory and practice. As the general trend of more applications being built and deployed onto the cloud continues, I would argue that idempotency should be a requirement that must be accounted for from the initial design to the final implementation.

Of course, that would be the path to take if building something from scratch. If working with an existing application, an overhaul of the existing logic might not be an option. However, it is possible to "bolt-on" additional logic during runtime to achieve more robustness using Spring AOP.

<!--preview-->
# Idempotency 101
Idempotency is a property of an operation being invoked multiple times but only the first invocation applies the side effects. In the world of applications being deployed in the cloud, there are all kinds of failures that can happen: clients or servers crash, connections drop, services restart, etc.
In such a hostile world, uncertantities can arise on the client side if a definitive response was not received from a server. A classic example would be a client sending a payment processing request to an API endpoint and not receiving a definitive response back. Various failure scenarios come to mind:
- Did the server not receive the client's request at all?
- Did the server successfully receive and process the client's request but was not able to successfully communicate back to the client?
- Did the server successfully recieve the client's request but was not able to process it fully?

Depending on the failure scenario, it may or may not be safe for a client to retry the operation again; the first scenario can definitely allow for the client to fire off the request again without consequence while the second scenario will likely not be safe.
The third scenario depends on the underlying implementation and semantics of the operation so it may or may not be safe. However, in all three scenarios, a client might not be able to distinguish the failure that happened - it will just be seen as a failure.

There are ways to address this problem. To get an idea of the approaches, I defer you to read these two well-written articles:
- [Using Atomic Transactions to Power an Idempotent API](https://brandur.org/http-transactions)
- [Implementing Stripe-like Idempotency Keys in Postgres](https://brandur.org/idempotency-keys)

The above articles should provide insight on how you can design and implement your idempotency solutions. However, the implementation described in the articles doesn't necessarily translate nicely into an object-oriented approach or a Spring-like approach.

# Bolting on idempotency logic
If you're implementing idempotency logic in a Spring application, this can be done conveniently with Spring AOP. Although commonly used for cross-cutting concerns such as logging and validation, AOP at its core is the ability to inject additional logic at runtime. This is nice because it provides two advantages:
- Eliminates the need to significantly modify existing business logic already implemented
- Minimizes the clashing of business logic with idempotency logic

The overall approach with AOP would be something along these lines:
- Identify various points in your business logic where idempotency logic will need to be injected (i.e. performing a check on the idempotency key)
- Construct point cuts that target the various points
- Write idempotency logic that handle the various cases per the idempotency key's design for the application.

The approach is straight-forward. The complexity really lies in the design of your idempotency key in the context of your business application - what kind of foreign mutations does your application perform, are the foreign mutations themselves idempotent, how should business logic alter based on the state of the idempotency key, etc.

As mentioned, this approach allows you to add new code without modifying a significant amount of existing code, which adheres to the [open-closed principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle).
