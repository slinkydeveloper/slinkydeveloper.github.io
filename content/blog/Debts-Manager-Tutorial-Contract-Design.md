---
title: "Debts Manager Tutorial Part 2: Contract Design"
description: "In this second chapter of Debts Manager Tutorial I would like to show you how I have designed the REST API of Debts Manager. I'm going to follow the API First approach, documenting all aspects of the API Design with OpenAPI 3."
date: 2019-01-20
tags: [development, web, web api contract, vertx, openapi, openapi 3]
---

Hi guys! Welcome back to this tutorial!

In this second chapter of Debts Manager Tutorial I would like to show you how I have designed the REST API of Debts Manager. I'm going to follow the _API First_ approach, documenting all aspects of the API Design with OpenAPI 3.

This post doesn't aim to provide you a full guide of how to design REST APIs: if you want more resources to learn it, [look at the end of this post](#Some-resources-to-learn-Web-API-Design-and-OpenAPI)

## Analysis

The REST APIs, in contrast with RPC, are driven by the data the services wants to expose. In the previous chapter I gave you an idea of the entities we must expose. Now I tabulate these and the relative operations on it.

| Entity            | Create   | Retrieve | Update   | Delete   |
|-------------------|----------|----------|----------|----------|
| User              |:heavy_check_mark:|:heavy_check_mark:|:x:|:x:|
| User relationship |:heavy_check_mark:|:heavy_check_mark:|:x:|:x:|
| Transaction       |:heavy_check_mark:|:heavy_check_mark:|:heavy_check_mark:|:heavy_check_mark:|
| Status            |:x:|:heavy_check_mark:|:x:|:x:|


This table is a pretty good starting point, but I must refine the analysis enforcing our methods with policies and logics.

These policies are primarly based on _who is making the request_. I'm going to define a login phase together with JWT to provide authorization and authentication. Each endpoint, except `login` and `register`, is secured with a JWT auth. My objective is expose, for each user, only a subset of data relative to the user itself.

## Models

