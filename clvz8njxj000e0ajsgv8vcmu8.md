---
title: "How to mock an API in 2 minutes"
seoDescription: "Learn how to fake an API in minutes using MockServer, Docker, and a JSON file."
datePublished: Thu May 09 2024 12:42:30 GMT+0000 (Coordinated Universal Time)
cuid: clvz8njxj000e0ajsgv8vcmu8
slug: how-to-mock-an-api-in-2-minutes
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715244867789/f51c53ee-99d1-464f-8347-261e7f61e834.jpeg
tags: tutorial, testing, devops, mocking, programming-tips

---

I don't need to enumerate all the situations in which you would need to mock an API or an HTTP server. There are many options available, such as [faker](https://github.com/dotronglong/faker). This article delves into [MockServer](https://www.mock-server.com) and shows how to set up a mock server using only Docker and a JSON file. Let's get started!

🤯 🔥 MockServer can also be used as a [proxy](https://www.mock-server.com/mock_server/getting_started.html) (to record and potentially modify requests/responses), or a combination of mock and proxy!

---

* [The docker-compose](#heading-the-docker-compose)
    
* [Defining the expectations (spec.json)](#heading-defining-the-expectations-specjson)
    
    * [The most basic expectation](#heading-the-most-basic-expectation)
        
    * [Using templates](#heading-using-templates)
        
    * [Importing the request spec from OpenAPI](#heading-importing-the-request-spec-from-openapi)
        
    * [Going further](#heading-going-further)
        
* [Conclusion](#heading-conclusion)
    

---

## The docker-compose

First, let's create a docker-compose for running the latest version of MockServer:

```yaml
services:
  mockserver:
      image: mockserver/mockserver:latest
      ports:
        - "1080:1080"
      environment:
        MOCKSERVER_PROPERTY_FILE: /config/mockserver.properties
        MOCKSERVER_INITIALIZATION_JSON_PATH: /config/spec.json
        # ↓ hot reload when the spec.json changes 😍
        MOCKSERVER_WATCH_INITIALIZATION_JSON: "true"
      volumes:
        # bind the config for easy editing
        - ./config:/config
```

Nothing special here. We are using a volume for the config and telling MockServer to load the endpoints to mock from the `./config/spec.json` file.

Before starting it, let's create the necessary files (we will see how to use them later):

```bash
mkdir config
touch config/mockserver.properties
echo '[]' > config/spec.json
```

Start the mock server by executing:

```bash
docker compose up -d
```

MockServer is now running on [http://localhost:1080](http://localhost:1080)! However, since the `spec.json` - defining the expectations, or endpoints to mock - is empty, it only returns 404's. Let's change that!

💡 the `mockserver.properties` will stay empty for this article, but I recommend you check out the [configuration documentation](https://www.mock-server.com/mock_server/configuration_properties.html) for all the possibilities it offers! Dynamic SSL certificates, logging, metrics, etc.

## Defining the expectations (spec.json)

🔥 *More examples available at* [*https://github.com/mock-server/mockserver/blob/master/mockserver-examples/json\_examples.md*](https://github.com/mock-server/mockserver/blob/master/mockserver-examples/json_examples.md) 🔥

### The most basic expectation

The `spec.json` file defines *expectations*, aka the endpoints to mock and what to return. It contains an object or an array of objects defining the HTTP request to match, and the response to return.

Unclear? Here is a basic example:

```json
{
    "httpRequest": {
        "path": "/hello"
    },
    "httpResponse": {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": "{\"hello\": \"world\"}"
    }
}
```

Save the `spec.json`, wait a second and try hitting [http://localhost:1080/hello](http://localhost:1080/hello). You should see the body `{"hello": "world"}`.

This is just a simple example, but we can go way deeper. The request may define headers, method, path and query parameters, authentication, and more, all using either string or regex matching. MockServer even supports stuff such as adding delay in the response or behaving differently depending on how often the endpoint is called!

### Using templates

The response often depends on the request, and enumerating all the possible input/output pairs is tedious. MockServer supports loops, conditionals, and 3 [response template](https://www.mock-server.com/mock_server/response_templates.html) engines: Javascript, mustache, and velocity.

Let's see a velocity example *(at the time of writing, Javascript is broken in Docker, see the*[*related issue*](https://github.com/mock-server/mockserver/issues/1326)*):*

```json
{
    "httpRequest": {
        "path": "/cart/item/{id}/",
        "pathParameters": { "id": ["\\d+"] }
    },
    "httpResponseTemplate": {
        "templateType": "VELOCITY",
        "template": "#set($id = $!request.pathParameters['id'][0]) #if ($id == 1) {\"statusCode\": 500} #else {\"statusCode\": 200, \"body\": \"{\\\"name\\\": \\\"foo\\\"}\"} #end",
    }
}
```

What is happening? Instead of specifying the `httpResponse` directly, we ask MockServer to "compute" it by interpreting a template. In the example above, the server returns a `500` for the item `1` and a JSON body otherwise.

Things to note:

* `#set`, `#if`/`#end`, `#foreach`/`#end`, etc. are MockServer constructs independent of the template type, allowing for complex logic inside the response.
    
* `$id` is a MockServer local variable, but `$!request` (notice the `!`) is a velocity template directive.
    
* there is full support for regular expressions in all request attributes.
    
* Request paths and query parameters can be named and accessed in the response. The index `[0]` is necessary because MockServer uses Java multivalued maps under the hood. Here, using the request path `/cart/item/\\d+/` and the condition `($request.path == '/cart/item/1/')` would have been simpler.
    
* Since the template itself is a JSON string, multiple escapes of `"` are necessary and no wrap is possible, making it difficult to read... The joys of JSON 😆.
    

Let's check the result of the above template:

```bash
➜ curl --fail-with-body http://localhost:1080/cart/item/13/
{"name": "foo"}
➜ curl --fail-with-body http://localhost:1080/cart/item/1/
curl: (22) The requested URL returned error: 500
```

Request attributes (headers, body, path, remote address, etc.) aside, other magic variables are at our disposal in templates. For example:

* `$!now_epoch` → get the current timestamp seconds since January 1, 1970 (UNIX epoch)
    
* `$!uuid` → get a random UUID
    
* `$!rand_int_100` → get a random number between 0 and 99
    
* ...
    

Check the [docs](https://www.mock-server.com/mock_server/response_templates.html) for more!

### Importing the request spec from OpenAPI

Often, you want to simulate existing endpoints from a known API. If the latter has an [OpenAPI](https://www.openapis.org) schema, MockServer can parse it to construct the request matchers.

Let's use the dev.to API as an example, and simulate the `GET /api/articles` endpoint: [https://developers.forem.com/api/v1#tag/articles/operation/getArticles](https://developers.forem.com/api/v1#tag/articles/operation/getArticles).

```json
{
    "httpRequest": {
        "specUrlOrPayload": "https://developers.forem.com/redocusaurus/plugin-redoc-1.yaml",
        "operationId": "getArticles"
    },
    "httpResponse": {
        "statusCode": 200,
        "body": "[{\"name\": \"super article\"}]"
    }
}
```

The two lines of the `httpRequest` above transform into a 93-line long request matcher. Here is an abridged version:

```json
{
  "method": "GET",
  "path": "/api/articles",
  "headers": { "api-key": [".+"] },
  "queryStringParameters":
    {
      "keyMatchStyle": "MATCHING_KEY",
      "?username": {"parameterStyle": "FORM_EXPLODED", "values": [{"schema": {"type": "string"}}]},
      "?top": { "parameterStyle": "FORM_EXPLODED", "values": [{"schema": {"format": "int32", "minimum": 1, "type": "integer"}}]},
      // ...
    }
}
```

When importing from OpenAPI, the request matchers are way more precise. For instance, the dev.to schema requires an authentication header. Without it in a request, there is no match, and the MockServer returns a 404. It doesn't complain about invalid query parameters though, which is expected.

```bash
# Missing the authorization header
➜ curl --fail-with-body http://localhost:1080/api/articles
curl: (22) The requested URL returned error: 404
# ok
➜ curl --fail-with-body -H 'api-key: x' http://localhost:1080/api/articles
[{"name": "super article"}]
```

### Going further

To see more, check out their [documentation](https://www.mock-server.com/mock_server/getting_started.html) and the [examples](https://github.com/mock-server/mockserver/blob/master/mockserver-examples/json_examples.md). Even better, try it out yourself!

## Conclusion

MockServer is a lightweight, convenient solution for mocking APIs. The Docker image is only 276MB! A basic setup only requires Docker and a JSON file, which is powerful enough for more use cases. The hot reloading feature makes it easy to set up and test.

However, while a JSON file has its advantages, especially when paired with Docker, it can be challenging in terms of readability and comprehension. For complex APIs, opting to define specifications in Java or Javascript and then exporting them to JSON might be a better alternative. To learn more, check out [Creating expectations](https://www.mock-server.com/mock_server/creating_expectations.html).

This article only covered the basics, but I hope it got you interested in trying out MockServer the next time you need to fake an HTTP(s) endpoint.

Thanks for reading, and happy coding!