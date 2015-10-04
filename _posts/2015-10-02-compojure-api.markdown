---
layout: post
title:  "Top-Down API Design using Clojure"
date:   2015-10-02 22:26:00
categories: productivity
---

## Compojure API

I've been looking at different ways of designing and implementing REST apis using clojure and my favorite project so far is [compojure api](https://github.com/metosin/compojure-api).  You can try it out by [installing leiningen](http://leiningen.org/) and typing the following three commands:

    ~$ lein new compojure-api api-test
    ~$ cd api-test
    ~$ lein ring server-headless

At the end of this, you'll have a jetty server running on [http://localhost:3000](http://localhost:3000).  Besides hosting a few test resources, you'll also have an automatically generated [Swagger 2.0 description](https://github.com/swagger-api/swagger-spec/blob/master/versions/2.0.md) of your api.  

The "compojure-api" template pulls in a set of useful libraries:

* Jetty for the underlying servlet container
* [Jackson](https://github.com/FasterXML/jackson) for JSON encoding/decoding
* [ring](https://github.com/ring-clojure/ring), the de facto standard library in Clojure for creating HTTP request/response handlers
* [compojure](https://github.com/weavejester/compojure) for composing ring handler functions
* a convenient set of [ring-http-response functions](https://github.com/metosin/ring-http-response) to help populate your response maps (which ring turns into your HTTP Response Objects)
* a set of libraries enabling all resources to support json, yaml, yaml-http, and transit representations
* the [prismatic schema library](https://github.com/Prismatic/schema) for validating the schema of both incoming and outgoing resources.

You'll also notice that the entire project has only one src file (src/test_api/handler.clj).  This file contains exactly 3 forms.  

1.  the `(ns test-api.handler ... )` form populates this namespace with the symbols and vars you'll need to construct your api
2.  the  `(s/defschema ...)` form creates a schema definition (this template creates a single sample resource)
3.  the `(defapi ... )` form describes the api

The last of the 3 expressions is shown here:

{% highlight clojure %}
(defapi app
  (swagger-ui)
  (swagger-docs
    {:info {:title "Test-api"
            :description "Compojure Api example"}
     :tags [{:name "hello", :description "says hello in Finnish"}]})
  (context* "/hello" []
    :tags ["hello"]
    (GET* "/" []
      :return Message
      :query-params [name :- String]
      :summary "say hello"
      (ok {:message (str "Terve, " name)}))))
{% endhighlight %}

There are 3 nested expressions making up this definition:

1.  `(swagger-ui)` binds a copy of the swagger UI to the root URL of the application.  This could be left out if your swagger descrription resources were going to be rendered by another application.
2.  `(swagger-docs ...)` creates and mounts Swagger 2.0 descriptions at [http://localhost:3000/swagger.json](http://localhost:3000/swagger.json).
3.  the final expression declares that the resource described by the schema `Message` is bound to `/hello` and supports the GET method.

You could extend this to include a PUT operation, with a uri template including a name parameter:

{% highlight clojure %}
(context* "/hello" []
  :tags ["hello"]
  (GET* "/" []
    :return Message
    :query-params [name :- String]
    :summary "say hello"
    (ok {:message (str "Terve, " name)}))
  (PUT* "/thing/:name" []
    :summary      "put a new thing resource on the server"
    :body         [body Message]
    :path-params  [name :- (describe s/Str "every Message must have a name!")]
    :responses    {201 {:schema s/Str :description "Successfully create a new Message resource"}
                   400 {:schema s/Str :description "bad api definition"}}
    (do
      ; TODO create the resource here
      (created "successfully created")))))
{% endhighlight %}

After saving this change, refresh your swagger page at [http://localhost:3000](http://localhost:3000).  You'll see your new PUT operation immediately because you started this application in a "dev" mode.   This mode automatically reloads any namespaces impacted by a src file change.

### Code and Data

The `(context* ...)` expression above should be thought of as both code and data.  While designing the api, we will most likely prefer to think of it as data.  In fact, during evaluation of this expression, the `(defapi ...)` macro will treat the expression as unevaluated data too!  The macro walks the data structure and uses it to create the application's swagger data.  Those familiar with Swagger will notice an obvious structural similarity between the above expression and the Swagger syntax for an `Operation`.  

Of course, running the program will evaluate the expression, and create a set of runtime handlers.  To put it another way, evaluation of this expression generates the application's controller layer.  

This is an elegant example of a single data structure being used as both program code and data.  Seen as data, it describes our API.  Seen as program instructions, it evaluates to setup our runtime handlers.  

The runtime handlers themselves also have some very interesting properties:

* automatic generation of `4xx` errors when the request data does not match the declared schema
* automatic generation of `5xx` errors when the response data does not match the declared response schema.  This is particularly interesting because we are saying that by default, we will not allow the program to return a response that does not validate against the declared interface!
* desctructuring of the request into form, body, path, and query parameters, complete with type coercion for all parameters

Even in some projects where we've decided to predominantly use Java, we've found that creating the API top-down in this manner is advantageous before we make a call to an operation that has been written in Java.  The `(do ...)` expression in the PUT definition could be written as:

{% highlight clojure %}
; imagine there is a class com.example.Operation with a constructor taking a String argument and a
; public String execute(java.util.Map m) method
; interoperate using the following form
(do
  (created (.execute (com.example.Operation. name) body)))
{% endhighlight %}

Note that the `body` and the `name` values were destructured from the original request.  This will actually validate that the `execute` method returns a `java.lang.String` at runtime.

This means that the entire definition of the API contract is currently written into a single file.  This certainly does not have to remain the case.  In fact, we already have several large projects where we've decided to break up some of our definitions into different files.  However, the data contributing to the overall contract of the API is very localized.  The remaining code of the app (the handler logic) might not even be written in Clojure!

### Request and Response types

Compojure-api uses a really fantastic library called the [prismatic schema library](https://github.com/Prismatic/schema).  The library demonstrates a very simple principle:  use clojure data structures to express the schema's constraints.  The [documentation](https://github.com/Prismatic/schema) on Prismatic schema is a great read so we won't reproduce any examples here.  It is sufficient to point out that the choice to use clojure data structures to represent schema types, allows compojure-api to again take advantage of the fact that this data structure can be transformed into a [JSON schema definition](http://json-schema.org/latest/json-schema-core.html).

Exactly as we saw with the `(defapi ...) expression, Prismatic core schema definitions function both as data, to define the JSON definitions, and as code, to validate, and coerce incoming and outgoing data types.

### A note on Docker images

It is trivial to include a Dockerfile into the template for one of these apps.  In fact, the Dockerfile we use today is identical across all of our projects.  The most interesting thing about it is line 5 where we pre-download all of the libraries into one layer.  This ensures that the api runtime will come up quickly when you first run the container.  

    FROM clojure
    RUN mkdir -p /usr/src/app
    WORKDIR /usr/src/app
    COPY project.clj /usr/src/app/
    RUN lein deps
    COPY . /usr/src/app
    CMD ["lein", "ring" "server-headless"]