Before defining the endpoints I must formally describe the data models representing the service entities. OpenAPI has its own Json Schema dialect to define models: [OpenAPI Schema](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md#schemaObject). This is an extended subset of [Json Schema Draft 5](http://json-schema.org/specification-links.html#draft-5). Meanwhile I'm writing, there is a proposal to allow usage of every version of Json Schema, including the newer versions, with an extension [https://github.com/OAI/OpenAPI-Specification/issues/1532](https://github.com/OAI/OpenAPI-Specification/issues/1532).

I place these schemas in main OpenAPI file under `components` and `schemas` keywords. I can refeer to it using Json schema references (`$ref` keyword).

The simplest model here is the user. I want to expose only the username, so I represent it with a simple `string`. This is the definition using OpenAPI Schema:

```yaml
Username:
  minLength: 5
  type: string
```

The status is represented by a map with users as keys and total debts\credits as values. In OpenAPI Schema:

```yaml
Status:
  description: 'Map with username as keys and debt as value'
  type: object
  additionalProperties:
    type: number
```

In JSON maps are usually represented or as json array of tuples key-value or as a json object. The json object is the natural way to represent it, but it has an important restriction: keys are strings. In my case I need to represent a map string → number, so json object representation fits good. The map values schema are defined using `additionalProperties` and, only with Json Schema Draft 7 or newer, keys schema are defined using `propertyNames`.

The main transaction model is described below:

```yaml
Transaction:
  type: object
  properties:
    id:
      type: string
    from:
      $ref: '#/components/schemas/Username'
    to:
      $ref: '#/components/schemas/Username'
    at:
      description: "Insertion datetime"
      format: date-time
      type: string
    value:
      type: number
    description:
      minLength: 1
      type: string
  required:
    - from
    - to
    - id
    - description
    - value
    - at
```

`$ref` keyword points to the `Username` schema I defined before.

This model doesn't fit good for my usage, because for each `Transaction` endpoint I want to apply some policies. A very common example is the `id` field: when user inserts a new transaction I want to designate the database to fill the `id` value. When the user creates a new transaction it shouldn't add the `id` field: that means that I can't use the `Transaction` model to describe the "create transaction" request body. Let's look at all restrictions I want to apply on various transaction endpoints:

* `id` and `at` are filled by the backend when user adds a new transaction and they are immutable from the API perspective
* When user updates a transaction he can't update the `from` (sender) and `to` (receiver) fields
* When user adds a new transaction he doesn't need to fill the `from` field because the backend fills it with the logged user

To apply these restrictions I create a new model for each endpoint. I'm going to refactor `Transaction` into 3 different models: `UpdateTransaction`, `NewTransaction` and `Transaction`.

These new models lead to a new problem: duplication of model fields definitions. Json schema solves the duplication with schema composition keywords: `allOf`, `anyOf` and `oneOf`. In particular I will use `allOf` to achieve inheritance of schemas.

This is the final result:

```yaml
UpdateTransaction:
  type: object
  properties:
    value:
      type: number
    description:
      minLength: 1
      type: string
NewTransaction:
  allOf:
    - $ref: '#/components/schemas/UpdateTransaction'
    - properties:
        to:
          $ref: '#/components/schemas/Username'
      required:
        - to
        - description
        - value
      type: object
Transaction:
  allOf:
    - $ref: '#/components/schemas/NewTransaction'
    - required:
        - from
        - to
        - id
        - description
        - value
        - at
      type: object
      properties:
        from:
          $ref: '#/components/schemas/Username'
        id:
          type: string
        at:
          description: "Insertion datetime"
          format: date-time
          type: string
```

The schemas inheritance tree is `UpdateTransaction`←`NewTransaction`←`Transaction`

## Endpoints

OpenAPI document structures the endpoint definitions as follow:

```yaml
paths:
  /pathA: {}
  /pathB:
    get: {}
    post: {}
    put: {}
  /pathC/{paramA}: {}
```

OpenAPI path strings allow path parameters using `{paramName}` and doesn't require an explicit definition of query parameters.

In OpenAPI terminology an **operation** is an API endpoint identified by a path and an HTTP method. Every operation could be uniquely identified with an `operationId`. The OpenAPI Specification (OAS) documents this field as optional, but I strongly suggest to specify it if you don't want to see your tooling explode. Most code generation tooling asserts that `operationId` is present. If it's not present they try to infeer it from path and http method producing unexpected results.

For each operation we are going to define:

* `operationId`
* `parameters` (if any): List of `header`, `path`, `query` and `cookie` parameters
* `requestBody` (if any): Content type and content schema of request bodies
* `responses`: Status code with response content type and schemas

I also fill the `security` field for each operation to require a JWT token to execute it.

### Transactions and Status

Let's start with transaction CRUDs:

| Operation           | `operationId`     | CRUD     | Path               | HTTP Method |
|---------------------|-------------------|----------|--------------------|-------------|
| Create a new transaction |`createTransaction`|**C**reate|`/transactions`|POST|
| Get a single transaction |`getTransaction`|**R**etrieve|`/transactions/{transactionId}`|GET|
| Get user related transactions |`getTransactions`|**R**etrieve multiple|`/transactions`|GET|
| Update a transaction |`updateTransaction`|**U**pdate|`/transactions/{transactionId}`|PUT|
| Delete a transaction |`deleteTransaction`|**D**elete|`/transactions/{transactionId}`|DELETE|

In OpenAPI:

```yaml
  /transactions:
    get:
      operationId: getTransactions
      responses:
        '200':
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Transaction'
        '401':
          description: 'Expired token'
      security:
        - loggedUserToken: []
    post:
      operationId: createTransaction
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/NewTransaction'
        required: true
      responses:
        '201':
          description: 'Successful response.'
        '401':
          description: 'Expired Token'
        '403':
          description: "Trying to create a transaction with receiver not connected to logged user"
      security:
        - loggedUserToken: []
  '/transactions/{transactionId}':
    get:
      operationId: getTransaction
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Transaction'
        '401':
          description: 'Expired Token'
        '403':
          description: "Trying to get a transaction where `from` or `to` is not the logged user"
      security:
        - loggedUserToken: []
    put:
      operationId: updateTransaction
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateTransaction'
        required: true
      responses:
        '202':
          description: 'Successful response.'
        '401':
          description: 'Expired Token'
        '403':
          description: 'Trying to update a transaction where `from` is not the logged user'
      security:
        - loggedUserToken: []
    delete:
      operationId: deleteTransaction
      responses:
        '204':
          description: 'Successful response.'
        '401':
          description: 'Expired Token '
        '403':
          description: 'Trying to remove a transaction where `from` is not the logged user'
      security:
        - loggedUserToken: []
    parameters:
      - name: transactionId
        in: path
        required: true
        schema:
          type: string
```

Note that for all operations under `/transactions/{transactionId}` path I haven't redefined every time the parameter `transactionId`: I have defined once at path level.

Status has only the _retrieve_ operation, but I want to let user customize the output based on transactions insertion datetime: clients can use query parameter `till` to ask the status till the date time provided, excluding newer transactions. You can use it to throw back in your house mate face that he didn't pay the bills for a quite long time.

```yaml
/status:
  get:
    operationId: getUserStatus
    parameters:
      - name: till
        in: query
        required: false
        schema:
          type: string
          format: 'date-time'
    responses:
      '200':
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Status'
      '401':
        description: 'Expired token'
    security:
      - loggedUserToken: []
```

### User and User relationships

The service supports creation and retrieval of users and user relationships. For simplicity I avoided to include **U** and **D** operations for user and user relationships.

I want to expose an endpoint to retrieve all registered users and an endpoint to retrieve only users that have a relationship with logged user:

```yaml
/users:
  get:
    operationId: getUsers
    parameters:
      - name: filter
        required: true
        in: query
        schema:
          type: string
    responses:
      '200':
        content:
          application/json:
            schema:
              type: array
              items:
                $ref: '#/components/schemas/Username'
      '401':
        description: 'Expired token'
    security:
      - loggedUserToken: []
/users/connected:
  get:
    operationId: getConnectedUsers
    responses:
      '200':
        content:
          application/json:
            schema:
              type: object
              properties:
                allowedTo:
                  description: "Users that logged user can bill"
                  type: array
                  items:
                    $ref: '#/components/schemas/Username'
                allowedFrom:
                  description: "Users that can bill the logged user"
                  type: array
                  items:
                    $ref: '#/components/schemas/Username'
      '401':
        description: 'Expired token'
    security:
      - loggedUserToken: []
```

In `getUsers` we defined a very basic search functionality with the optional query parameter `filter`

In `getConnectedUser` I prefeered to define the request schema directly inside the request body definition because It's a schema strictly related to this operation and It isn't parent of any other schema.

This is the endpoint to create a user connection (user relationship):

```yaml
/users/connected/{userToConnect}:
  post:
    operationId: connectUser
    parameters:
      - name: userToConnect
        required: true
        in: path
        schema:
          $ref: '#/components/schemas/Username'
    responses:
      '201':
        description: 'User connected'
      '401':
        description: 'Expired token'
    security:
      - loggedUserToken: []
```

### Login, registration and JWT

When an user wants to start using this API he must **authenticate** with his credentials following this process:

1. User calls the `/login` endpoint passing his credentials in the request body
2. The backend checks if credentials are correct
3. The backend writes the response with a JWT token containing the username inside the payload
4. User stores the received JWT token

For each request the server must **authorize** the user. The user must include inside each request the header `Authorization: Bearer <jwt token>`. When the backend receives the request it checks the signature validity and the token expiration time. If the token is valid It parses the payload, where It can read the username of the logged user.

This is the `login` operation definition:

```yaml
/login:
  post:
    operationId: login
    requestBody:
      content:
        application/json:
          schema:
            required:
              - username
              - password
            type: object
            properties:
              username:
                $ref: '#/components/schemas/Username'
              password:
                type: string
      required: true
    responses:
      '200':
        description: 'Returns the JWT token'
        content:
          text/plain: {}
      '400':
        description: 'Wrong username or password'
```

The `register` operation creates a new user and logins it:

```yaml
/register:
  post:
    operationId: register
    requestBody:
      content:
        application/json:
          schema:
            required:
              - username
              - password
            type: object
            properties:
              username:
                $ref: '#/components/schemas/Username'
              password:
                type: string
      required: true
    responses:
      '200':
        description: 'Returns the JWT Token'
        content:
          text/plain: {}
      '400':
        description: 'Username already exists'
```

I don't cover in this tutorial the logout process, but I want to give you a tip: create a whitelist or blacklist of tokens.

As you already saw, each secured operation has the `security` field:

```yaml
security:
  - loggedUserToken: []
```

The `security` field is called **security requirement** and it tells the user that he needs `loggedUserToken` **security schema** to access to this endpoint. Security schemas must be defined under `#/components/securitySchemes`:

```yaml
securitySchemes:
  loggedUserToken:
    type: http
    scheme: bearer
```

## Some resources to learn Web API Design and OpenAPI

I give you a couple of useful links:

* [Rest API Tutorial](https://www.restapitutorial.com/): Very simple and coincise tutorial for newbies of REST APIs world
* [API Stylebook](http://apistylebook.com/): Collection of API styleguides from different IT companies
* [OpenAPI Specification repository](https://github.com/OAI/OpenAPI-Specification/): Contains the spec and examples
* [OpenAPI Directory](https://apis.guru/browse-apis/): Collection of OpenAPIs of different public APIs

## Conclusion

You can find the complete OpenAPI definition here: [/src/main/resources/debts_manager_api.yaml](https://github.com/slinkydeveloper/debts-manager/blob/master/src/main/resources/debts_manager_api.yaml)

After you learnt how to design a REST API, approacching to OpenAPI is very simple. The operation definition is very intuitive because of 1:1 mapping with HTTP (methods, parameters, status codes, content types and so on). The tricky and magic part, for me, is definining and organizing the JSON Schemas. When you define simple models, you tend to put everything inside the same file. But when you raise the complexity using composed schemas, you get flooded by smaller and unclear schemas. My suggestion for you is to document the schemas with `title` and `description` keywords and organize these in multiple files.

In next chapter I'm going to bootstrap the project and start writing first Vert.x code, stay tuned!
