---
layout: single
toc: true
toc_sticky: true
title:  "Validating your microservice contracts at compile time"
date:   2023-05-26 22:00:00 -0300
---

One of the challenges that microservice architectures hold is that, since code is distributed, it becomes harder to verify all the pieces work together properly.

You may say, that's why you will need to write integration tests! And that's right! However, integration tests are usually harder to write and more expensive to run than unit tests or integration tests in simpler architectures (e.g. a monolithic application). Those aspects multiply whenever you add more services and when your services are managed by multiple teams. Therefore, while its definitely wise to plan integration tests for your use cases, you will be most likely limited in the kind and amount of tests you will be able to add.

The second tool that will help you to verify proper integration is contract testing. There are many ways to check contracts between multiple services, but the underlying idea is simple: we want to verify that whenever we have an interaction between two services, that contracts (i.e. the methods and payloads structures) are respected. In this blog post we are presenting an scalable approach to contract testing at compile time, which relies on Typescript, NestJS and OpenAPI.

# What is OpenAPI?

OpenAPI is an open standard for defining HTTP service contracts. It's the de-facto standard for defining JSON apis, specially in the Javascript ecosystem. While a bit more limited, it also has support of XML, and there's tooling for other languages.

There are two opposing paradigms when it comes to writing and maintaining your open api specs: design-first and code-first.

Under design-first paradigm, you would typically work on your specification before coding your actual service. In to ensure the specification is accurate you would then be forced to add some piece of software that would help you verify that your application actually respects the specification. Code-first on the other hand, as the name suggests, is based on the idea that the development should focus on the actual code itself first. Then at some point you would write the docs. Here we will be explore the code-first paradigm on NestJS. We won't be expanding on the trade-offs involved on choosing each ones of these alternatives, but if you are interested in knowing more there's an excellent article [here](https://apisyouwonthate.com/blog/theres-no-reason-to-write-openapi-by-hand/).

# Challenges of the code-first paradigm

There is no single way to get code-first done. That said, it is certainly quite common that there will be some tooling involved that would help you build the specification from metadata that lives in the code itself. This metadata sometimes exists in the form of special comments or annotations:

____ EXAMPLE 1 _____

The main drawback of this approach is that there's no way to guarantee that these special artifacts in the code correctly reflect the behaviour of the actual service. To overcome these problems some other tools have arised, like RSwag, that help you build your specification in the form of special unit tests, what ensures that every endpoint that is part of your specification is backed up by a special unit test:

____ EXAMPLE 2 ______

While "the RSwag way" certainly improves the maintainability of your solution, it also adds the need for you to write these tests, which creates overhead to the development process. It also means that you not only need to learn OpenAPI, but also now you will also have to be versed on RSwag DSL as well.

# NestJS to the rescue!


