---
title: Why use Web API specifications?
description: "Let's talk about why you should use Web API specifications even in small projects"
date: 2017-06-03 19:30:00
tags: [swagger, openapi, web development, web api, backend, frontend]
---

You can see Web API Specifications like Javadoc: you can't use a Java library without Javadoc! Web API spec are "Javadocs" of web services. But there's one big difference: you can write your Web API Spec before start writing your server. It generates a big difference inside development process. I will introduce you to this magic world.

## Design Driven Development
Design Driven Development is the name of technique which consists in: write the Web API spec before start writing any other code. Then, when you start writing code, you already have implemented a "dictionary" of server-client interface, so front-end and back-end developers go straight to implement logic of application. If you choose the right tools, you don't have to care about:
                                                                                                                                                                                                                                                                                                                                 
* What is the correct HTTP Request to do something
* If you have done request with correct and valid parameters
* If logged user has correct requirements to complete the operation requested

The flexibility of Web API specifications enables this to be used for a lot of use cases:

* Develop a SPA (single page application) with back-end in any language and front-end in Javascript
* Develop a mobile application with back-end in any language and front-end natives
* Develop a web service with backend in every language and auto-generate clients for every language


## Approaches to use Web API Specification in you project
Both client and server can be linked to Web API Spec through two different approches:

* Static code generation
* Dynamic code generation

Static code generation is the most simple to achieve, you can find a code generation library for every language/framework/api specification standard combination. Depending on your project and on library you use it can be useful approach or not. It's a really interesting approach on client-side, but it lacks of flexibility in server side. Most code generators on server side create a **stub of server code**, with all validation and security routines generated directly from code generator. It can be really useful when you write an Web API Spec that you assert that will __wont't change__ during development of back-end. One important feature of static code generation is represented by performances, so I prefer this for a web service project

Dynamic code generation is the most interesting for server side of small project. You can change your spec every time you want, and server will generate new validation flow. It's interesting when you write a client-server complete stack, and you change spec during development.

## Small projects
Do you want to write a magic mobile/web application with a couple of your friends that do something astonishing? Use this tools:

* OpenAPI specification 2 (version 3 is newer, but you can't find tools related)
* Node.JS backend with dynamic code generation library (I suggest [swaggerize-express](https://github.com/krakenjs/swaggerize-express/))
* Static client code generators
* Good feeling between you and your friends when you write the spec :)
