---
title: "Creating a dotnet tool"
date: 2020-08-12T18:30:17+01:00
draft: true
---

One of the great features of .NET Core is that it gives you a simple
way to create and distribute CLI tools via NuGet. You just create a Console
application, add a few entries to the `.csproj` file, and publish it to
NuGet. Then other people can install it with the `dotnet tool install`
command. I've published a couple of tools this way before, but I've just
published another one so I thought I'd take time to describe the steps
involved, as well as a couple of neat NuGet packages that I used along
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


