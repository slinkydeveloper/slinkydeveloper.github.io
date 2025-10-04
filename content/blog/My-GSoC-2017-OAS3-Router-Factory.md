---
title: My GSoC 2017 - OpenAPI 3 Vert.x support
description: "Good library it's all about good interface"
date: 2017-08-12
tags: [web development, gsoc 2017, gsoc, vertx, vertx web, openapi, openapi3, router factory, factory]
---

The support to OpenAPI 3 it's located in maven package `vertx-web-api-contract-openapi`, and most classes extends/subclass from interfaces/classes inside maven package `vertx-web-api-contract-common` (the package designed to contain all API Specs standards common classes). Most important interfaces of `vertx-web-api-contract-openapi` are:

* The `OpenAPI3ValidationHandler` class that fills [`BaseValidationHandler` maps](https://slinkydeveloper.github.io/articles/My-GSoC-2017-Validation/#structure-of-validation-framework)
* The `OpenAPI3RouterFactory`, the interface that enable users create a router with your API spec

As I said in a [previous blog post](https://slinkydeveloper.github.io/articles/Whats-New-In-OAS3-Parameters/), OpenAPI 3 added a lot of new things, in particular about serialization styles and complex form bodies (url encoded and multipart). So when I started working on OpenAPI 3 requests validations, I had to add a lot of things to validation framework that I haven't expected before.

## The validation handler
`OpenAPI3ValidationHandler` is an interface extension of `HTTPOperationRequestValidationHandler` (located inside `vertx-web-api-contract-common`), that is an interface extension of `ValidationHandler`. This class contains all methods to elaborate the `Operation` object (Java representation of [OAS3 operation object](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#operationObject)) and the list of `Parameter` objects (Java representation of [OAS3 parameter object](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#parameterObject)).

When constructed, it generates all `ParameterValidationRule` and `ParameterTypeValidator` it needs: in fact, **It doesn't elaborate the api spec nor work with api spec Java models during the validation**. It does everything when It's constructed, so It iterates through various parameters and It generates objects needed for validation.

<p class="image-pull-right">
<img src="{{ site.url }}/images/messy_code_fry.jpg" alt="">
</p>
If you read this class, It seems messy, because It's messy :smile:. This is because of complexity of OAS 3, that forced me to write some little tricks to support all things (like for example to manage the `deepObject` serialization style).

To give a quick explanation of how this class elaborates parameters:

1. It checks if it's supported (parameters with `allowReserved: true` are not supported)
2. It checks if the parameter needs a _workaround_ to get validation working and applies the specific _workaround_
3. If none workaround is needed, it constructs the correct type validator

Behind the scenes all the validation work is done by [validation framework](https://slinkydeveloper.github.io/articles/My-GSoC-2017-Validation/)

## The router factory
The router factory is intended to give the most simple user interface to generate a router based on an API Spec. In fact, it provides this functionalities:

* Async loading of specification and its schema dependencies
* OpenAPI 3 compliant API specification validation (thanks to [Kaizen-OpenApi-Parser](https://github.com/RepreZen/KaiZen-OpenApi-Parser))
* Load handlers and failure handlers with operationId
* Automatic 501 (Not implemented) response for operations with missing handlers (can be enabled/disabled with `mountOperationsWithoutHandlers(boolean)`)
* Automatic `ValidationException` failure handler (can be enabled/disabled with `enableValidationFailureHandler()` and manually configured with `setValidationFailureHandler()`)
* Path's regular expression generation (to support `matrix` and `label` style unsupported natively from Vert.x)
* Lazy methods: the generation of the `Router` is done only when you call `getRouter()`
* Automatic mount of security validation handlers

### Lazy methods
It's usual to run into problems regards route declaration order. For example if you declare two routes in this order:

1. `GET /hello/{parameter}`
2. `GET /hello/world`

With actual Vert.x `Router` implementation, `/hello/world` handler will never called, unless you explicitly call `RoutingContext#next()` inside `/hello/{parameter}` handler (that causes `Router` to run the next route matching the pattern). With lazy methods It's guaranteed that routes will be loaded with order declared inside API specification.

I choose lazy methods also for code style reasons, It helps a lot to manage the code of router factory.

## And the final result
With this tools, user can bring OpenAPI 3 power to its Vert.x server implementation as simple as:

```java
OpenAPI3RouterFactory.createRouterFactoryFromFile(this.vertx, "src/main/resources/petstore.yaml", ar -> {
            if (ar.succeeded()) {
                // Spec loaded with success
                OpenAPI3RouterFactory routerFactory = ar.result();

                // Add some handlers
                routerFactory.addHandlerByOperationId("listPets", routingContext -> {
                    RequestParameters params = routingContext.get("parsedParameters");
                    // Handle listPets operation
                });
                routerFactory.addFailureHandlerByOperationId("listPets", routingContext -> {
                    Throwable failure = routingContext.failure();
                    // Something really bad happened during listPets handling
                });

                // Add a security handler
                routerFactory.addSecurityHandler("api_key", routingContext -> {
                    // Handle security here and then call next()
                    routingContext.next();
                });

                // Now you have to generate the router
                Router router = routerFactory.getRouter();

                // Now you can use your Router instance
                HttpServer server = vertx.createHttpServer(new HttpServerOptions().setPort(8080).setHost("localhost"));
                server.requestHandler(router::accept).listen();

            } else {
                // Something went wrong during router factory initialization
                Throwable exception = ar.cause();
            }
        });
```

Next time I'm going to introduce you `slush-vertx`, a new generator for Vert.x project. Stay tuned!
