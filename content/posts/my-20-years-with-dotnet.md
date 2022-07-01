---
title: "My 20 Years with .NET"
description: "For the 20th birthday of .NET, a look back at my time with it."
date: 2022-02-13T14:25:17+01:00
draft: false
tags:
- dotnet
categories:
- General
---

It's .NET's 20th birthday: 20 years since Microsoft released v1.0 of
their managed runtime, a Base Class Library, a couple of application
frameworks, and the C# programming language. So I thought I'd do a
retrospective of my 20 years working with .NET, because I'm
extraordinarily vain and I assume people want to read stuff like this.

## 1.0 beta

It was some time in late 2001 when Microsoft released the beta version
of their new framework. I'd just returned to programming after a brief
stint as a professional stand-up comic, which I tried for a couple of
years before deciding it wasn't for me.

I was working for a company who shall remain nameless, developing and
maintaining a CRM system for non-profits that was written in
[Gupta Team Developer](https://en.wikipedia.org/wiki/Gupta_Technologies). It was a dying platform, and we needed something
to replace it. We were using Visual C++ and VB 6.0 to create ActiveX 
components, and Classic ASP for our web application interface, so
Microsoft's new hotness was an obvious line of investigation.

For the first beta, there was no Visual Studio, which made life
interesting. No IntelliSense, and no real documentation either. The
first program I wrote was a console app that reflected over every
public type in the .NET Framework assemblies and generated a very
basic set of HTML pages, one per type, divided into a directory
per namespace. It didn't extract the XML doc comments, it just
dumped out all the methods, properties, constructors, etc for
every type. I wrote the app in Vim, and compiled it from the
command line with `csc`. And then I went browsing around my HTML
pages, seeing what was there, comparing it with VB 6.0 and MFC.

## 1.0

By the time .NET 1.0 and Visual Studio .Net were released in February
2002, I'd written a handful of Windows Forms experiments and was
convinced that this was the way forward. It was more fully formed
out of the box than Java, fairly easy to learn if you were already
familiar with the Microsoft ecosystem, and had some impressive
features. ADO.NET! Data-binding! A WYSIWYG window designer! And C#!
A language that felt like C++ but without having to remember to
`delete` objects when you were finished with them!
I started using it to write utilities for internal use,
and lobbying to rewrite our product in C#. But that first version
was very rough around the edges. The only resizable collection type
was `ArrayList`, and the only dictionary was `HashTable`, neither of
which were strongly typed. If you wanted a resizable list of `int`,
you had to box them to add them to an `ArrayList`, and cast them
back to `int` when you accessed them. And if you wanted to pass
them to a method, that method had to know the `ArrayList` had `int`s
in it. It was practically a dynamically typed environment. Microsoft
had already announced that .NET and C# 2.0 would be getting "generics",
a magical new language feature that would fix this problem, so we
decided to wait before starting the big porting project.

## 2.0

Four years passed between the releases of .NET 1.0 and 2.0, which is
mad to think about now that we get a new version every year. It came
with Visual Studio 2005, and the new generics feature was
ground-breaking at the time. It was similar to C++ templates, but
much easier to work with. It was also implemented properly, thanks to
the Microsoft Research team who were working on a functional
programming language for .NET. Generics in .NET were reified, which
meant they were supported by the runtime and not just the compiler.
Using reflection, you could create new generic objects and methods
at runtime.

I finally got permission to start a project to rewrite our core
product in C#. It was, in the best tradition of 1990s line-of-business
applications, mostly just a collection of windows with too many
textboxes and comboboxes, providing basic CRUD functionality but
with an advanced "query-by-example" feature inspired by Informix 4GL.
So rather than doing a screen-by-screen rewrite, I was building my
own mini-framework to provide the same functionality.

At the time, .NET had a feature that was supposed to make working with
complex database schema less of a chore:
[Strongly Typed DataSets](https://docs.microsoft.com/en-us/dotnet/framework/data/adonet/dataset-datatable-dataview/generating-strongly-typed-datasets).
At the time, writing plain old C# objects was a chore. C# 2.0 did not
have automatic properties like this:

```c#
public class Member
{
    public int Id { get; set; }
}
```

You had to provide your own backing field and implement the `get` and
`set` manually, like this:

```c#
public class Member
{
    private int _id;

    public int Id
    {
        get
        {
            return _id;
        }
        set
        {
            _id = value;
        }
    }
}
```

Multiply that by a database with some 200 tables averaging 30 columns
each. And then implementing the code to map from a `SqlDataReader` to
those objects. [NHibernate](https://en.wikipedia.org/wiki/NHibernate)
was in early development, but it was scary "Open Source" and we
weren't allowed to use it. So Strongly Typed DataSets, which you could
pretty much just generate from your database and start using, were
The Way.

Except they were terrible, and performance was horrible, and they were
awful to work with, and data-binding to them from Windows Forms was
hideous. But I persisted, and made progress, and had the core part of
our core product running on .NET some time in 2006.

And then Microsoft released .NET 3.0.

## 3.0

Just two years after .NET 2.0, in November 2006, Microsoft released
3.0, at the same time
as Windows Vista. The ambition for these two releases had been huge; a
new file-system that was part database was one of the many features
that never made it to the final code. But we did get Windows
Presentation Foundation, a brand new desktop application framework
that improved on Windows Forms in many, many ways. The data-binding was
far better, for a start, and the layout was declared in XAML instead
of hundreds of lines of C# generated by the WinForms designer. And
applications developed with WPF *looked* modern.

So of course I decided to rewrite my rewrite in WPF.

At this point I had a junior developer working with me on the rewrite
project, but there were three senior developers and three juniors
still hard at work on the old Gupta Team Developer codebase, adding
features that customers asked for and polishing the UI as best they
could. And this junior developer and I were trying to catch up to them
and get to a point where we had a new product with the same
functionality as the old one. So throwing out thousands of lines of
code and starting again with a brand new framework, which we had to
learn as we went along, was probably one of the worse decisions I've
made in my career.

## 3.5

Less than a year after 3.0, along came 3.5, bringing with it Language
Integrated Query, or LINQ. This included LINQ-to-SQL, which allowed
you to generate proper C# classes from your database and query against
it right in your code. This was much better than Strongly Typed
DataSets, so out they went and in came LINQ. More thousands of lines
of code ditched and replaced with the new hotness.

By the end of 2008 my attempts to keep up with Microsoft's new .NET
features were seriously damaging my ability to deliver the promised
rewrite. .NET just kept getting more and more awesome, and I couldn't
help chasing after every new feature they added. Except WCF. I never
liked WCF.

I ended up burning myself out, and the company scrapped my
project in favour of a VB.NET application from a company they'd
acquired in the meantime. I didn't want to do VB, so I quit and
got a job at .NET Solutions, which sounded like my kind of place.

## .NET Solutions

My new employers were a Microsoft Gold Partner and we got to work
on some amazing projects. Getting excited about new features was
pretty much part of the job, and projects were shorter so if
something new landed while you were working on something, you could
just finish that and then jump into a new one using the latest
version. When I joined, they were just finishing up a .NET Micro
Framework project that ran on a tiny circuit board connected to a
fridge.

You know how during the World Cup Final there's a spike in demand for
electricity at half time? People think it's all the kettles being
boiled, but it's not. It's all the fridges being opened and warming
up while people put milk in their teas and coffees, and then kicking
in the refridgeration unit to cool down again. It spikes so much we
have to buy extra electricity from France to cope with it.

The idea behind this project was that all the fridges would be able
to talk to a web API that would say whether or not they could turn on.
So when everybody put the milk back and closed the door, the fridges
could coordinate and avoid all demanding extra electricity at once.

It was a fun project. The .NET Micro Framework didn't have an
`HttpClient` built in, so we had to implement our own over the `Socket`
types. And there were inexplicable bugs which turned out to be due to
voltage fluctuations causing problems with the custom ASIC on the
board or some such thing.

.NET Solutions also had a bonus scheme that rewarded, among other
things, blogging and speaking at user groups, so that was when I
started doing that stuff. The first talk I did was about the
LINQ-related features in .NET 3.5, particularly lambdas and
extension methods, which allowed you to do some funky things with
C#. For example, catching more than one exception type...

```c#
public static void Try<TException1, TException2>(
    Action action,
    Action<Exception> catchAction)
{
    try
    {
        action();
    }
    catch (TException1 ex)
    {
        catchAction(ex);
    }
    catch (TException2 ex)
    {
        catchAction(ex);
    }
}
```

The third time I gave that talk was at the Developer Developer
Developer conference at the Microsoft Campus in Reading. I was
setting up when one of the other speakers came up to me and said
"you know Jon Skeet's in the audience?" So that was terrifying.
But he loved it, and couldn't help joining in and making suggestions
for ways to push things even further, and I ended up pairing with him
once the talk was finished, when I also discovered he'd been tweeting
about it all the way through. That was the day I passed 1,000 followers
on Twitter, thanks to .NET, C# and Jon.

## ASP.NET MVC

In 2007, Scott Guthrie announced
[ASP.NET MVC](https://en.wikipedia.org/wiki/ASP.NET_MVC), a .NET
implementation of the Model/View/Controller pattern for building
web applications. Up to then, the way to create web apps with
.NET had been WebForms, an ambitious framework which tried to let
developers pretend they were building a desktop application
without having to worry about things like the HTTP protocol
or round-trips to the browser. Unfortunately this required some
ugly hacks, most notably ViewState, a potentially enormous chunk of
data that was sent back and forth between browser and server
on every request.

MVC did away with all of this, replacing it with Controllers that
could handle HTTP requests and render Views, embracing the HTTP
paradigm. It also came with Razor, a new way of combining C# with
HTML to render pages from models.

But maybe the most interesting thing about ASP.NET MVC was that it
was open source, with all the source code available on Microsoft's
CodePlex site. Before this, Microsoft, under Steve Ballmer, had
been the sworn enemy of open source software; ASP.NET MVC was the
first sign that this was starting to change.

## 4.0

.NET 4.0 and C# 4.0 were a weird release. The headline feature was
the Dynamic Language Runtime, or DLR, and the `dynamic` keyword in
C#. I think the main reason for it was to improve Office interop,
but there were more interesting things you could do with it.

Around the same time that 4.0 came out, I'd been learning Ruby, and
using an ORM called [DataMapper](https://github.com/datamapper/dm-core)
which used Ruby's `method_missing` feature to map dynamic properties 
and methods to database calls. There was a new base class in .NET 4.0,
[DynamicObject](https://docs.microsoft.com/en-us/dotnet/api/system.dynamic.dynamicobject?view=net-6.0),
that let you do pretty much the same thing, so as an experiment I wrote
a simple library that let you have a `dynamic` object and call
properties and methods on it to do database things, like this:

```c#
dynamic db = new Database(connectionString);
var order = db.Orders.FindById(id);
Console.WriteLine(order.Description);
```

I called it Simple.Data, wrote some blogs about it and announced it on
Twitter, and to my surprise it became quite popular. It's had 800K
downloads from NuGet, which isn't bad. But as people asked for more
and more features, and I added them, it became quite complicated and
difficult to maintain. Eventually what it really needed was a ground-up
rewrite, but by then I had a five year old daughter and a baby son and
time was a scarce resource, so I wound the project down. Sorry if you
liked it.

## 4.5

The big thing in .NET 4.5 was the `Task`-based asynchronous
programming model. Instead of methods with `Begin` and `End`
prefixes, we had methods that returned a `Task` which could
trigger a callback when the underlying operation, usually
some form of IO, completed. It was easier to use than the
`Begin/End` model, but still resulted in code like this:

```c#
client.GetAsync(uri)
    .ContinueWith(r => r.ReadBodyAsStringAsync()
        .ContinueWith(body => ...))
```

The levels of callback nesting in a chain with a lot of
`Task`-returning methods quickly got completely out of control.

## C# 5.0

This was hugely improved with the release of C# 5.0, with the new
`async/await` keywords, which made this kind of asynchronous
programming much easier. Now we could just write code like this:

```c#
var response = await client.GetAsync(uri);
var body = await response.ReadBodyAsStringAsync();
...
```

This was much easier to understand and reason about, although there
was an enormous amount of bike-shedding about what the actual
keywords should be, particularly `await`, which many people thought
sounded too much like `Wait` and implied that the current thread
would be blocked waiting for the result, when what actually
happened was that the current thread was returned to the thread
pool and another one would be used when the `Task` completed.

I remember spending a fair amount of time going through code
using either `Task.ContinueWith` or the old `Begin/End` methods
and replacing it with `async/await` around this time, and also
wishing there were an `IAsyncEnumerable` type.

## 4.6, 4.7, 4.8

After 4.5, .NET seemed to stagnate for a while. New features were
added, but nothing that merited a major version bump. But the C#
team were working on something completely new.

## C# 6.0

Up to this point, the C# and VB.Net compilers had been written in
C++. With C# 6.0, the compiler teams embarked on a multi-year project,
codenamed Roslyn, to rewrite the compilers in their own languages.
So the C# compiler was rewritten in C#, and the VB.Net compiler was
rewritten in VB.Net. This would make it easier to add new features,
although the only changes in C# 6.0 itself were some syntactic
sugar for string interpolation and the like.

The really revolutionary thing about Roslyn, though, was that it
represented a move to a Compiler-as-a-Service. The Roslyn packages
were published on NuGet, and you could use them to parse, modify,
analyse and generated C# code at runtime. It became possible to
write your own code analysers and fixes and publish them as
Visual Studio extensions or NuGet packages. More on this later...

## Project K

In late 2015, on the ASP.Net Insiders mailing list, which I was lucky
enough to be on,
[Damian Edwards](https://twitter.com/damianedwards) and
[David Fowler](https://twitter.com/davidfowl) announced a new project
they'd been working on: a cut-down, light-weight and, most importantly,
*cross-platform* version of .NET. It would run on Linux and Mac as
well as Windows, and was optimised for writing web applications,
without all the baggage of the full-fat, Windows-only .NET Framework.
No Windows Forms, no WebForms, no WPF or WCF, just the minimum
necessary to run ASP.NET MVC applications.

Moreover, Project K would be properly open source, developed in
the open on GitHub, and accepting contributions from the
community.

Obviously I downloaded and started playing with this right away,
and I loved it. OK, so a huge chunk of the Base Class Library was
missing, but it was C# and ASP.NET MVC and Web API working on
Linux.

At the time I was a working at a place where the new Linux container
technology, Docker, was being enthusiastically embraced. Apps were
written in Node.js and Python to run in a cluster using an
orchestration platform that I don't remember the name of, but it
wasn't Kubernetes.

Project K meant that we could use .NET in this environment, and I
found an actual use for it. I was working on a "web bug" tracking
system that would follow users through the site and then off to
external links to verify that they purchased something so we
could send them a stuffed animal. This required generating a lot
of UUIDs, and under heavy load. And it turns out that generating
UUIDs is a compute intensive task, something that Node.js is not
particularly good at.

So I wrote the simplest possible microservice in Project K,
although it might have been called ASP.NET 5 by that point,
which had a single endpoint that just returned a `Guid` as
plain text. The Node.js app could call this service asynchronously,
which is something that Node is *very* good at, and suddenly the
system scaled. I don't know if this was the first production use
of the new .NET outside of Microsoft, but it must have been
pretty close.

## .NET Core 1.0

Project K became ASP.NET 5 became .NET Core 1.0. Along the way
the promised new `project.json` project files were scrapped,
which upset a lot of people, but the `csproj` files for Core
applications were far simpler than those for .NET and despite
being XML are mostly pretty simple to work with and edit manually
when the need arises.

I remember watching a talk given by Damian and David at NDC Oslo
where they talked about performance in .NET Core. Damian shared
an anecdote wherein [Kelly Sommers](https://twitter.com/kellabyte)
benchmarked the new Kestrel web server that was the engine of
ASP.NET Core against her own Haywire engine, written in C.
Kestrel was not fast: something like 70,000 requests per second
compared to Haywire's 1 million requests per second. Per
Damian, he asked the Kestrel team why it was so slow, to which
they responded "you didn't tell us you wanted it to be fast."

I don't know how much of that is apocryphal, but there was a
concerted effort to optimise the hell out of Kestrel, with a lot
of contributions from [Ben Adams](https://twitter.com/ben_a_adams),
who had a vested interest in the project as his company, Ilyiad Games,
make a game that runs on .NET on the server. By the time 1.0 was
released, .NET Core was actually becoming competitive on the
famous
[TechEmpower benchmarks](https://www.techempower.com/benchmarks/).

Annoyingly, when .NET Core 1.0 launched, I was working at a place
where new stuff was frowned upon so I had to continue working with
.NET 4.something and WebAPI.

## .NET Core 2.1

There was a 2.0 release, which was mainly concerned with adding back
a lot of missing APIs from the .NET Framework, but the big release
was .NET Core 2.1, which was when performance really became a feature
of the new framework. 2.1 introduced the new `Span<T>` and `Memory<T>`
types, which provided a better way of working with arrays or other
chunks of memory, including unmanaged memory. This came with C# 7.2,
which added a `ref struct` modifier that reduced copies when passing
`struct` types between methods and is, I'm pretty sure, only really
used for `Span<T>`; I've certainly never used it in my own code.

At this time I was working for one of the big investment banks,
with a remit to extract functionality from a behemoth of a WPF
application into shared services, and we *were* allowed to use
.NET Core 2.1, so I got to play with it. But the biggest thing I
actually achieved there was to use some of the .NET Core
packages that were backward-compatible with .NET 4.x to add
telemetry to the WPF application.

My favourite thing was a custom-written client for
[InfluxDB](https://www.influxdata.com/products/influxdb/). The
official client for .NET was quite inefficient: it allocated
a *lot* of `Dictionary<string,string>` or `Dictionary<string,object>`
objects for every line of metrics, and was too slow for our
purposes. So I got to write an alternative client which used
large byte arrays and wrote metrics passed through the new
`System.Diagnostics.DiagnosticSource` model directly to
`Span<byte>`s. My only regret was that we weren't allowed to
open source that library. If I can find the time, I might recreate
it as an InfluxDB target for the new OpenTelemetry framework.

## .NET Core 3.1

Again, there was a 3.0 release, but at this time Microsoft were
using the `.1` versions of .NET Core as the "LTS" (Long Term Servicing)
releases. The big thing with 3.1 was that Windows Forms and WPF
were ported over to Core, so you could build desktop Windows apps
with it. Sadly this did not extend to making them cross-platform,
which would be a huge effort since both WinForms and WPF are
fairly thin wrappers over the underlying Windows APIs.

At the time .NET Core 3.1 was released,
[Scott Hunter](https://twitter.com/coolcsh) announced that the next
version would be .NET 5.0. .NET Core was becoming .NET; this would
be where Microsoft invested time and energy in moving the platform
forwards, and .NET 4.x would go into long term support, getting
security updates and continuing to work on new versions of Windows,
but no longer being improved or enhanced.

With this announcement came the news that the work of migrating
.NET APIs to .NET Core was finished; if something hadn't been
migrated across now, it wasn't going to be. This included Windows
Communication Foundation, the framework that had been added way
back in .NET 3.0 for building Service-Oriented Applications,
usually using SOAP over HTTP as the protocol. The official
Microsoft recommendation was to migrate WCF services to gRPC,
the current standard for building distributed systems and
microservices using the RPC pattern. gRPC had been added to
ASP.NET Core as a first-class citizen, with a brand new implementation
that was fast and efficient and worked with the rest of the
ASP.NET Core stack.

As someone who had failed a Microsoft certification twice on the
WCF test, and had fought against it on a variety of projects, I was
delighted at this news, but it made a lot of people very angry. And
I can understand why. I started playing with the new gRPC framework
and was impressed, and realised that it mapped pretty easily to
WCF's RPC concepts. In fact, it mapped closely enough that I thought
it might be possible to use Roslyn to convert WCF code to use gRPC
instead.

I spent a week learning the Roslyn SDK and managed to get it doing
a very basic conversion of a simple WCF service to gPRC, and could
see that it was possible to do more complicated things as well, and
[Visual ReCode](https://visualrecode.com) was born. Although I
briefly considered making it open source, my wife would have killed
me so I partnered with
[Gibraltar Software](https://gibraltarsoftware.com) to sell it as
a commercial product. I'm still working on it today.

## Unity

This is not exactly a .NET version, but Unity is a game engine that
uses C# as its scripting language, and provides most of the framework
classes for building games. I put it here because it was around this
time that I learned it. I used it to build an homage to the
Club Penguin games with my son; he would draw the "graphics" in felt
tip pen, and we'd scan it in and import it into the game, with
me writing C# code and teaching him as we went along. I've also
made a silly little game called *Cheese Times Tables* and actually
released it on the various mobile App Stores. And I could do all of
this with the fundamental knowledge of C# and .NET that I've picked
up over the years writing line-of-business apps and web sites.
I think that's awesome. If I could find the time I'd be working
on *Extreme Croquet*...

## 5.0 and 6.0

And that brings us to today. I've been a .NET developer for 20 years,
I've built things with WinForms, WebForms, WPF, WCF, MVC, the Micro
Framework, Silverlight, Project K, .NET Core and .NET 6.0. For most
of that time Microsoft have innovated on the framework and the C#
language and compiler platform, and that innovation has accelerated
since the Core era began. We now have Blazor, an amazing new
web app framework that can run over a WebSocket between a browser
and server, or be compiled to WebAssembly to run .NET code right
in modern browsers. It's easier to work with than WebForms ever
was, while providing a lot of the same benefits such as simplified
event handling and session state. And we've very nearly got MAUI,
the new desktop and mobile application framework that is the
evolution of Xamarin and a replacement for WPF, WinForms and UWP
that also runs on Macs and might, one day, support Linux. There's even
a hybrid MAUI+Blazor model that wraps a Blazor Server application in
a MAUI app, letting you target desktop, mobile and web with a single
codebase.

With every release, .NET gets better performance across ASP.NET Core,
Entity Framework Core, and all parts of the framework. C#, now on
version 10 (with version 11 being loudly discussed on GitHub and
[Twitter](https://twitter.com/markrendle/status/1492470230588829696)),
continues to evolve, offering ways of writing more
expressive code with more optimal and safer runtime behaviour.
The first previews of .NET 7.0 and C# 11 are likely to land any
day now, and I'm looking forward to seeing what's new there.

I don't know if I'll still be coding professionally in another 20
years; I'm getting on a bit and ideally I'll be retired by then.
But if there's a celebration of 40 years of .NET in 2042, I hope to
be there for the party.
