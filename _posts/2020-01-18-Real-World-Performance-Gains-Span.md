---
layout: post
title: Real World Performance gains with Span&lt;T&gt;
published: true
---

## What Is Span&lt;T&gt;?

.Net Core has brought many performance gains and continues to do so with every release.  There are already many great resources that cover how Span&lt;T&gt; works so I won't be going into that in detail.  Instead, I’m going to focus this post on applying it to improve performance in a real world scenario.  Here are a few excellent resources that I found myself reading over a few times while working on the sample project.

- [All About Span: Exploring a New .NET Mainstay](https://docs.microsoft.com/en-us/archive/msdn-magazine/2018/january/csharp-all-about-span-exploring-a-new-net-mainstay)
- [Memory&lt;T&gt; and Span&lt;T&gt; usage guidelines](https://docs.microsoft.com/en-us/dotnet/standard/memory-and-spans/memory-t-usage-guidelines)
- [How to use Span&lt;T&gt; and Memory&lt;T&gt;](https://medium.com/@antao.almada/how-to-use-span-t-and-memory-t-c0b126aae652)

In short, I think of it as a tool that gives us the ability to easily work with data that is already allocated somewhere without creating more allocations in the process.  Reducing allocations is valuable when performance tuning because it means there is less work for the garbage collector.  Less work for the GC means more processing power for your code.	


## Why Bother?

Normally you would want to have some level of proof that it's worth the time and effort to optimize a process before just diving in.  I find that tracking and measuring the performance of critical processes over time is often a great guide for where to spend time optimizing code.

In this case, I just wanted to work with Span&lt;T&gt; as a learning exercise.  I also wanted to be able to apply this to a real product if the results were favorable.  I pretty quickly landed on process that I know has a very high volume of executions and even if only a small gain is achieved, it will have a measurable impact.  The code found in this post and repo are a simplified version of what's found in the real product and will focus on the IdentityKeyBuilder aspect only.

The [sample code](https://github.com/RickBlouch/PerformanceTests) for this scenario can be found in IdentityKeyBuilder folders in my github repo.

## Real World Scenario

I have a process that creates an `IdentityKey` that we need to associate with records as a customer sends imports data.  Per customer, I can receive data from many different source systems that don't talk to each other.  Because of that, I often see overlap in the values that those source systems use as record identifiers.  The `IdentityKey` normalizes those identifiers to ensure record uniqueness across all source systems in our system.

Because each customer's scenario is different, how we generate the `IdentityKey` can vary by customer and is defined through a configuration that is stored as a list of which properties to include in the key (order matters).  For a very simple example, let's say we had a customer that knew their account numbers could overlap between system 1 and 2.  I would configure their `IdentityKey` to be a combination of `SystemCode` and `AccountNumber`.  

{% gist a5f143a72b56e9f75310cdb15249e7b6  %}

*The expected output here would be:*

Source System   | IdentityKey
----------------|------------
Source System 1 | S1_123456
Source System 2 | S2_123456 

## The version using StringBuilder

The original code, and the baseline for this experiment, was written with `StringBuilder`.  It's important to note that I didn't have any performance concerns with this code the way it was written.  It's also more readable and less complex which has lower maintenance costs over time.  Whether the gains are worth the added complexity depends on the scenario.



{% gist cfd475eb6a7c33a20ff35cd0b426443d %}

## Improving it with Span&lt;T&gt;

Below is the code for the same process but using Span&lt;T&gt; instead.  There are a couple of noteworthy areas of code to point out.

{% gist fae867e7b01ac4f4af62c7aaf2d30e2c  %}

By getting the length of the output string that will be created, I can initialize our `Span<char>` to the exact length that will be needed.  

You might be thinking that the overhead of calculating the length might not be worth it.  I did too, so I created two versions of this code to test it.  You can find the version that [does not have the length calculation here](https://github.com/RickBlouch/PerformanceTests/blob/trunk/src/IdentityKeyBuilder/IdentityKeyBuilder_WithSpan_NoLengthCalc.cs).  It's important to know that a functional use case for this code is that I need to support `IdentityKey` lengths of up to 1000 characters, but most of the time they are less than 250 and usually around 50-100 characters, so my benchmark scenarios reflect that.

After benchmarking, I found that the code performed better without the length calculation only when the max `IdentityKey` length was less than 600ish.  Since my scenario needs to support up to 1000 characters, I stuck with the length calculation version.  If the max identity key length was 500 or less, it would be better to use the version without the length calculation.  Both approaches result in the same number of allocations.  

{% gist 635fef0b136991f527699bc2683ff55e %}

This also seemed like a good candidate to [use a stackalloc expression](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/stackalloc) to avoid a heap allocation until the end.

{% gist 96c6bec5eb9a8a2ed66e3ba1d313d1b2 %}

This is the syntax for copying an input string into the `identityKeySpan` in the correct position.  This does not cause an allocation.  

{% gist 64f230d4d2bd2743dc10fa64e6763e5d }

Once the identityKeySpan has been created in full, I call ToString() which will allocate the final string before returning.

{% gist 0dd099c66babad61d927aa7fe90af6dc %}

## Benchmarks

|                            Method | Iterations |           Mean |        Error |        StdDev |         Median | Ratio | RatioSD |     Gen 0 | Gen 1 | Gen 2 | Allocated |
|---------------------------------- |----------- |---------------:|-------------:|--------------:|---------------:|------:|--------:|----------:|------:|------:|----------:|
|     IdentityKey_WithStringBuilder |          1 |       369.0 ns |      7.25 ns |       8.63 ns |       366.9 ns |  1.00 |    0.00 |    0.1645 |     - |     - |     688 B |
|              IdentityKey_WithSpan |          1 |       269.1 ns |      5.32 ns |       6.92 ns |       268.2 ns |  0.73 |    0.03 |    0.0362 |     - |     - |     152 B |
| IdentityKey_WithSpan_NoLengthCalc |          1 |       308.2 ns |      5.46 ns |      10.25 ns |       307.2 ns |  0.84 |    0.04 |    0.0362 |     - |     - |     152 B |
|                                   |            |                |              |               |                |       |         |           |       |       |           |
|     IdentityKey_WithStringBuilder |         10 |     3,780.4 ns |     74.38 ns |     141.52 ns |     3,745.1 ns |  1.00 |    0.00 |    1.6441 |     - |     - |    6880 B |
|              IdentityKey_WithSpan |         10 |     2,613.4 ns |     44.21 ns |      55.91 ns |     2,604.4 ns |  0.68 |    0.03 |    0.3624 |     - |     - |    1520 B |
| IdentityKey_WithSpan_NoLengthCalc |         10 |     3,107.9 ns |     70.12 ns |     194.31 ns |     3,046.0 ns |  0.83 |    0.07 |    0.3624 |     - |     - |    1520 B |
|                                   |            |                |              |               |                |       |         |           |       |       |           |
|     IdentityKey_WithStringBuilder |       5000 | 2,266,970.3 ns | 53,296.95 ns | 149,450.36 ns | 2,214,432.0 ns |  1.00 |    0.00 | 1093.7500 |     - |     - | 4581840 B |
|              IdentityKey_WithSpan |       5000 | 1,352,236.0 ns | 26,818.29 ns |  60,533.35 ns | 1,325,954.7 ns |  0.60 |    0.05 |  197.2656 |     - |     - |  831920 B |
| IdentityKey_WithSpan_NoLengthCalc |       5000 | 1,536,566.1 ns | 30,425.87 ns |  71,717.35 ns | 1,499,633.8 ns |  0.68 |    0.05 |  197.2656 |     - |     - |  831920 B |

## Summary

I'm happy with both execution speed gains and the amount of reduced allocations.  For a process that runs hundreds of millions of times per day, it adds up.  Just doing some quick back of the napkin math for 1 large customer import, this could reduce memory usage by around 1GB.  It's also opened my eyes to some other areas where it can be applied for most likely even greater gains.

