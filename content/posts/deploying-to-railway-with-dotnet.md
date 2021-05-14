---
title: "Running a .NET 5 web app on Railway"
description: "I built a small ASP.NET Core 5 web app and deployed it to Railway in about 2 hours."
date: 2021-05-13T17:30:17+01:00
draft: false
tags:
- dotnet
- csharp
- cloud
- streambadger
categories:
- DevOps
---

I've been working on a fun little project on
[my stream](https://twitch.tv/markrendle) recently. It's an app
called *StreamBadger* that adds
overlay images and plays sounds on the stream through OBS Studio.
It uses the Twitch API so it needs to authenticate with Twitch, and as it's
a desktop application I wanted to build that authentication flow where you
log in through the browser and the app sits and waits and then magically
gets connected.

This is pretty straightforward: open a browser page, passing some kind of
unique key, then poll an endpoint on the server until it returns the mystic
runes you seek. So this needs to be an ASP.NET Core MVC app, and it needs
some transient external storage like, say, a Redis instance. And this is
an open source side-project and I really don't want to be paying for
hosting for an essential component of it.

A couple of weeks earlier, [Khalid](https://twitter.com/buhakmeh) tweeted
a link to a .NET app running on a `.railway.app` sub-domain, and I'd been
intrigued and checked out what that was. Turns out,
[Railway](https://railway.app) is a cloud platform that provides really,
really simple hosting and deployment, as well as plug-ins for things like
Redis, MongoDB and Postgres. The good bit is that it's **free** for up
to three projects. So I thought I'd give it a try and last night,
on the stream, I went for it.

## Creating a Railway project

I used the Railway web UI to create the project and added the Redis plugin
to it. This sets some variables like `REDISHOST` and `REDISPASSWORD` which
are available as environment variables at runtime. I also added my Twitch
app details as custom variables here.

## Installing the Railway CLI

Railway provides a CLI that lets you set up projects, run them locally
and deploy them to the cloud with simple commands like `railway run`.
Unfortunately it doesn't work very well on Windows, but that's no problem
these days because Windows runs Linux, and it works perfectly there.
I initially tried to install it using NPM, but that didn't work. That's not
Railway's fault: NPM just didn't work.

Luckily for me Jake from Railway was in the stream chat and he
suggested using the Curl installation, and very helpfully pasted the
command into the chat. So I copied and pasted it into my WSL terminal,
because running Linux commands from my Twitch chat with `sudo` is how I roll.

Then I `cd`'d into the Windows project directory from my WSL shell
(`cd /mnt/d/Twitch/StreamBadger`). Technically there's a tiny performance
overhead from working with Windows directories in WSL, but in this case
it made no noticeable difference.

## Logging in

First up I had to do `railway login` which, funnily enough, took me through the
exact same process I'm trying to create for StreamBadger: it opened a
browser page and, because I was already logged in, just magically
authenticated. Now my WSL session was connected to my Railway account.

## Linking the project

On the Railway dashboard page from my project there was yet another
command I could just copy and paste into my terminal to link the current
directory with the Railway project.  So I did that, but not with `sudo`
this time because that was not necessary. I linked from the project root
directory where the `.sln` and `Dockerfile` files were.

## Adding a Dockerfile

Railway runs a bunch of different things like Node.js, Python and Ruby,
but it will also just run Docker images, which means it can handle pretty
much anything. If there's a Dockerfile in your directory, it will just use
it. So I created a standard Dockerfile for my web app:

```docker
FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build-env
WORKDIR /app

# Copy csproj and restore as distinct layers
COPY ./StreamBadger.sln .
COPY ./src/StreamBadger/StreamBadger.csproj ./src/StreamBadger/
COPY ./src/StreamBadgerDesktop/StreamBadgerDesktop.csproj ./src/StreamBadgerDesktop/
COPY ./src/StreamBadgerLogin/StreamBadgerLogin.csproj ./src/StreamBadgerLogin/
RUN dotnet restore

# Copy everything else and build
COPY . .
RUN dotnet publish --no-restore -c Release -o out ./src/StreamBadgerLogin

# Build runtime image
FROM mcr.microsoft.com/dotnet/aspnet:5.0
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["./StreamBadgerLogin"]
```

I also added a `.dockerignore` file to ignore `bin` and `obj` directories.

```
**/bin/
**/obj/
```

This is important if you're developing on Windows because the `obj` directory
contains a file called `project.assets.json`, which is like a lock file
for Nuget packages and such. If you copy this into the Docker image, which
is running Linux, then the `dotnet restore` build step will explode and kill
your cat.


## Running the project locally

You can test your code straight away, pointing at the actual resources in
the cloud, like my Redis plugin. I just had to tweak the code to read the
connection details from the environment variables mentioned earlier. Then
it's just `railway run` and it builds the Docker image and runs your app
on `localhost`. Slightly weird thing, though: it arbitrarily chooses a port
to bind to. In my case it chose 4411. This is also exposed as an
environment variable, `PORT`, which I believe is quite a common practice in
the land of script. Jake from Railway pointed this out and I added some code
to my app to use this port:

```c#
    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();

                // For running in Railway
                var portVar = Environment.GetEnvironmentVariable("PORT");
                if (portVar is {Length: >0} && int.TryParse(portVar, out int port))
                {
                    webBuilder.ConfigureKestrel(options =>
                    {
                        options.ListenAnyIP(port);
                    });
                }
            });

```

Then I did `railway run` again and there was my app, happily running in
Docker and connecting to the Railway Redis instance.

## Deploying to the cloud

The theory here is that if `railway run` works on your machine, then it
should work in the cloud too. To deploy it, you just use `railway up`.
That's it. Under the covers it builds your Docker image, pushes it to
Railway's registry, spins up the container and connects it to your sub-domain
(in this instance
[streambadger-production.up.railway.app/](https://streambadger-production.up.railway.app/)). And it just worked. Took a couple of minutes and there it
was.

## Connecting an actual domain

I've bought [streambadger.com](https://streambadger.com) because of course
I have, so I wanted to get that hooked up as well. I do pretty much all my
DNS through Cloudflare because they're awesome and do magic SSL and such,
so I just needed to set up a CNAME pointing to the Railway sub-domain and...
no wait, infinite redirect loop. I'd missed a helpful bit in the
[Provider Specific Instructions](https://docs.railway.app/deployment/custom-domains#provider-specific-instructions)
that tells you to use the *Full* SSL/TLS encryption mode with Cloudflare.
This is because Railway just won't serve your site over plain old `HTTP`, but Cloudflare's *Flexible* encryption mode only proxies to `HTTP`.

## In summary, then

I'm super impressed with [Railway](https://railway.app).
They've obviously worked very hard on the
user experience, and it shows. I had Jake in the stream chat to help me out
so I wasn't having to refer to
[the docs](https://docs.railway.app/), but they're really good. The one
small hiccup was the Windows issues with the CLI, which isn't great, but
I'd expect any developer in 2021 to be running WSL anyway.

You get three projects with two plugins per project on the Free plan, with
a couple of environments (e.g. Staging and Production) as well. The next
step up is the $20/month "Early Adopter" plan, which is just unlimited
everything: projects, plugins, environments, deploys. And access to something
called Teams, which I don't know what that is.

I've got a few things I run online, and I'm pretty sure if I moved them all
over to Railway on that $20 plan I'd be saving money over the various cloud
platforms I'm using at the moment. It won't replace Azure for more complex
stuff with a wider range of platform services, but for side projects and
simple sites or APIs I think Railway looks great.

## Disclaimer

This post was **not** sponsored or requested by Railway. I used it
and liked it and just wanted to share.

## Twitch

If you want to watch me learn more stuff like this in real-time, stop by
[my Twitch channel](https://twitch.tv/markrendle) some time.
I'm usually on at 7pm UK (6pm UTC) on Tuesdays and Thursdays.
