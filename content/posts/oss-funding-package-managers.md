---
title: "Funding OSS maintainers like YouTube funds creators"
description: "An idea for applying the YouTube monetization model to OSS package managers"
date: 2022-07-01T15:30:00+01:00
draft: false
tags:
- open-source
categories:
- Community
---

[Dustin Moris Gorski](https://twitter.com/dustinmoris) yesterday published
[Fund OSS through package managers](https://dusted.codes/fund-oss-through-package-managers)
on his blog over [Dusted Codes](https://dusted.codes/). In it, he suggests adding payment
mechanisms to package managers such as [NuGet](https://nuget.org) or
[NPM](https://www.npmjs.com), and that's certainly an option. But it involves creators
choosing how much to charge for their packages, and potentially developers having to raise
purchase requests every time they want to adopt a new package, and potentially have to
pay different creators for different packages on an annual basis, and so on.

As I was reading Dustin's post, a possible alternative model occurred to me: something like
YouTube Premium's subscription model.

## How does YouTube Premium work?

My understanding, and please correct [me on Twitter](https://twitter.com/markrendle) if
I'm wrong, is that when you buy a YouTube premium subscription, your money is divided
between the creators of the videos you watch, based on the total amount of time you
spent watching their videos. If I spend three hours watching people building computers on
[Linus Tech Tips](https://www.youtube.com/c/LinusTechTips) and one hour watching Hot Wheels
cars being raced on [3Dbotmaker](https://www.youtube.com/c/3Dbotmaker), then Linus gets 75%
of my subscription, and 3D gets 25%. Or something like that.

There's also a **Subscribe** button which YouTubers are extremely keen for you to click,
so I don't know if that increases the percentage of your money that they get, but I'm
going to imagine it might.

Of course, if you don't have a YouTube Premium subscription you can still watch all those
videos but you'll get shown ads, and YouTube pay a share of that ad revenue to the
creators. Unless you have a YouTube-aware ad-blocker, in which case you'll enjoy the hard
work of all those content creators and they won't get any money. Which is why I have a
Premium subscription and I turn off my ad-blocker on YouTube.

## How would that work for Package Managers?

First of all, I'm not suggesting that Package Managers should show you ads every time
you install a package. I don't even know how that would work.

But what if, instead, I could get a NuGet Premium subscription? And I pay $9.99 a month
for that subscription, and I get a token which I can add to my feed URL, something like
`https://api.nuget.org/v3/index.json?npt=9au3efoj9as3ofh9`.
Which I can configure at machine level for NuGet and `dotnet`, and in my CI
pipelines, and so on. And the people who maintain those NuGet packages can opt-in to
monetization, and then every month, a percentage of my Premium subscription money
goes to each of those creators based on how often their packages are downloaded using
that token? (Hopefully Microsoft would *not* choose to opt-in to monetization so there'd
be some money left for everybody else.)

If there's a package or a creator I particularly want to support, I can "subscribe" to
them, which would increase their share of my subscription.

And if you don't have a Premium subscription, maybe there could be a not-too-obtrusive
message every now and then, in the output of `dotnet restore` or in one of those little
message bars Visual Studio sometimes puts at the top of the document well, saying "hey,
you could support the creator of *X* package by subscribing to NuGet Premium for as little
as $4.99 a month!"

## Nobody would do this

I'd do it. And I think a lot of people might, if it was as easy as setting up a monthly
payment and changing a setting in Visual Studio or Rider or `dotnet`. And you could have
Personal and Corporate subscriptions, and maybe have leaderboards to show who's the most
generous.

I don't know if it would generate enough revenue for people to live off their OSS work,
but if it could be sending a few people a few bucks a month, without them having to set
up invoicing or PayPal or Ko-Fi donation pages, maybe it would be worth it.
