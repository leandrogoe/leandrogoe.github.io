---
layout: single
toc: true
toc_sticky: true
title:  "Validating your microservice contracts at compile time"
date:   2023-05-26 22:00:00 -0300
---

A while ago the company I work for started to push the engineering team to move closer to microservice-oriented architecture. While microservices do have advantages in some areas, they also pose a set of challenges. One of these challenges is that, now that your code is distributed, it becomes harder to verify all the pieces work together properly.

You may answer, that's why you will need to write integration tests! And that's right! However, integration tests are usually harder to write and expensive to run than unit tests or integration tests in simpler architectures (e.g. a monolithic application). Those aspects multiply whenever you add more services and when your services are managed by multiple teams. Therefore, while its definitely wise to plan integration tests for your use cases, you will be most likely limited in the kind and amount of tests you will be able to add.

The second tool that will help you to verify proper integration is contract testing. There are many ways to check contracts between multiple services, but the underlying idea is simple: we want to verify that whenever we have an interaction between two services, that contracts (i.e. the methods and payloads structures) are respected. In this blog post we are presenting an scalable approach to contract testing at compile time, which relies on Typescript, NestJS and OpenAPI.

# What is OpenAPI?

OpenAPI is an open standard for defining HTTP service contracts. It's the de-facto standard for defining JSON apis, specially in the Javascript ecosystem. While a bit more limited, it also has support of XML, and there's tooling for other languages.

There are two opposing paradigms when it comes to writing and maintaining your open api specs:
