---
title: "Nullable References in Enumerables"
description: "Handling nullable reference types in IEnumerable operations"
date: 2020-12-05T13:30:17+01:00
draft: false
tags:
- dotnet
- csharp
categories:
- Coding
---

I'm a big fan of the [Nullable Reference Types](https://docs.microsoft.com/en-us/dotnet/csharp/nullable-references)
feature added in C# 8.0. It lets the compiler catch a bunch of potential runtime errors,
and I guarantee if you turn it on you'll get a bunch of warnings. As of .NET 5.0, the
entire BCL is now covered with nullable annotations, and a lot of the extended ecosystem
supports them too.

But there are situations where the compiler is unable to work out that you've done a null
check, meaning you either have to use the "null-forgiving operator" (where you append a
bang to the identifier, like `item!.Name`), disable the null checking, or just ignore the
warning.

One of these that I run across a lot is in LINQ chains. For example, I'm just writing
some code that enumerates through the `MetadataReferences` property of a Roslyn
`Project` to find the file paths of references:

```csharp
var files = project.MetadataReferences
    .OfType<PortableExecutableReference>
    .Select(p => p.FilePath);
```

In the Roslyn library, `PortableExecutableReference.FilePath` is declared as `string?`,
because apparently you can have a `PortableExecutableReference` that isn't actually a
file. So the type of this expression is `IEnumerable<string?>`.

Now, I can filter out the null file paths with a simple `Where` call:

```csharp
var files = project.MetadataReferences
    .OfType<PortableExecutableReference>
    .Select(p => p.FilePath)
    .Where(f => f is not null);
```

But the compiler isn't actually clever enough to work out what that `Where` does, so the
type of the expression is *still* `IEnumerable<string?>`. Vexing. There's actually no
easy way to deal with this within a chain of LINQ methods (that I can think of, at
least).

So, I've added an extension method to my common library of "stuff that's not in
the BCL" that filters out nulls and changes the expression type accordingly.

```csharp
public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T?> source) where T : class
{
    foreach (var item in source)
    {
        if (item is not null) yield return item;
    }
}
```

And I thought that might be useful for other people so I'm sharing it here.

## Advertisement

I ran into this while working on [Visual ReCode](https://visualrecode.com). It's an
extension for Visual Studio 2019 that helps you migrate code from .NET 4.x to
.NET Core 3.1 and .NET 5.0, including a WCF to gRPC conversion to upgrade your
services to the modern, cross-platform standard for RPC.
