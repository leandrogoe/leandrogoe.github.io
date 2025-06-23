---
layout: single
toc: false
toc_sticky: false
title:  "Seamlessly verifying API contracts at compile time with NestJS and Typescript"
date:   2023-05-26 22:00:00 -0300
---

One of the challenges that microservice architectures present is that, since code is distributed, it becomes harder to verify whether all the pieces work together properly.

While integration tests are a key instrument to address this challenge, they are usually harder to write and more expensive to run than the average automated test in simpler architectures (e.g. a monolithic application). These challenges grow whenever you add more services and when your services are managed by multiple teams. Therefore, while it's definitely wise to plan integration tests for your use cases, you will most likely be limited in the number and types of tests you will be able to add.

The second tool that will help you verify proper integration is contract verification. There are many ways to check contracts between multiple services, but the underlying idea is simple: we want to verify that whenever we have an interaction between two services, the contracts (i.e. the methods and payload format) are respected. In this blog post, we present a scalable approach to contract verification at compile time, which relies on TypeScript, NestJS, and OpenAPI.

# What is OpenAPI?

OpenAPI is an open standard for defining HTTP service contracts. It's the de-facto standard for defining JSON APIs, especially in the JavaScript ecosystem. While a bit more limited, it also has support for XML, and there's tooling for other languages.

# Writing and maintaining API specifications

There are two opposing paradigms when it comes to writing and maintaining your OpenAPI specs: design-first and code-first.

Under the design-first paradigm, you would work on your specification before coding your actual service. Code-first on the other hand, as the name suggests, is based on the idea that the development should focus on the actual code itself first and then on the specification.

Here we will explore the code-first paradigm. We won't be expanding on the trade-offs involved in choosing between these alternatives, but if you are interested in a deep dive on the subject [here is an excellent article](https://medium.com/@tiokachiu/api-first-vs-code-first-choosing-the-right-approach-for-your-project-868443e73052).

Regardless of what path you choose, you will need to somehow ensure that your API specification correctly defines the behavior of your application and that whenever changes happen you can easily keep them synchronized.

There is no single way to get code-first done. That said, it is certainly quite common that development teams rely on tools that build the specification from metadata that lives in the code itself. This metadata sometimes exists in the form of special comments or annotations. For instance, some libraries, like swagger-jsdoc, will translate your JSDoc comments into an OpenAPI specification:

```javascript
  /**
   * @openapi
   * /blogs:
   *   post:
   *     description: Creates a blog post
   *     tags: [Blogs]
   *     produces:
   *       - application/json
   *     consumes:
   *       - application/json
   *     requestBody:
   *       application/json:
   *         schema:
   *           type: object
   *           properties:
   *             title:
   *               type: string
   *             content:
   *               type: string
   *     responses:
   *       200:
   *         description: blog
   */
  app.post('/blogs', (req, res) => {
    // The actual implementation goes here
  });
```

The main drawback of this approach is that there's no way to guarantee that these special artifacts in the code correctly reflect the actual behaviour of the service. To overcome these problems some other tools have arisen, like RSwag, that help you build your specification in the form of special unit tests, which ensures that every endpoint that is part of your specification is backed up by a special unit test:

```ruby
# spec/requests/blogs_spec.rb
require 'openapi_helper'

describe 'Hello world api' do

  path '/' do

    get 'Creates a blog' do
      tags 'Blogs'
      consumes 'application/json'
      parameter name: 'blog', in: :body, schema: {
        type: :object,
        properties: {
          title: { type: :string },
          content: { type: :string }
        },
        required: [ 'title', 'content' ]
      }

      response '201', 'blog created' do
        let(:request_params) { { 'blog' => { title: 'foo', content: 'bar' } } } }
        run_test!
      end

      response '422', 'invalid request' do
        let(:request_params) { { 'blog' => { title: 'foo' } } }
        run_test!
      end
    end
  end
end
```

While "the RSwag way" certainly improves the maintainability of your solution, it also adds the need for you to write these tests, which creates overhead to the development process. It also means that you not only need to learn OpenAPI, but also now you will also have to be versed in RSwag DSL as well.

# Code first done right

The holy grail of code-first would be a system that ties the actual behaviour of your service to its specification without the need to have special artifacts that add complexity and overhead.

NestJS is an example of such a framework. A controller in NestJS looks like this:

```typescript
import { Controller, Post, Body } from '@nestjs/common';

class CreateBlogDto {
  title: string;
  content: string;
}

@Controller('blogs')
export class BlogsController {
  @Post()
  create(@Body() blogParameters: CreateBlogDto): Blog {
    const blog = dataLayer.createBlog(blogParameters);
    return blog;
  }
}
```

