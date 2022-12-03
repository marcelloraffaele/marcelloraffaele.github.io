---
layout: post
title: Mock your microservices with Smocker
author: rmarcello
date: 2022-12-03 00:00:00 +0000
image: assets/images/smocker-mock/smocker.png
categories: [Microservices, Cloud, Mock, Docker, Kubernetes]
comments: false
featured: true
---
Today I want to talk to you about how simple it is to use a tool that allows you to create mocks to test your microservices.
Those of you that work with microservices had the need to create test cases that require you to interact with other microservices.

[Smocker](https://smocker.dev/) allows you to create mockups of your microservices and quickly test the interaction as if you were using the originals.
You can install Smocker either using a binary Linux file or via docker. It is simple to configure using a graphical interface or programmatically via API.
Smocker allows you to define three different types of mocks:
+ *Static Mocks*: Return a static response for a given   request.
+ *Dynamic Mocks*: Returns a response with variable parts. They can be declared using Go templates and Lua.
+ *Proxy*: Forward the request to an actual server in cases where actual mocking is not possible for testing.

During this short post, I will show you its use through docker both using the graphical interface and through the configuration via API.

# Configure your Mock via User Interface

First of all, we can start Smocker server using the following command:

```
docker run -d \
  -p 8080:8080 \
  -p 8081:8081 \
  --name smocker \
  thiht/smocker
```

As you can see, there are two exposed ports:

+ Port 8080: is the mock HTTP server port that answer to Smoker requests
+ Port 8081: is the configuration port. It's the port you will use to register new mock, via user interface and API.

If we try to call an example api:
```
curl localhost:8080/hello -v
```
We will se as result the error "No mock found matching the request":
```
> GET /hello HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 666 status code 666
< Content-Type: application/json; charset=UTF-8
< Date: Sat, 03 Dec 2022 17:22:23 GMT
< Content-Length: 263
<
{"message":"No mock found matching the request","request":{"path":"/hello","method":"GET","origin":"172.17.0.1","body_string":"","body":"","headers":{"Accept":["*/*"],"Host":["localhost:8080"],"User-Agent":["curl/7.68.0"]},"date":"2022-12-03T17:22:23.5538402Z"}}
```

If we make access to the admin console via browser  [http://localhost:8081](http://localhost:8081), we can see the history of the requests made grouped per session :

![smocer-history]({{site.baseurl}}/assets/images/smocker-mock/smocker-history.png)

From here we can directly ***directly create a new mock from request*** :

![smocer-history]({{site.baseurl}}/assets/images/smocker-mock/smocker-create-mock.png)

Now that the mock is created we can run again the Curl:
![smocer-history]({{site.baseurl}}/assets/images/smocker-mock/smocker-created-mock.png)

The result is the following:
```
$ curl localhost:8080/hello -v
> GET /hello HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.68.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Content-Type: application/json
< Date: Sat, 03 Dec 2022 17:34:21 GMT
< Content-Length: 21
< 
* Connection #0 to host localhost left intact
{ message: "Hello!" }
```

Off course we can create Mock directly before the request via user interface:
![smocer-history]({{site.baseurl}}/assets/images/smocker-mock/smocker-create-mock-manual.png)

The result is the following:
```
$ curl localhost:8080/colors -v
> GET /colors HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.68.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Sat, 03 Dec 2022 17:39:56 GMT
< Content-Length: 72
< Content-Type: text/plain; charset=utf-8
< 
[
{ id: 1, name: "red"}
{ id: 2, name: "blue"}
{ id: 3, name: "green"}
]
```

It's clear that when we have many test cases and we want use automation we need a quicker way to configure our Mock Server.
Smocker can be configured using it's API on the port 8081.

# Configure Smocker using the API
Smocker can be configured using a very simple [API](https://smocker.dev/technical-documentation/api.html#api).

We can define mocks using YAML files defined as in [Smocker documentation](https://smocker.dev/technical-documentation/mock-definition.html#format-of-request-section).

When we have our Yaml files ready we can configure the smocker via a POST HTTP request.

## Reset the Smocker configuration
First of all, if our Smocker is already configured let's clean the configuration:

```
curl -XPOST localhost:8081/reset
{"message":"Reset successful"}
```

Now if we check the Smocker configuration, we can list the mocks that after a reset is an empty list:
```
$ curl -XGET localhost:8081/mocks
[]
```

## YAML Mock definition
Let's define a Yaml file with two mock requests:
```
- request:
    method: GET
    path: /football/players
  response:
    status: 200
    headers:
      Content-Type: application/json
    body: >
      [
      { "name": "Cristiano", "surname": "Ronaldo", "nationality": "Portugal" },
      { "name": "Kylian", "surname": "Mbappé", "nationality": "France" },
      { "name": "Lionel", "surname": "Messi", "nationality": "Argentina" }
      ]

- request:
    method: GET
    path: /football/player/3
  response:
    status: 404
```

The first mock returns a (little) football player list.
The second mock, return always an HTTP 404 error.

If we send the mock configuration request with the Yaml that we prepared:
```
$ curl -XPOST --header "Content-Type: application/x-yaml" --data-binary "@football.yaml"  localhost:8081/mocks

{"message":"Mocks registered successfully"}

```

We can inspect the mocks:

```
$ curl -XGET localhost:8081/mocks
[
    {
        "request": {
            "path": {
                "matcher": "ShouldEqual",
                "value": "/football/player/3"
            },
            "method": {
                "matcher": "ShouldEqual",
                "value": "GET"
            }
        },
        "response": {
            "status": 404,
            "delay": {}
        },
        "context": {},
        "state": {
            "id": "m2ehdTFVgz",
            "times_count": 0,
            "locked": false,
            "creation_date": "2022-12-03T17:58:03.4583058Z"
        }
    },
    {
        "request": {
            "path": {
        },
        "context": {},
        "state": {
            "id": "m262dTK4g",
            "times_count": 0,
            "locked": false,
            "creation_date": "2022-12-03T17:58:03.4582274Z"
        }
    }
]
```

and Call the first mock:
```
$ curl -XGET localhost:8080/football/players | jq
[
  {
    "name": "Cristiano",
    "surname": "Ronaldo",
    "nationality": "Portugal"
  },
  {
    "name": "Kylian",
    "surname": "Mbappé",
    "nationality": "France"
  },
  {
    "name": "Lionel",
    "surname": "Messi",
    "nationality": "Argentina"
  }
]
```
and the second mock:
```
$ curl -XGET localhost:8080/football/player/3 -v


> GET /football/player/3 HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.68.0
> Accept: */*
>
< HTTP/1.1 404 Not Found

```

# Concusion
Smocker is a very useful tool that can help us to test out microservices application.
It can be integrated with our Dev/Ops tool without write tons of code.
I hope you liked this post, happy tests!