---
title: "One way to architecture your app"
meta_title: ""
description: "A simple way to architect your golang application without a fancy design"
date: 2023-09-30T05:00:00Z
categories: ["golang", "architecture"]
author: "Hugo Sjoberg"
tags: ["golang", "architecture"]
draft: false
---

# One way to architecture your app

One common question/talking point in the golang community is how to architect your application. It's a pretty loaded question and a lot of people have a strong opinion on what's the "right" way. I don't think that there is one way of architecture your app, that being said I will present one way of architecture your app.

There are many architectures out there such as
 - *Hexagonol Architecture*
 - *Domain Driven Design*
 - *CQRS*
 - And many more

 Typically these design patterns do not have its roots in golang but rather stem from more traditional programming languages such as Java and C# where inheritance polymorphism and other OOP patterns is a big thing. While this kind of architecture might make sense in another programming language I don't necessarily think it makes sense in Golang.

# Project Structure

Let's start with the basic project structure, I typically try to follow this [resource](https://github.com/golang-standards/project-layout) as it's pretty commonly used throughout the industry. Also, the go team just wrote a [blog-post](https://go.dev/doc/modules/layout) about it.

According to [Golang-standards Project layout](https://github.com/golang-standards/project-layout) we'll need the following:

`/cmd`

This will serve as the entry point to our program eg. `go run /cmd/api/my-app.go`, the code here should be super light, just invoke code from `/internal` and `/pkg` nothing else.

`/internal`

Private app and library code, this code is only to be consumed by this application/library. This is typically where most of your app code will end up.

`/pkg`

Code that is exported, other applications/libraries may import from here, for example, if we have an auth-middleware that can be shared with multiple projects.

`/api`

Here is where OpenAPI/Swagger files should be placed.

`Makefile`

Makefile to handle the commands needed to operate the project, e.g.: `make lint` for linting the code, `make generate` to run code-generators.

# Generate your APIs

As a programmer I'm lazy and as I'm getting older I also want consistency. For example, it's annoying when your OpenAPI/swagger differs from the actual implementation, so let's generate our APIs from our OpenAPI specification.

To do this we can use a package such as [OAPI-codegen](https://github.com/deepmap/oapi-codegen) or the more flexible [goapi-gen](https://github.com/discord-gophers/goapi-gen). I prefer the latter since it allows to use middleware per endpoint instead of only adding middleware on the project level.

Below is an example spec for two APIs, one user endpoint to fetch the current user and one admin endpoint to create a new user.


```yaml
openapi: 3.0.3
servers:
  - url: 'http://localhost/v1'
components:
  schemas:
    Error:
      type: object
      properties:
        code:
          type: integer
          format: int32
        message:
          type: string
    User:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
paths:
  /user:
    get:
      description: Get's the current user
      responses:
        200:
          description: user response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        default:
          description: error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
  /admin/user:
    post:
      description: Creates a user
      responses:
        200:
          description: Creates a user
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        default:
          description: error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
      requestBody:
        required: true
        content:
         application/json:
           schema:
            type: object
            properties:
              name:
                type: string
```

And a file to generate the handlers:

```go
/internal/api/main.go
//go:generate go run github.com/discord-gophers/goapi-gen@v0.3.0 -o gen.go -package api ../../api/api.yaml
package api
```

Running `go generate ./...` will generate the interfaces for your handlers, pretty neat right? Now we just need to implement the interfaces.

# Implement the interface

We can create two new packages, `user` and `admin`, and you guessed it, all code needed for the user is in the user package and all the code needed for the admin is in the `admin` package. When I say all code needed for the user it would be all the handlers(API), all the business logic, database interactions etc. To keep things short I will just implement a super simple handlers.

```golang
// internal/user/handler.go

type UserHandler struct{}

func (u *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) *api.Response {
	id := "123"
	name := "John Doe"
	return api.GetUserJSON200Response(api.User{ID: &id, Name: &name})
}

func NewUserHandler() *UserHandler {
	return &UserHandler{}
}
```

```golang
// internal/admin/handler.go

type AdmimHandler struct{}

func (u *AdmimHandler) PostAdminUser(w http.ResponseWriter, r *http.Request) *api.Response {
	var req api.PostAdminUserJSONRequestBody
	if err := render.Bind(r, &req); err != nil {
		errorCode := int32(http.StatusBadRequest)
		errorMessage := "Bad request"
		return api.PostAdminUserJSONDefaultResponse(api.Error{Code: &errorCode, Message: &errorMessage})
	}
	id := "123"
	return api.PostAdminUserJSON200Response(api.User{ID: &id, Name: req.Name})
}

func NewAdmimHandler() *AdmimHandler {
	return &AdmimHandler{}
}
```

And we can merge the Handlers together so that we implement the interface which we generated.

```golang
// server/server.go

type handlers struct {
	user.UserHandler
	admin.AdmimHandler
}

func newHandler() *handlers {
	return &handlers{
		UserHandler:  *user.NewUserHandler(),
		AdmimHandler: *admin.NewAdmimHandler(),
	}
}

func Start() {
	apiHandler := api.Handler(newHandler())
	http.ListenAndServe(":8080", apiHandler)
}
```

Everything should run now :crossed_fingers:

# Conclusion

This architecture is not the most advanced architecture, as you can see there's not much abstraction and nothing magical(Ok, the code generation is pretty magical). I also don't think an architecture should be much more advanced than this, code is meant to be easy to read and easy to understand. The more abstractions or \<insert-buzzword\> design-pattern we use the harder it get's to read, write and refactor code.

The code used in this blog post can be found [here](https://github.com/hugosjoberg/blog-code/tree/main/architecture)
