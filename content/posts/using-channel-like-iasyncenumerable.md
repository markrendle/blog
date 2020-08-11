---
title: "Using Channel Like IAsyncEnumerable"
date: 2020-07-29T12:13:25+01:00
description: "How System.Threading.Channels can be used like IAsyncEnumerable in older .NET versions"
draft: false
tags:
- dotnet
categories:
- Coding
---

Ironically, as I am creating a Visual Studio extension that migrates .NET 4.x code to .NET Core 3.1, I find myself mainly coding with .NET 4.7.2 and C# 7.3. While it is technically possible to use a subset of C# 8 features outside of .NET Core 3.x, the VS SDK is a difficult beast and introducing it to new things often makes it unhappy. I've lost an afternoon tracking down a reference to `System.Collections.Immutable` that was upsetting it.

One of the big things from C# 8 that I'm missing is `IAsyncEnumerable<T>` and `await foreach`. These would be incredibly useful for getting a stream of values from a background thread and updating the UI. But I can't use them, so instead I turned to `System.Threading.Channels`, which scratched a similar itch before C# 8 came along.

If you're not familiar, `System.Threading.Channels` is a new-ish feature of .NET designed for inter-thread messaging. It's a really nice abstraction compared to things like `BlockingCollection`.

[Steve Gordon](https://twitter.com/stevejgordon) wrote a great [Introduction to Channels](https://www.stevejgordon.co.uk/an-introduction-to-system-threading-channels) article that you should read if you want to know more about them.

The fun thing I've found is that you can treat a `ChannelReader<T>` almost exactly like an `IAsyncEnumerable<T>`, and write useful extension methods over it. For example, while writing unit tests for a method that returns a `ChannelReader<Thing>`, I wanted to collate all the values it returned into a `List<Thing>`. The code was so obviously generic that I refactored it into this extension method:

```csharp
internal static class ChannelReaderExtensions
{
    public static async Task<List<T>> ToListAsync<T>(this ChannelReader<T> channelReader, CancellationToken cancellationToken = default)
    {
        var list = new List<T>();
        while (await channelReader.WaitToReadAsync(cancellationToken))
        {
            while (channelReader.TryRead(out var item))
            {
                list.Add(item);
            }
        }

        return list;
    }
}
```

You could pretty much write any of the `IEnumerable<T>` extension methods over `ChannelReader<T>`, although you'd be creating new `Channel`s and piling things on thread pools a lot, so, you know... *caveat developer*. But anyway, here's a `Where` extension method as an example.

```csharp
public static ChannelReader<T> Where<T>(this ChannelReader<T> channelReader,
    Func<T, bool> predicate, CancellationToken token)
{
    var output = Channel.CreateUnbounded<T>(new UnboundedChannelOptions {SingleWriter = true});
    var writer = output.Writer;
    Task.Run(async () =>
    {
        while (await channelReader.WaitToReadAsync(token))
        {
            while (channelReader.TryRead(out var item))
            {
                if (predicate(item))
                {
                    if (!writer.TryWrite(item))
                    {
                        await writer.WriteAsync(item, token);
                    }
                }
            }
        }

        writer.TryComplete();
    });
    return output.Reader;
}
```