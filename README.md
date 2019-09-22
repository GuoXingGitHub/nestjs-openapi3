# `@eropple/nestjs-openapi3` #
**Please note that this library is in an _alpha_ state. The API should be more or less stable and the failure cases of the library are pretty minimum, but no promises.**

`@eropple/nestjs-openapi3` is a library for [NestJS]() to generate [OpenAPI 3.x]() documents from your API specification. It attempts to be more integrated with the flow of your application than [@nestjs/swagger]() and to push you towards building clean, well-separated APIs along the way.

## Release History ##
### `0.1.0` ###
- Initial release.

## Installation ##
Pop into your NestJS project and `npm install @eropple/nestjs-openapi3` or `yarn add @eropple/nestjs-openapi3`.

You'll need reasonably up-to-date versions of `@nestjs/common`, `@nestjs/core`, `reflect-metadata`, and `rxjs`. **NestJS before version 6.5 is unsupported and shall not be supported.**

## Usage ##
First and foremost, you should be aware that `@eropple/nestjs-openapi3` is a heavily integrated library. This is going to touch a lot of your codebase. If you're new to OpenAPI, you should investigate [the OpenAPI 3.x spec](). If you've used Swagger, whether through `@nestjs/swagger` or another avenue, you should read up on [what's new in OpenAPI 3.x]().

### Attaching `OpenapiModule` ###
You'll need to attach the module to your application. We have to do it this way, rather than as a normal module dependency on `AppModule` or similar, because we need to get access to the completely populated dependency injection container.

```ts
// It's good practice to make this a separate method; you can then
// do handy stuff like generate your OpenAPI document without having
// to start a web server. Take a look at our
async function buildOpenapiDocument(app: INestApplication) {
  return OpenapiModule.createDocument(
    {
      app,
      // You can pass `console` here if you're lame, but you
      // should be using Bunyan. Check out `@eropple/nestjs-bunyan`!
      baseLogger: openapiLogger,
    },
    (b) => {
      b.addTitle(`${APPLICATION_NAME} API`);
      b.addVersion(PACKAGE_VERSION);
      b.addServer({
        url: API_URL,
      });
      b.addSecurityScheme('token', {
        name: 'Authorization',
        type: 'apiKey',
        in: 'header',
      });
    },
  );
}

async function configureApp(app: INestApplication) {
  // Wherever you set up CORS, helmet, etc. - just do this

  await OpenapiModule.attach(
    {
      app,
      baseLogger: openapiLogger,
      document: await buildOpenapiDocument(app),
    },
  );
}
```

### Decorators, Decorators, and More Decorators ###
This package provides a number of decorators and it's probably best to look at [Ed's skeleton project]() to get a works-in-anger example of the full API.

One thing I've found to be very useful is to import decorators under a namespace in order to make it clearer when decorators that _aren't_ from NestJS are being used. My standard approach, then, is to import the entire package:

```ts
import * as OAS from '@eropple/nestjs-openapi3';
```

#### Parent- and Endpoint-Level Decorators ####
Some decorators can be applied to controllers (and all endpoints within that controller) and modules (and all endpoints within all controllers). These include:

- `@OAS.Tags()` - tags operations in the document (often used to more richly define clients)
- `@OAS.Deprecated()` - marks as deprecated
- `@OAS.Ignore()` - skips OpenAPI handling
- `@OAS.Parameter()` - defines a parameter that's used outside of the handler. For example, in [Ed's skeleton project](), it's used to provide `tenantName` in a way that can be passed to `@eropple/nestjs-auth` to determine tenancy, from which a user can be found.
- `@OAS.SecurityScheme()` - adds a security scheme, with scopes for OAuth2 or OIDC schemes

#### Endpoint-Level Decorators ####
In addition to the above, some decorators apply explicitly to the endpoint handler.

- `@OAS.Get()`, `@OAS.Post()`, `@OAS.Put()`, etc. - replaces NestJS's http method decorators (such as `@Get()`) with decorators that accept OpenAPI operator details.
- `@OAS.Operation()` - the "break glass in case of" option. If you want to write a literal operation for inclusion in your OpenAPI document, use this one. You'll need to use a NestJS HTTP method decorator, such as `@Get()`, to make sure that NestJS can route requests.

#### Argument-Level Decorators ####
These decorators only apply to the arguments in an endpoint hander function.

- `@OAS.Body()` - wraps NestJS's `@Body()` and accepts schemas for request bodies.
- `@OAS.Path()` - wraps NestJS's `@Param()`. The name change is _intentional_ to match up with the `in` field of an OpenAPI parameter object.
- `@OAS.Header()` - wraps NestJS's `@Headers()`, only supporting a single selected header.
- `@OAS.Query()` - wraps NestJS's `@Query()`.

