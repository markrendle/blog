---
title: "Upgrading ASP.NET projects one page at a time"
description: "Looking at an approach to migrate old ASP.NET WebForms or MVC projects to ASP.NET Core in a gradual way"
date: 2021-01-08T18:30:17+01:00
draft: false
tags:
- dotnet
- aspnet
categories:
- Coding
---

**TL;DR:** I've come up with a way to gradually migrate old ASP.NET WebForms (or MVC)
applications to .NET 5.0, and you can watch me figure it out in real-time on
[my brand new Twitch channel](https://twitch.tv/markrendle).

---

I've been working on Visual ReCode for a while now. It's a plug-in for Visual Studio 2019
that helps developers migrate old .NET 4.x code to new .NET 5.0 or .NET Core 3.1.
[We just released version 2.0](https://visualrecode.com/blog/Visual-ReCode-2-Announcment/)
, with a new feature called
[Guided Project Upgrades](https://visualrecode.com/blog/Guided-Project-Upgrades-with-ReCode/)
that helps you upgrade your solution one project at a time,
switching to the new SDK-style `csproj` format, and adding modern target frameworks like
`net5.0` or `netstandard2.0`. It checks your NuGet packages and upgrades them if
necessary, and makes sure you upgrade projects in the right order.

Visual ReCode also provides a way to migrate WCF applications, which are not supported in
the new .NET, to gRPC, which is very much supported (in fact, it's the best-performing
gRPC implementation there is). And right now I'm working on providing the same
functionality for ASP.NET WebAPI, migrating it to ASP.NET
Core MVC, and looking at a WCF ReST to ASP.NET Core MVC feature.

That still leaves a couple of old frameworks on the table, though. **ASP.NET
MVC** is a possible target for future investigation, although there are quite a lot of
differences that mean it would be more of a "helper" than an automated migrator. But
**ASP.NET WebForms** is a whole other thing. It's been around for 20 years now;
it's really, really complicated; it's completely unlike MVC or Blazor or anything else
in the modern .NET stack. There's really no way to automate the migration of WebForms
applications to ASP.NET Core.

And the chances are, if you've got a WebForms application in
production, it's a massive behemoth of code that's built up over years, and rewriting
it would be a huge undertaking. Having worked on rewrite projects in the past, I know
from experience that they're a nightmare, and get derailed all the time by the
maintenance requirements of the existing code.

## So what to do?

The big problem is that there's no way to gradually upgrade WebForms (or MVC) to
ASP.NET Core. You can't run Core on .NET 4.x, and you can't run WebForms
on Core. But what if you could run two separate applications and make it look as if
they're the same one?

Enter **YARP**.

YARP stands for Yet Another Reverse Proxy, and it's a Microsoft project to build a
reverse proxy server on top of the blazing-fast Kestrel HTTP server. It's not a
stand-alone application; it's a package that runs in an ASP.NET Core app. You just
reference the
[Microsoft.ReverseProxy](https://www.nuget.org/packages/Microsoft.ReverseProxy/1.0.0-preview.7.20562.2)
package and add it as an endpoint in your `Startup` class.

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddReverseProxy() 
        .LoadFromConfig(Configuration.GetSection("ReverseProxy"));
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseRouting();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapReverseProxy();
    });
}
```

Being just part of the ASP.NET Core pipeline means that you can do all sorts of
creative things with your reverse proxy by adding more middleware. And of course,
one of the pieces of middleware you could add in there is ASP.NET Core MVC.

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllersWithViews();
    services.AddReverseProxy() 
        .LoadFromConfig(Configuration.GetSection("ReverseProxy"));
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseRouting();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllerRoute(
            name: "default",
            pattern: "{controller=Home}/{action=Index}/{id?}");
        
        endpoints.MapReverseProxy();
    });
}
```

Now, let's say the home page for your WebForms application is `~/Default.aspx`. If
you add an MVC action with that route to your Core application, the routing will hit
that first. Any routes that aren't hit in your application, though, will be
transparently proxied through to the old app. Now you can rewrite individual pages
in your site however you want, while continuing to support and maintain the old
application, until eventually everything is migrated and the old code can be retired.

## The complexity

If only it were just that easy. There are a couple of things in the way of this approach:
session state and authentication. WebForms applications in particular tend to be heavy
users of server-side session, whereas in ASP.NET Core it's much less used. And
authentication is very different, too.

I'm pretty sure I've figured out a way to address these issues, though. And as a new
thing for 2021, I wanted to try my hand at streaming. So I'm combining these two things:
you can join me on [my Twitch stream](https://twitch.tv/markrendle) and watch as I try
to gradually upgrade the
[Wingtip Toys sample application](https://github.com/Project6/Wingtip-Toys)
to ASP.NET Core 5.0 using this approach. I've done two streams to date, and so far
I've got the home page to look roughly the same, and implemented basic session sharing
between the two applications (that even works with "in-proc" session on the WebForms
side), as well as copying and pasting a bunch of Entity Framework 6.0 classes and
finding out that they work perfectly in EF Core 5.0.

Once I've got everything working in this test application, I'm going to package up the
various components as open-source NuGet packages so that everyone can try this approach.

If you want to catch up on the streams, I've archived them on YouTube:
[Part 1](https://www.youtube.com/watch?v=1DEXDKQguK8) and
[Part 2](https://www.youtube.com/watch?v=2V-ntu0rtq4). Or if you just want to browse
around the code, you can [find it on GitHub](https://github.com/VisualReCode/Facade).
