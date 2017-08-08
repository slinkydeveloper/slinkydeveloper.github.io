---
layout: post
title: What's new in OpenAPI Specification 3 - Parameters
excerpt: "A guide to new Parameter Object inside OpenAPI Specification"
modified: 2017-06-23T23:36:44+02:00
categories: articles
tags: [swagger, openapi, openapi3, web development, web api]
share: true
ads: false
---

OpenAPI 3 Parameter Object it's totally different from old OpenAPI 2. It gives the power to describe complex parameters, using the power of Schema object. I will introduce to you this breaking changes

## No more form and body parameters
One of the major changes is that body parameters (forms, json, ...) are moved to a new object called `RequestBody`. So now `Parameter` supports only request parameters `in`:

* `header`
* `query`
* `path`
* and the new one `cookie`

## `schema` is better than `type`
In OpenAPI 2 a parameter is defined as:
```yaml
parameters:
  - name: smile
    in: query
    type: string
    format: ([;:]-*([()\[\]])\2*) # Smile regex
```
In OpenAPI 3 you can find the same parameter as:
```yaml
parameters:
  - name: smile
    in: query
    schema:
      type: string
      format: ([;:]-*([()\[\]])\2*) # Smile regex
```
The major difference is that now you **have to** define a `schema` for every single parameter, even the most simple.

It seems annoying, but It gives some interesting opportunities. For example: A model identifier has a particular regular expression (`format`) that describes it and you want to write CRUD methods for this model. This is what you have to do in OpenAPI 2:
```yaml
# get path
parameters:
  - name: id
    in: query
    schema:
      type: string
      format: ^[a-z]{3}[0-9]{10}$
# delete path
parameters:
  - name: id
    in: query
    schema:
      type: string
      format: ^[a-z]{3}[0-9]{10}$
# And so on
```
Now with OpenAPI 3 you can define this single string as a schema and reference to it where you want:
```yaml
# define your identifier schema in components/schema
components:
  schemas:
    my_model_identifier:
      type: string
      format: ^[a-z]{3}[0-9]{10}$

# get path
parameters:
  - name: id
    in: query
    schema:
      $ref: '#/components/schemas/my_model_identifier'
# delete path
parameters:
  - name: id
    in: query
    schema:
      $ref: '#/components/schemas/my_model_identifier'
# And so on
```
You can also reuse it to define complete model:
```yaml
components:
  schemas:
    my_model_identifier:
      type: string
      format: ^[a-z]{3}[0-9]{10}$
    my_model:
      type: object
      parameters:
        id:
          $ref: '#/components/schemas/my_model_identifier'
        other_field:
          type: string
```

## Objects inside path/query/header parameters
This is tricky, but actually it's possible. With Schema object support you can define object as query, header, cookie, path parameter. This is an example:
```yaml
parameters:
  - name: parameter
    in: query
    schema:
      type: object
      properties:
        a:
          type: integer
        b:
          type: string
      required:
        - a
        - b
```

I will explain later how to submit this type of requests

One interesting usage is when you have multi-dimensional key for a model, for example a geolocation model.

## Serialization style changes
`style` is the new name of `collectionFormat` field. But It isn't only a name change. Also `style` is supported by another field: `exploded`. This is the comparison table with OpenAPI 2

|`style` | `explode` | OpenAPI 2 `collectionFormat` |
|:-----------|:------:|----------:|
| matrix | false | not supported |
| matrix | true | not supported |
| label | false | not supported |
| label | true | not supported |
| form | false | `csv` |
| form | true | `multi` |
| simple | false | `csv` |
| simple | true | `csv` (not for object) |
| spaceDelimited | false | `ssv` |
| pipeDelimited | false | `pipes` |
| deepObject | true | not supported |
{: .table}

For more informations about how to use this two new fields, check out this [table](https://github.com/OAI/OpenAPI-Specification/blob/OpenAPI.next/versions/3.0.md#style-examples).

## Complex parameters with `content`
If you think `schema` isn't enough, check out `content` field. I will explain this further when I will cover `RequestBody`

## Tips to create a good parameter object in OpenAPI 3
* Try to keep parameters as simple as possible. Avoid as much as possible objects and arrays, They create only a lot of headaches
* Use `query` parameters if you need to pass arrays to operation and use `form` style
* Pass objects to an operation with `RequestBody`, or split it in different primitive parameters.
* Take as much as possible advantage of `schema` inside `Parameter` object. In particular use it to define object identifiers
