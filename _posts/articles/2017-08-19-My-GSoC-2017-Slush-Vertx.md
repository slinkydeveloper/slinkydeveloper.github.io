---
layout: post
title: My GSoC 2017 - Slush-vertx
excerpt: "Scaffolder for Slush-vertx project"
clippy: I help devs too!
modified: 2017-08-19T10:33:41+02:00
categories: articles
tags: [web development, gsoc 2017, gsoc, vertx, vertx web, openapi, openapi3, generator, slush-vertx]
share: true
ads: true
---

[`slush-vertx`](https://github.com/pmlopes/slush-vertx) is a project created by [Paulo Lopes](https://www.jetdrone.xyz) born to simplify build tools configurations for Vert.x. I totally refactored `slush-vertx` to create a multi purpose code generator for simplify various configurations of Vert.x powered projects.

## Multi purpose and simple to enlarge
When I designed the new slush-vertx, I tried to create a Vert.x project generator for every configuration needed, not only an OpenAPI 3 server or OpenAPI 3 client. Another important variable of my project is create a generator that generates code for different languages and different build tools.

Now `slush-vertx` It's like a _"code generation hub"_: It contains a set of project generators, based on what type of Vert.x project are you going to scaffold. At the moment I'm writing this post, slush-vertx contains:

* Vert.x Starter project generator: Based on original Paulo's project, generates an empty project configured for Vert.x 3 Framework
* Vert.x Web Server Starter generator: Generates a skeleton with sources and tests for Vert.x 3 Web powered REST server
* Vert.x Web Server OpenAPI project generator: Generates a skeleton based on Swagger 2/OpenAPI 3 specification with sources and tests for Vert.x 3 Web powered REST server
* Vert.x Web Client OpenAPI project generator: Generates a client based on a Swagger 2/OpenAPI 3 specification and Vert.x 3 Web Client

I hope it will grow in  the future, creating a tool that can help people to connect with Vert.x world.

## How it works
<script type="text/javascript" src="https://asciinema.org/a/134050.js" id="asciicast-134050" async></script>

## Behind the scenes
slush-vertx is metadata and template driven code generator. This means:

* For every language/build tool it has a set of metadata, that can be extended with user inputs
* It uses a template engine to generate code, based on metadata

If you want a complete scenario of behaviours of slush-vertx, give a look at [this wiki page](https://github.com/pmlopes/slush-vertx/wiki/Slush-Vert.x-Structure)

But, why that complexity behind a code generator? I mean, It's only a code generator! Yes, It's only a code generator, but I wanted to create a tool simple to extend with new generators routines, giving to Eclipse Vert.x a powerful tool.

## Generate unit tests
So, if you have a powerful build tool that generates pretty everything you want, why don't take advantage of it doing things that you don't want to do? And this is what I've done! Copy-pasting code from other generators I've created, I builded [a unit test generator for `vertx-web-api-contract-openapi`](https://github.com/pmlopes/slush-vertx/tree/master/src/generators/vertx_web_unit_test_generator). This generator takes all operations declared in [this oas 3 spec](https://github.com/pmlopes/slush-vertx/blob/master/src/generators/vertx_web_unit_test_generator/openapi.yaml) and generates a specific test to validate the correct parsing of parameters on server side. This is the final result: [`OpenAPI3ParametersUnitTest.java`](https://github.com/slinkydeveloper/vertx-web/blob/designdriven/vertx-web-api-contract/vertx-web-api-contract-openapi/src/test/java/io/vertx/ext/web/designdriven/openapi3/OpenAPI3ParametersUnitTest.java). This unit tests helped me a lot to complete the `vertx-web-api-contract-openapi` module.

With some small changes this can be a complete server libraries/frameworks compatibility test tool for OpenAPI 3

## And now?
Now use it! Follow the [readme inside GitHub repository](https://github.com/pmlopes/slush-vertx) to install and start using it.

You can also [contribute](https://github.com/pmlopes/slush-vertx/wiki/How-to-contribute) to this project [adding new generators](https://github.com/pmlopes/slush-vertx/wiki/Create-a-new-generator) and [updating existing ones with new languages](https://github.com/pmlopes/slush-vertx/wiki/Add-new-language-to-existing-generatorhttps://github.com/pmlopes/slush-vertx/wiki/Add-new-language-to-existing-generator)