### Fetching the OpenAPI Document ###
The OpenAPI specification strongly suggests that the OpenAPI JSON document should be served at `/openapi.json`. We follow that suggestion.

### Validation and Coercion ###
One important difference between `@nestjs/swagger` and `@eropple/nestjs-openapi3` is that this library does type conversion to match your desired schema. By default, in NestJS, the following HTTP handler doesn't do what you'd expect:

```ts
@Controller('foo')
export class FooController {
  @Get()
  doSomething(@Query('v') v: number) {
    console.log(typeof(v)); // This will return `string`, not `number`!
  }
}
```

This stinks, so `@eropple/nestjs-openapi3` tries to do one better. Since OpenAPI schemas are just JSON schemas, we use [ajv]() to validate incoming parameters and request bodies. ajv also supports type coercion, which we've enabled. So instead, you've got something more like this:

```ts
@Controller('foo')
export class FooController {
  @OAS.Get()
  doSomething(@OAS.Query('v') v: number) {
    console.log(typeof(v)); // This will return `number`.
  }
}
```

ajv's type coercion also applies to objects in request bodies.

There are some sharp edges to ajv type coercion. Most notably, if a client passes `null` when a string is expected, it will be coerced to the empty string (`""`). You might want to take a look at the [ajv coercion chart]() to avoid any footguns.

### Upgrading from `@nestjs/swagger` ###
Discussing this in-depth is a big ol' **TODO** item. I've converted projects using Swagger 2.0 to using this library; if you have a basic understanding of a Swagger document and its components, porting is pretty easy. Some notes (please feel free to add more to this list):

- NestJS's built-in routing decorators, like `@Get()` and `@Post()`, can be replaced with `@OAS.Get()` and `@OAS.Post()`. You can specify operation information as an argument to this decorator.
- `@Param()` is replaced with `@OAS.Path()`
- `@Headers()` is still usable to fetch all headers from a request, but `@Headers('specific-header')` is replaced with `@OAS.Header()`.
- `@Query()` is replaced with `@OAS.Query()`.
- `@Body()` is replaced with `@OAS.Body()`.
- Burn all that `@ApiExplicitParam()` stuff with cleansing fire.
- Add `@OAS.Model()` or `@OAS.ModelRaw()` to all your request bodies or responses. (`@OAS.ModelRaw()` is useful for turning off model introspection if you've got an object that hinges on discriminators or other advanced)
- Replace `@ApiModelProperty()` with `@OAS.Prop()` or, if you're writing something that's hard to describe, `@OAS.PropRaw()`.

## What This Library Doesn't (And Probably Won't) Do ##
- [Discriminators]() are not supported and probably never will. I think they're a bad idea and nearly impossible to model in a generic way. If you've got a brilliant PR suggestion, I'd be willing to look at it, but seeing as how it makes a lot of things break (for example, we no longer can use [ajv]() to validate schemas), I'm skeptical.

## Avenues for Contribution ##
Look for `TODO`s in the codebase; they're usually good contribution opportunities!

- Add optional Swagger UI functionality, similar to `@nestjs/swagger`.
- Tests in isolation. I'm not clear on how to adequately unit test against functionality hanging directly off of NestJS; right now this library (along with other highly integrated libraries of mine, like nestjs-auth) relies on integration tests in example projects.
- Only the simplest parameter styles are currently supported: `simple` (the only one for headers, though paths have other options that we don't support) and `form` for query parameters. If somebody out there needs more complex styles, you should
- Cleaner types around some of the API. For example, right now you must use one of `content` or `multiContent` for request bodies, but there's no type checking to assert that one _must_ be used. Similarly, operation responses are a little kludgy. Probably an easy fix for somebody!
- Deeply nested schemas are probably not being handled correctly (allowing the use of a `SchemaLike`--a JSON schema or a class constructor decorated with `@OAS.Model()`) in all cases.
- [Example functionality]() is entirely absent. Squaring it with simplified flavors of content objects is hard and I don't use it; pull requests welcome.
- There are a couple of odd behaviors resulting from using [ajv]() to validate client parameters/request bodies against our OpenAPI 3 schemas. In particular, when dealing with a string field, ajv coerces `null` to the empty string. I think this should fail hard instead, but

[NestJS]: https://nestjs.com/
[Ed's skeleton project]: https://github.com/eropple/nest-and-next-skeleton
[OpenAPI 3.x]: https://swagger.io/docs/specification/about/
[@nestjs/swagger]: https://docs.nestjs.com/recipes/swagger
[the OpenAPI 3.x spec]: https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md
[what's new in OpenAPI 3.x]: https://swagger.io/blog/news/whats-new-in-openapi-3-0/
[Example functionality]: https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#exampleObject
[Discriminators]: https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#discriminatorObject
[ajv]: https://github.com/epoberezkin/ajv
[ajv coercion chart]: https://ajv.js.org/coercion.html
