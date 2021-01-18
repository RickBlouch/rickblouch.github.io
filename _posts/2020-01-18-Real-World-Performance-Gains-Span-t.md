---
layout: post
title: Real World Performance gains with Span&lt;T&gt;
published: true
---

## Test


.Net Core has brought many performance gains and continues to do so with every release.  There are already many great resources that cover how Span&lt;T&gt; works so I won't be going into that in detail.  Instead, I’m going to focus this post on applying it to improve performance in a real world scenario.  Here are a few excellent resources that I found myself reading over a few times while working on the sample project.

- [All About Span: Exploring a New .NET Mainstay](https://docs.microsoft.com/en-us/archive/msdn-magazine/2018/january/csharp-all-about-span-exploring-a-new-net-mainstay)
- [Memory&lt;T&gt; and Span&lt;T&gt; usage guidelines](https://docs.microsoft.com/en-us/dotnet/standard/memory-and-spans/memory-t-usage-guidelines)
- [How to use Span&lt;T&gt; and Memory&lt;T&gt;](https://medium.com/@antao.almada/how-to-use-span-t-and-memory-t-c0b126aae652)

In short, I think of it as a tool that gives us the ability to easily work with data that is already allocated somewhere without creating more allocations in the process.  Reducing allocations is valuable when performance tuning because it means there is less work for the garbage collector.  Less work for the GC means more processing power for your code.	


