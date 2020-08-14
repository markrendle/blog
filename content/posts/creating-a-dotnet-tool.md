---
title: "Creating a dotnet tool"
description: "Building, packaging and publishing a `dotnet` tool via NuGet"
date: 2020-08-14T13:30:17+01:00
draft: false
tags:
- dotnet
- nuget
- open-source
categories:
- Coding
---

One of the great features of .NET Core is that it gives you a simple
way to create and distribute CLI tools via NuGet. You just create a Console
application, add a few entries to the `.csproj` file, and publish it to
NuGet. Then other people can install it with the `dotnet tool install`
command. I've published a couple of tools this way before, but I've just
published another one so I thought I'd take time to describe the steps
involved, as well as a handful of neat NuGet packages that I used along
the way.

## nuke-from-orbit

You know how doing `dotnet clean` (or right-click > Clean solution in Visual Studio)
doesn't actually delete all the `bin` and `obj` directories and the stuff in them?
And you know how sometimes you really, really have to do that, because something
hiding in one of those directories has got itself into such a muddle that the only
way to fix it is to nuke the site from orbit? Yeah. That's why I wrote this.

The tool is published as [RendleLabs.NukeFromOrbit](https://www.nuget.org/packages/RendleLabs.NukeFromOrbit/),
so you can install it by running:

```shell
dotnet tool install -g RendleLabs.NukeFromOrbit
```

The `-g` tells dotnet to install the tool globally so you can use it from anywhere.

This should output:

```text
You can invoke the tool using the following command: nuke-from-orbit
Tool 'rendlelabs.nukefromorbit' (version '1.0.3') was successfully installed.
```

Which helpfully tells you that you can now call it as `nuke-from-orbit`. If you run it
with the `--help` argument, it will helpfully tell you how to use it:

```text
NukeFromOrbit:
  Dust off and nuke bin and obj directories from orbit. It's the only way to be sure.

Usage:
  NukeFromOrbit [options] [<workingDirectory>]

Arguments:
  <workingDirectory>    [default: D:\blog]

Options:
  -y, --yes         Don't ask for confirmation, just nuke it. [default: False]
  -n, --dry-run     List items that will be nuked but don't nuke them. [default: False]
  --version         Show version information
  -?, -h, --help    Show help and usage information
```

That little feature comes from a NuGet package called
[System.CommandLine](https://www.nuget.org/packages/System.CommandLine),
which is pre-release at the moment but dead useful for this kind of thing.

## System.CommandLine

This package provides a command line argument parser which makes it easy to handle
flags, options and arguments, and even sub-commands if you're building something big.
You just create a `Command`, assign a `Handler` to it, and then `Invoke` it with
the `args` parameter from your `Main` method.

For `nuke-from-orbit` the command looks like this:

```csharp
var command = new RootCommand
{
    new Option<bool>(new[]{"--yes", "-y"}, () => false,
        "Don't ask for confirmation, just nuke it."),
    new Option<bool>(new[]{"--dry-run", "-n"}, () => false,
        "List items that will be nuked but don't nuke them."),
    new Argument<string>("workingDirectory", () => Environment.CurrentDirectory)
};

command.Description = "Dust off and nuke bin and obj directories from orbit. It's the only way to be sure.";
```

It's using C#'s list initializer syntax so you can add as many `Option`s and `Argument`s
as you need. The `Argument`s will be *positional* if there's more than one, in the order
that they're added in the list. The `Option`s can be anywhere, before or after the
`Argument`s.

For each option, you can specify an array of *aliases*, like `--yes` or `-y`. You can
also specify a default value for both `Option`s and `Argument`s by using a `Func<T>`
that returns the default; if you specify this, the `Option` or `Argument` will be
optional.

The `Description` property is output in the Help, just under the name of the command.

To add a handler, we just create a `CommandHandler` using an `Action` or `Func`, like this:

```csharp
command.Handler = CommandHandler.Create<bool, bool, string>(
    async (yes, dryRun, workingDirectory) =>
    {
        await Nuke(yes, dryRun, workingDirectory);
    }
);
```

`System.CommandLine` uses the names of the parameters on the delegate to match to the
`Option` and `Argument` names; `Option`s with hyphens are converted to `camelCase` so
`--dry-run` matches `dryRun`.

> **Note:** at the time of writing System.CommandLine is still in preview, so you'll
need to enable pre-release packages to use it in your project.

## Finding bin and obj folders

Have you ever written an application that worked with files or directories and muttered
dark curses at whichever API designer decided that the `File` and `Directory` classes
should be `static`, so they're impossible to test against? You are not alone.
Rejoice! for there is a NuGet package that fixes this, too:
[System.IO.Abstractions](https://www.nuget.org/packages/System.IO.Abstractions) provides
interfaces for all the static `System.IO` types, along with default implementations,
so you can write proper tests without contaminating the file system of whatever
machine the tests run on.

```csharp
foreach (var directory in _fileSystem.Directory.EnumerateDirectories(currentDirectory))
{
    var name = _fileSystem.Path.GetFileName(directory);
    if (name is null) continue;
```

It even allows you to mock `Path` so you can change things like `DirectorySeparatorChar`
and simulate Linux or Windows in your tests.

Here's an example using [NSubstitute](https://www.nuget.org/packages/NSubstitute/)
to mock the `IDirectory` interface:

```csharp
        private static IDirectory FakeDirectory(string[] directories, IPath path, string[] files)
        {
            var directory = Substitute.For<IDirectory>();
            
            directory.EnumerateDirectories(Arg.Any<string>())
                .Returns(c => directories.Where(d => path.GetDirectoryName(d) == c.Arg<string>()).Distinct());
            
            directory.EnumerateFiles(Arg.Any<string>())
                .Returns(c => files.Where(f => path.GetDirectoryName(f) == c.Arg<string>()).Distinct());
            
            directory.Exists(Arg.Any<string>()).Returns(true);
            
            return directory;
        }
```

The other really nice thing about `System.IO.Abstractions` is that you can add extension
methods to the interfaces, like this one to detect whether the file system is case
sensitive:

```csharp
public static class FileSystemExtensions
{
    public static bool IsCaseSensitive(this IFileSystem fileSystem, string directory)
    {
        if (!fileSystem.Directory.Exists(directory))
            throw new InvalidOperationException("Directory does not exist.");
        
        if (directory.Where(char.IsLetter).Any(char.IsLower))
        {
            if (fileSystem.Directory.Exists(directory.ToUpper())) return false;
        }
        else
        {
            if (fileSystem.Directory.Exists(directory.ToLower())) return false;
        }

        return true;
    }
}
```

## Not deleting Git-controlled files

There are bad people in the world. People who hate puppies, eat your last Jaffa Cake, and
add files in their `bin` and `obj` directories to source control. I'm sure they have very
good reasons for doing that, but they don't. Let's humour them anyway, and check whether
and `bin` or `obj` files are in Git before deleting them.

From the command line, you can get a complete list of files that are version controlled
using the `git ls-files` command. So we can run that, find any entries with `/bin/` or
`/obj/` in them, and add them to some kind of protected list.

I don't know if you've ever written the code to run a process and capture its output, but
it's more complicated than it should be. You have to remember to redirect standard
output, and not use ShellExecute (or is it *use* ShellExecute? I can never remember),
and then handle the events and everything. It's just annoying.

Enter another great NuGet package: [CliWrap](https://www.nuget.org/packages/CliWrap/).
This package wraps all that `ProcessInfo` and `Process.Start` business up in a really
well-designed fluent API and takes care of remembering the hard stuff for you.

Here's a call to `git ls-files` using `CliWrap`:

```csharp
private async Task<HashSet<string>> ListFileAsync()
{
    var set = new HashSet<string>(_stringComparer);

    var result = await Cli.Wrap("git")
        .WithArguments("ls-files")
        .WithWorkingDirectory(_workingDirectory)
        .WithStandardOutputPipe(PipeTarget.ToDelegate(line =>
        {
            line = _fileSystem.Path.Normalize(line);
            
            if (line.Contains(Bin) || line.Contains(Obj))
            {
                set.Add(Path.Combine(_workingDirectory, line));
            }
        }))
        .ExecuteAsync();

    if (result.ExitCode != 0)
    {
        set.Clear();
    }
    return set;
}
```

That `WithStandardOutputPipe` call works with `PipeTarget` objects: there are
implementations built-in for delegates, streams and `StringBuilder` and you
can implement your own, too.

If `git` isn't installed, or the working directory is not a `git` repo, then the
exit code will be non-zero and we can just forget it.

## Making it a tool

A `dotnet` tool is just a NuGet package with a bit of extra metadata specified in the
project file:

```xml
  <PropertyGroup>
    <PackAsTool>true</PackAsTool>
    <ToolCommandName>nuke-from-orbit</ToolCommandName>
    <PackageId>RendleLabs.NukeFromOrbit</PackageId>
  </PropertyGroup>
```

The `PackAsTool` property sets the `tool` flag in the NuGet package, and the
`ToolCommandName` is the name used to invoke the command. I'd normally go for something
a bit shorter, but I'm generally so annoyed by the time I need this one that I want
the satisfaction of typing `nuke-from-orbit` and smashing the *Enter* key.

## Publishing to NuGet

There are a bunch of ways to publish NuGet packages these days, but I wanted to try
GitHub Actions. This involves adding a *workflow* file to your git repo which is, of
course, YAML. This is the workflow for NukeFromOrbit, in a file at
`.github/workflows/dotnet-core.yml`:

```yml
name: .NET Core

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

jobs:
  build:

    runs-on: windows-latest

    steps:
    - uses: actions/checkout@v2
      with:
        fetch-depth: 0
    - name: Setup .NET Core
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 3.1.301
    - name: Install dependencies
      run: dotnet restore
    - name: Nerdbank.GitVersioning
      id: nbgv
      uses: dotnet/nbgv@v0.3.1
    - name: Build
      run: dotnet build --configuration Release --no-restore
    - name: Test
      run: dotnet test --configuration Release --no-build --verbosity normal test\NukeFromOrbit.Tests
    - name: Pack
      run: dotnet pack --configuration Release --no-build -p:PackageVersion=${{ steps.nbgv.outputs.SimpleVersion }} --output . src\NukeFromOrbit\NukeFromOrbit.csproj
    - name: Push
      run: dotnet nuget push *.nupkg -s https://api.nuget.org/v3/index.json -k ${{ secrets.NUGET_API_KEY }} --skip-duplicate
```

This is mostly the default .NET Core workflow template, with a couple of additions.

### NerdBank.GitVersioning

This step takes care of assigning version numbers to assemblies, using the "height" of
the git history (i.e. the number of commits) as the revision number, and taking the major
and minor version numbers from a `version.json` file in the root of the repo.

```json
{
  "$schema": "https://raw.githubusercontent.com/dotnet/Nerdbank.GitVersioning/master/src/NerdBank.GitVersioning/version.schema.json",
  "version": "1.0",
  "assemblyVersion": {
    "precision": "revision"
  }
}
```

The `assemblyVersion` property tells NBGV to include the `revision` in the
`AssemblyVersion` value; without this the default is just `{version}.0.0`. I want it
to include the revision because `System.CommandLine` automatically adds a `--version`
flag that prints the `AssemblyVersion` value.

The workflow uses an *output variable* from the NBGV step to set the package version
in the `dotnet pack` step.

NerdBank.GitVersioning is in the GitHub Actions Marketplace, pulled in with the
`uses: dotnet/nbgv@v0.3.1` line.

### NUGET_API_KEY

GitHub Actions provides a *Secrets* store where you can put API keys, passwords and the
like. You can store Secrets at the repo level, or at the organisation level, where you
can specify which repos the secrets are available to.

## That's it!

Hopefully this post has shown you how easy it is to publish your own `dotnet` tool,
from `dotnet new console` to CI/CD. If you have any simple tools lying around then
why not share them? And if you do, be sure to drop by
[github.com/natemcmaster/dotnet-tools](https://github.com/natemcmaster/dotnet-tools)
and submit a PR to let people know about it.

The source code for the tool, including the workflow stuff, is at
[github.com/RendleLabs/NukeFromOrbit](https://github.com/RendleLabs/NukeFromOrbit).
Let me know if you like it, or open an issue or submit a PR if it needs fixing.
