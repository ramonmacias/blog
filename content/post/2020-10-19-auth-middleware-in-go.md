---
layout: post
title: "Auth Middleware in Go"
subtitle: "A simple proposal for auth middleware"
date: 2020-10-19
author: "Ramon"
URL: "/2020/10/19/auth-middleware-in-go/"
image: "img/menorca_spring.jpg"
tags:
    - Go
    - Auth
---

>  We’re just enthusiastic about what we do. -- Steve Jobs

## Proposal

I have many reasons why I fell in love with Go, when I started working with go there wasn't too much packages at that moment, so basically you needed to built everything from scratch, this was a great experience because I learnt a lot of basic concepts related with client and server comunications, that I didn't learn before in other languages because we used big frameworks such as Spring in Java. During my professional career I was focus on improve my knowledge about design, creation and maintaining of APIs, and one of the first thing that you will probably learn is how I can add a auth layer to my API. The aim of this blog is to show my proposal of a simple middleware that will authenticate your requests.

One of the packages I have been using for a long time is https://github.com/gorilla/mux so for me it makes sense to use it as a base for this middleware. If you want to see all the code we are about to talk, go here https://github.com/ramonmacias/go-auth-middleware.

## Packages

On this project we have three main packages

* **api**: this package provides an implementation of error interface for the api layer.
* **auth**: this package provides an auth.Provider interface, a JWT implementation of that interface and a cookies service.
* **middleware**: this package provides a two different approaches of how to implement the auth with a bearer token, one based on Authorization header and the other one based on cookies.

## Concerns

 Before we start with the two different approaches there is two things that we should comment. The first one is related with auth package, there you will find and auth.Provider interface and an implementation based on JWT, so if we want to create a new auth.Provider based on JWT we should use this.

 ```
 provider := auth.NewJWTProvider(signingKey, tokenExpiryTime)
 ```

 The second thing that we should talk about, is the function we have on the middleware package

 ```
 // ValidForRefresh is a function that can be used for give an
// extra context on the middleware about is the business layer
// accept this user as a valid user
type ValidForRefresh func(userSession *auth.Session) error
```

I realized that sometimes we need to apply some kind of validation before we let the user update his token, for example we can have some user status and depending on which status is the given user we should or not allow it to refresh the token (ban a user), but it could be a lot more of validation that we can apply depending on the business layer. So this func could be useful for apply those validations.

On this project you will find two different approaches of how I managed the authentication request process, the first one is using the Authorization header and the second one is based on Cookies, both have advantatges and disadvantatges, so you can choose whatever suits better on your project. Both services are using https://github.com/gorilla/mux as a base for setup the server and his routes.

## Authorization header based

The authorization header approach will take the bearer token from the Authorization http header, using the value on this way **Bearer $TOKEN**, once it receives this token, the middleware will check if the token is valid or not, if is not valid, it could be because is expired, so we will check if we are allowed to refresh the token and answer back with a specific refresh token error message, this kind of error should we used from the client in order to ask for a new refreshed token. If the token is invalid or mallformed, the middleware will answer back with a forbidden or unathorized errors. If the token is valid, then the request will reach the final handler, adding on the context request the given session.

In order to add this middleware on your routing:

```
router := mux.NewRouter()
router.HandleFunc("/", sampleHandler)
router.Use(middleware.AuthAPI(authProvider, middleware.ValidForRefresh()))
```

Then your sampleHandler should be something like this:

```
func sampleHandler(wr http.ResponseWriter, r *http.Request) {
  session := middleware.GetSessionFromContext(r)
  // Add here the rest of the handler logic
}
```

The next figure will show you a sample of how it can work your API with this middleware

![](/img/authorization-header-based.jpg)

## Cookie based

The Cookie based approach is working similar to the Authorization header, but the main difference is that we use a cookie for transport the token between the client and the server. On this case we need to use a cookie named **bearer-token** and inside the cookie value we should have our token. Once the request reach our middleware we will validate the token inside the cookie, in case the token is expired and we are able to refresh it, we will do it and we are going to update the token in the cookie so we can keep the session alive, in case the token is not valid we will expire the cookie, so the client need to ask for a new cookie (login). If the token is valid then the request will reach the final hanlder keeping the session on the request context.

In order to add this middleware on your routing:

```
router := mux.NewRouter()
router.HandleFunc("/", sampleHandler)
router.Use(middleware.AuthCookieAPI(authProvider, middleware.ValidForRefresh()))
```

Then your sampleHandler should be something like this:

```
func sampleHandler(wr http.ResponseWriter, r *http.Request) {
  session := middleware.GetSessionFromContext(r)
  // Add here the rest of the handler logic
}
```

The next figure will show you a sample of how it can work your API with this middleware

![](/img/cookie-based.jpg)