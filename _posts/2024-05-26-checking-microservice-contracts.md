---
layout: single
toc: true
toc_sticky: true
title:  "Seamlessly verifying API contracts at compile time with NestJS and Typescript"
date:   2023-05-26 22:00:00 -0300
---

One of the challenges that microservices architectures present is that, since code is distributed, it becomes harder to verify all the pieces work together properly.

While integration tests are a key instrument to address this challenge, integration tests are usually harder to write and more expensive to run than the average automated test in simpler architectures (e.g. a monolithic application). These challenges grow whenever you add more services and when your services are managed by multiple teams. Therefore, while it's definitely wise to plan integration tests for your use cases, you will most likely be limited in the number and types of tests you will be able to add.

The second tool that will help you verify proper integration is contract verification. There are many ways to check contracts between multiple services, but the underlying idea is simple: we want to verify that whenever we have an interaction between two services, the contracts (i.e. the methods and payload format) are respected. In this blog post, we are presenting a scalable approach to contract testing at compile time, which relies on TypeScript, NestJS, and OpenAPI.

# What is OpenAPI?

OpenAPI is an open standard for defining HTTP service contracts. It's the de-facto standard for defining JSON apis, specially in the Javascript ecosystem. While a bit more limited, it also has support of XML, and there's tooling for other languages.

# Writing and maintaining API specifications

There are two opposing paradigms when it comes to writing and maintaining your open api specs: design-first and code-first.

Design-first paradigm, basically mean that you would work on your specification before coding your actual service. Code-first on the other hand, as the name suggests, is based on the idea that the development should focus on the actual code itself first and then on the specification. Here we will be explore the code-first paradigm. We won't be expanding on the trade-offs involved on choosing each ones of these alternatives, but if you are interested in knowing more there's an excellent article [here](https://apisyouwonthate.com/blog/api-design-first-vs-code-first/).

Regardless of what path you choose, you will need to somehow ensure that your API specification correctly defines the behavior of your application and that whenever changes happen you can easily keep them synchronized.

There is no single way to get code-first done. That said, it is certainly quite common that development teams rely on tools that build the specification from metadata that lives in the code itself. This metadata sometimes exists in the form of special comments or annotations. For instance, some libraries, like swagger-jsdoc, will translate your jsdoc comments into an open api specification:

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

The main drawback of this approach is that there's no way to guarantee that these special artifacts in the code correctly reflect the actual behaviour of the service. To overcome these problems some other tools have arisen, like RSwag, that help you build your specification in the form of special unit tests, what ensures that every endpoint that is part of your specification is backed up by a special unit test:

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

While "the RSwag way" certainly improves the maintainability of your solution, it also adds the need for you to write these tests, which creates overhead to the development process. It also means that you not only need to learn OpenAPI, but also now you will also have to be versed on RSwag DSL as well.

# Code first done right

The holy grail of code-first would be a system that ties the actual behaviour of your service to its specification without the need of having special artifacts that add complexity and overhead.

NestJS is an example of such a framework. A controller in NestJS looks like this:

```typescript
import { Controller, Post } from '@nestjs/common';

class CreateBlogDto {
  name: string;
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

As you can see, it heavily relies on Typescript, both for annotations and typing. The `@Controller('blogs')` annotation tells NestJS that the controller will be handling responses to the `blogs` path. Then we also have the `@Post` annotation which states that the `create` method will be used to handle the response to `POST` http calls in the controller's path. Finally the parameters are annotated with the `@Body` annotation, which tells us that the service will expect these parameters serialized in the body.

Note that all these annotations, unlike what happens with our previous jsdoc example **actually define the service behaviour**, and are not separate artifacts that will need to be maintained on top of our implementation.

You may be correctly guessing that all these artifacts, while originally intended to define *behaviour* can also be repurposed to output an *specification*. That's where the `@nestjs/swagger` package comes into place. By simply adding a few lines of code to your application bootstrapping, you can ensure an openAPI specification is published in one of the application paths:

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

TaDa! Now when your app starts, it will also write an openAPI specification to a file. Pretty cool, isn't it?

Well, here is the thing: I lied to you. While that setup will indeed output an specification, you may probably want to add some special artifacts whose only purpose is to add more details to your specification and have no behavioural impact whatsoever. For example you will explicitly need to add the `@ApiProperty` annotation on top of each one of your Dto attributes that you want to include in the documentation:

```
class CreateBlogDto {
  @ApiProperty
  name: string;

  @ApiProperty
  content: string;
}
```

You can also add further attributes to your annotations which will further enrich your specification:

```
class CreateBlogDto {
  @ApiProperty({
    description: 'The name of your blog',
    example: 'Yet another tech blog'
  })
  name: string;

  @ApiProperty({
    description: 'The contents of your blog',
  })
  content: string;
}
```

