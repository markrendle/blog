---
title: "C# Union Types for Polymorphic JSON"
description: "A modelling technique for providing JSON-friendly 'abstract' types"
date: 2021-05-06T17:30:17+01:00
draft: true
tags:
- dotnet
- csharp
- json
categories:
- Coding
---

I'm currently working on a project that deals with the
[ECF 5.0 XML Schema](https://github.com/oasis-tcs/legalxml-courtfiling-5.0-spec),
for legal court filings. Fun, right? It actually kind of is, because some
of the XML Schema features that the spec uses are incompatible with the
built-in .NET code generators, so I wrote my own. The C# code that it
generates is kind of horrible, because XML is kind of horrible, so I'm also
hand-crafting an object model that maps to the generated types but
(a) is less horrible to use in application code and (b) serializes to JSON
nicely.

One feature of XML Schema that is extensively used by the ECF spec (and,
incidentally, is one of the features that the .NET generator doesn't like) is
[Substitution Groups](https://www.w3.org/2002/ws/databinding/examples/6/09/SubstitutionGroup/).
This allows an element in a sequence or list to be replaced by another 
element, for a kind of polymorphism. The fun thing is that the elements
can be completely different types, without needing a base class.

To model this in C# I wanted to use something like a Discriminated Union
type. In F# a discriminated union is a type that can be one of a set of
defined types.  Here's an example from
[the F# docs](https://docs.microsoft.com/dotnet/fsharp/language-reference/discriminated-unions).

```fsharp
type Shape =
    | Rectangle of width : float * length : float
    | Circle of radius : float
    | Prism of width : float * float * height : float
```
