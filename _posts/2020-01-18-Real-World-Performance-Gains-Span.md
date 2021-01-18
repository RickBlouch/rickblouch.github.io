---
layout: post
title: Real World Performance gains with Span&lt;T&gt;
published: true
---

.Net Core has brought many performance gains and continues to do so with every release.  There are already many great resources that cover how Span&lt;T&gt; works so I won't be going into that in detail.  Instead, I’m going to focus this post on applying it to improve performance in a real world scenario.  Here are a few excellent resources that I found myself reading over a few times while working on the sample project.

- [All About Span: Exploring a New .NET Mainstay](https://docs.microsoft.com/en-us/archive/msdn-magazine/2018/january/csharp-all-about-span-exploring-a-new-net-mainstay)
- [Memory&lt;T&gt; and Span&lt;T&gt; usage guidelines](https://docs.microsoft.com/en-us/dotnet/standard/memory-and-spans/memory-t-usage-guidelines)
- [How to use Span&lt;T&gt; and Memory&lt;T&gt;](https://medium.com/@antao.almada/how-to-use-span-t-and-memory-t-c0b126aae652)

In short, I think of it as a tool that gives us the ability to easily work with data that is already allocated somewhere without creating more allocations in the process.  Reducing allocations is valuable when performance tuning because it means there is less work for the garbage collector.  Less work for the GC means more processing power for your code.	

## Why Bother?

Normally you would want to have some level of proof that it's worth the time and effort to optimize a process before just diving in.  I find that tracking and measuring the performance of critical processes over time is often a great guide for where to spend time optimizing code.

In this case, I just wanted to work with Span&lt;T&gt; as a learning exercise.  I also wanted to be able to apply it to a real product if the results were favorable.  I pretty quickly landed on a process that I know has a very high volume of executions, and even if only a small gain is achieved, it will have a measurable impact.  The code found in this post and repo are a simplified version of what's found in the real product and will focus on the IdentityKeyBuilder aspect only.

[sample code](https://github.com/RickBlouch/PerformanceTests) can be found in IdentityKeyBuilder folders in my github repo.

## Real World Scenario

I have a process that creates an `IdentityKey` that we need to associate with records as a customer sends imports data.  A single customer can import data from many different source systems that don't talk to each other.  Because of that, I often see overlap in the values that those source systems use as record identifiers.  The `IdentityKey` normalizes those identifiers to ensure record uniqueness across all source systems.

Because each customer's scenario is different, how we generate the `IdentityKey` can vary by customer.  It is defined through a configuration that is stored as a list of the properties to include in the key (order matters).  For a very simple example, let's say we had a customer that knew their account numbers could overlap between their source systems 1 and 2.  I would configure their `IdentityKey` to be a combination of `SystemCode` and `AccountNumber`.  

{% gist a5f143a72b56e9f75310cdb15249e7b6  %}