As you can see, it heavily relies on Typescript, both for decorators and typing. The `@Controller('blogs')` decorator tells NestJS that the controller will be handling requests to the `blogs` path. Then we also have the `@Post` decorator which states that the `create` method will be used to handle the HTTP requests that use the `POST` verb in the controller's path. Finally, the parameters are annotated with the `@Body` decorator, which tells us that the service will expect these parameters to be serialized in the request's body.

Note that all these decorators, unlike what happens with our previous JSDoc example, **actually define the service behaviour** and are not separate artifacts that will need to be maintained on top of our implementation.

That said, while originally intended to define *behaviour*, these artifacts can also be repurposed to output a *specification*. That's where the `@nestjs/swagger` package comes into play. By simply adding a few lines of code to your application bootstrapping, you can ensure an OpenAPI specification is published in one of the application paths and/or written to a file:

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import * as jsyaml from 'js-yaml';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('Blogs example')
    .setDescription('The blogs API description')
    .setVersion('1.0')
    .addTag('blogs')
    .build();
  const document = SwaggerModule.createDocument(app, config);

  const yamlDocument = jsyaml.dump(document, {
    skipInvalid: true,
    noRefs: true
  });

  SwaggerModule.setup('api', app, document);

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

Ta-da! Now when your app starts, it will also write an OpenAPI specification to a file. Pretty cool, isn't it?

Well, here is the thing: I lied to you.

<img src="/assets/images/liar.jpg">

While that setup will indeed output a specification, you may probably want to add some special artifacts whose only purpose is to add more details to your specification and have no behavioural impact whatsoever. For example you will explicitly need to add the `@ApiProperty` decorator on top of each one of your DTO attributes that you want to include in the documentation:

```typescript
class CreateBlogDto {
  @ApiProperty
  title: string;

  @ApiProperty
  content: string;
}
```

You can also add further attributes to your decorators which will further enrich your specification:

```typescript
class CreateBlogDto {
  @ApiProperty({
    description: 'The title of your blog',
    example: 'Yet another tech blog'
  })
  title: string;

  @ApiProperty({
    description: 'The contents of your blog',
  })
  content: string;
}
```

As you may be realizing, adding these second set of artifacts, which only affect the specification but are unrelated to behaviour is inevitable, as the OpenAPI specification includes aspects that are only intended to add context to the API consumers in the form of free text, and will never easily translate into a specific application behaviour.

# The consumer

Having a specification is great for documentation, but our goal in this article is not to document but to introduce mechanisms for ensuring service contracts are respected.

One simple way to have your contracts respected is to generate strongly-typed clients from an OpenAPI specification. Having such a client would allow you to check your contracts at compile time!

The OpenAPI ecosystem provides one tool to make that job very simple: the [OpenAPI generator](https://openapi-generator.tech/). The OpenAPI generator is a fantastic code generating tool that works over multiple languages. It can both generate mock servers and clients. Here we are interested in that latter use case.

Let's say your consumer service is coded in TypeScript. You can generate a strongly-typed client very easily:

```
npx @openapitools/openapi-generator-cli generate -i $PATH_TO_OPEN_API_SPEC \
    -g typescript-axios -o $OUTPUT_CLIENT_PATH \
    --additional-properties=useSingleRequestParameter=true
```

That will generate code for an `npm` package that you can either publish or just integrate directly into your consumer service. Then, in your consumer, instead of calling your APIs directly with some HTTP library like axios, you will use your strongly typed client:

```typescript
import { Api, Configuration } from './generated-clients/my-service'

async function callMyApi() {
  const apiConfig = new Configuration({ basePath: process.env.API_URL });
  const apiClient = new Api(apiConfig);

  apiClient.createBlog({ createBlogParameters: { title: 'How my dog ate my homework', content: 'No content, it was eaten as well' }})
}
```

Here the `createBlog` method will be typed in Typescript, which will verify at compile time that you are respecting the contract of your service. Isn't that exciting?!!!

<img src="/assets/images/dog_fake_smile.jpg">

Well maybe it won't bump your heart rate, but it definitely reduces the chance of contract-related issues on your deploys :)

# Putting it all together

Let's review how this works:
* You build your API using NestJS, which will help you to build an OpenAPI specification that matches your implementation with very little overhead or maintainability issues.
* Once you have an OpenAPI specification, you can generate a client on the language of your choice with `openapi-generator`.
* Working on any strongly typed language on your consumer application, like Typescript, will ensure that you benefit from compile time checks on your service calls.
* In order to ensure all this infrastructure is easily maintained, the CI/CD pipeline of your service will need to publish a new version of the client each time the API changes.
