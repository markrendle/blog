---
title: "Loading old-style projects with Roslyn in .NET Core"
date: 2020-08-03T20:28:08+01:00
draft: true
---

First up, let me just say that [Roslyn](https://github.com/dotnet/roslyn),
the compiler platform for C# and .NET, is amazing and I love it.
I couldn't have built [Visual ReCode](https://visualrecode.com) without it.
But I have run into an issue with the `MSBuildWorkspace` type that, ostensibly,
lets you load .NET solutions quickly and easily.

The problem is, when you use `MSBuildWorkspace` in a .NET 4.x project, it uses the
old, full-fat .NET version of MSBuild.
But when you use it in a .NET Core 3.1 project, it uses .NET Core's version of MSBuild,
and they're not compatible.
Specifically, because the `MSBuildWorkspace` internals fully parse the project file
and resolve all the targets and everything, when .NET Core's MSBuild tries to load
the .NET assemblies used in old projects' targets, it all kind of explodes.

```text
Msbuild failed when processing the file 'D:\ReCode\Samples\Hotel\Hotel\Hotel.csproj' with message: 
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\MSBuild\Current\Bin\Microsoft.Common.CurrentVersion.targets: (1489, 5): 
The "AssignProjectConfiguration" task could not be instantiated from 
"Microsoft.Build.Tasks.Core, Version=15.1.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a".
Method not found: 'Byte[] System.AppDomainSetup.GetConfigurationBytes()'.
```

## Why this is annoying me

This is annoying me because I love .NET Core and I want to use it for
everything. Right now, the Visual ReCode engine is all in the same solution as the
Visual Studio extension, which means it's written in C# 7.3 for .NET 4.7.2, and
there's a whole world of useful stuff I don't have access to: nullable reference types,
proper `Span` support, even a bunch of new APIs that I'd forgotten were new to .NET Core
like `Path.GetFullPath`.

So I'm extracting the engine from the extension, with the plan to run it as an
out-of-process server, using (of course) gRPC to talk to the VS extension. Doing this
also solves a bunch of other problems like getting out of Visual Studio's *weird* Task
scheduler, running in a 32-bit process alongside whatever other extensions are installed,
and having to spin up a whole new VS instance to do a little debugging.

But my .NET Core 3.1 gRPC app can't use `MSBuildWorkspace` to load old-style projects,
like, oh, *WCF projects*, which ReCode kinda needs to work with.

## A solution

There is another `Workspace` implementation: `AdhocWorkspace`. This is a much more
tolerant and forgiving workspace, but it takes a little more effort to use. I already
use it extensively in unit tests, where I create solutions, projects and documents
in memory from strings. The tests happily pass when running in .NET Core 3.1, so I
thought, why don't I just create an `AdhocWorkspace` from an actual solution and
its projects? All I need to do is parse the `.sln` file and the `.csproj` files
and build the thing up manually.

So I did it, 
