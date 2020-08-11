---
title: "Loading old-style projects with Roslyn in .NET Core"
date: 2020-08-11T17:08:08+01:00
draft: false
tags:
- dotnet
- roslyn
categories:
- Coding
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
proper `Span` support, `IAsyncEnumerable`, and even a bunch of new APIs that I'd
forgotten were new to .NET Core like `Path.GetFullPath`.

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

It's a well-known fact that the Visual Studio solution file is weird and hard to
parse, but if all you're doing is looking for projects it's actually not that bad.
A Project entry looks like this:

```text
Project(""{FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}"") = ""Hotel"", ""Hotel\Hotel.csproj"", ""{0AB9EB14-38A1-40FD-B093-B756E9679FE5}""
EndProject
```

Se we can just look for lines that start with `Project`, skip past the `=` symbol and grab
the next two quoted values, which are the name of the project and the path relative to the
solution file.

With the path to the `csproj` file we can open that up and parse it using `XDocument`.
If the project is the old-style `csproj`, the top-level `<Project>` element will have an
`xmlns` attribute, which we'll need to use to find elements.

```csharp
var itemGroupName = XName.Get("ItemGroup", xmlns);
var compileName = XName.Get("Compile", xmlns);
```

All the C# files (or "Documents" as Roslyn calls them) will be in `<Compile>` elements

```xml
<ItemGroup>
  <Compile Include=""HotelService.svc.cs"">
    <DependentUpon>HotelService.svc</DependentUpon>
  </Compile>
  <Compile Include=""IHotelService.cs"" />
  <Compile Include=""Properties\AssemblyInfo.cs"" />
</ItemGroup>
```

We also need to handle references, both to other projects in the solution, and to
.NET assemblies which might be in a `packages` folder, the Global Assembly Cache, or
just stored somewhere on disk. Project references are in `<ProjectReference>` elements,
and all the others, including to NuGet packages, are in `<Reference>` elements, with a
`<HintPath>` element if they're not in the GAC.

Loading these into an `AdhocWorkspace` is done using Roslyn's `*Info` classes, which are
like builders for Solutions, Projects and Documents. For each document, we create a
`DocumentInfo` object:

```csharp
var di = DocumentInfo.Create(DocumentId.CreateNewId(projectId), documentSource.Name,
    filePath: documentSource.Path,
    loader: new FileTextLoader(documentSource.Path, null));
```

With a list of all the documents, as well as the Project and Assembly references we
got from the project file, we can create the `ProjectInfo` object:

```csharp
return ProjectInfo.Create(projectId, VersionStamp.Default, projectSource.Name, projectSource.Name, LanguageNames.CSharp,
    projectSource.Path,
    projectReferences: projectReferences,
    metadataReferences: metadataReferences,
    documents: documents);
```

And with all our `ProjectInfo` objects we can create a `SolutionInfo` and add it to our
`AdhocWorkspace`:

```csharp
var workspace = new AdhocWorkspace();
var solutionInfo = SolutionInfo.Create(SolutionId.CreateNewId(),
                    VersionStamp.Default, solutionFilePath, projectInfos);
workspace.AddSolution(solutionInfo);
```

This will turn all those `Info`s into actual Roslyn projects and documents that we can poke
around in using `SyntaxNode`s and `Symbol`s in our .NET Core 3.1 application.

### New "SDK-style" Projects

Some projects in the solution might be the new SDK-style that came out of .NET Core but can
be used for .NET 4.x projects as well. These are much simpler than the old-style projects,
but actually harder to work with in this instance because all the default references
(e.g. mscorlib, System.Runtime, etc) are inferred from the SDK. In this instance you can
actually use `MSBuildWorkspace` to load the project up and then just grab the `Document`,
`ProjectReference` and `MetadataReference` info and copy it into the `AdhocWorkspace` so
everything works together.

## It's on NuGet

I've wrapped all this up in a NuGet package,
[RendleLabs.LegacyWorkspaceLoader](https://www.nuget.org/packages/RendleLabs.LegacyWorkspaceLoader/),
and the source code is at
[github.com/RendleLabs/LegacyWorkspaceLoader](https://github.com/RendleLabs/LegacyWorkspaceLoader).

In its current state, it works for my use case, but I'm sure there are edge cases that it
doesn't catch. If I run into any while using it in [Visual ReCode](https://visualrecode.com)
I'll fix them up and update the package, but if you take it for a spin and run into problems,
please create an Issue on the GitHub repo or send a pull request.
