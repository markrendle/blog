
---
title: "Handling Session changes in Cocoon"
description: "A look at how the Cocoon Session provider handles changes made to values during a request lifetime"
date: 2021-01-08T18:30:17+01:00
draft: true
tags:
- dotnet
- aspnet
- cocoon
categories:
- Coding
---

[Cocoon](https://github.com/visualrecode/cocoon) is a project I'm working on to help
developers migrate legacy ASP.NET applications to .NET Core. It works by using
[YARP](https://microsoft.github.io/reverse-proxy/) in an ASP.NET Core web application
to proxy calls to a backend server running the old app. Then you replace a page at a
time by adding endpoints to the new application that match the old URLs.

The things that Cocoon handles are those cases where the new front-end application
needs to interact with the internals of the old back-end application, like
authentication and session-handling.

The first iteration of the Session provider was very basic, and had `GetAsync` and
`SetAsync` methods which called the back-end to get and set values. There were a
couple of issues with this approach. Firstly, that's obviously an expensive way to
set session values, incurring an HTTP request every time. Secondly, there's
an issue with the way Session is often used in old ASP.NET code. Particularly for
in-proc session, it is common to store an object with multiple properties. When you
get this object from Session, you get a reference to it. So, if you change a property,
that property is persisted in the Session. The original Cocoon code wouldn't do this;
you'd have to change the property and remember to call `SetAsync` to save it.

## Deferring Session save to the end of the Request

The solution was to make the Cocoon Session provider automatically save changed values
at the end of the Request lifecycle. Fortunately some of the design decisions in
ASP.NET Core and [the Dependency Injection library](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)
made this very easy.

Firstly, you can register a dependency as `Scoped` with the DI container. `Scoped` means
that the first time a dependency is requested within an `HttpContext` lifetime, an
instance is created, and that same instance is then used for the rest of the request.

Secondly, if a dependency type implements `IDisposable` or `IAsyncDisposable`, the
DI container will call `Dispose` or `DisposeAsync` when the response is complete and
the `HttpContext` is cleaned up.

So all I needed to do was make sure that `CocoonSession` was registered as `Scoped`,
and add code to a `DisposeAsync` method to look for changes and save them to the
back-end.

The internals of the session provider already used a `Dictionary<string, object>` as
a cache for retrieved values, and `SetAsync` was updating that cache with new values.
To check for changes, I added another `Dictionary<string,byte[]>` that held the
original values from the back-end in their raw serialized form. The `DisposeAsync`
method then serializes the values in the cache and compares the two `byte[]` arrays.
If they're different, it sends the new value to the back-end. And because this process
is run after the response is completed, there is no impact on performance from the
user's perspective.
