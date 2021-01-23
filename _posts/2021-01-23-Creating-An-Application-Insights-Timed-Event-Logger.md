---
layout: post
title: Creating An Application Insights Timed Event Logger
published: true
---

Application Insights is an application monitoring service for developers to monitor their live applications.  There are a number of different types of telemetry that can be created, many of which are automatically created for you when you [configure it for your .net core web app](https://docs.microsoft.com/en-us/azure/azure-monitor/app/asp-net-core).  You can also create telemetry as needed using the TelemetryClient.  [This doc](https://docs.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics) is a good reference for the different types of telemetry that can be created.

## Custom Events

It's common to want to measure how a piece of code performs over time.  Custom Events allow us to do just that, and then we can see the data in our azure dashboard.  

I want to note that I'm not going to cover how to create and initialize a TelemetryClient instance in your app, there are lots of resources including the one linked above that covers those details.  This section of code assumes that you have a configured TelemetryClient that has been injected into the current process.

The [TrackEvent](https://docs.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics#trackevent) method has a couple of overloads that allow us to add some additional properties and metrics that get sent with the event.  Those can be used as measurements or display values in charts and when querying on your data in the Azure Portal.  

## First take

When I first got started with Custom Events, my code looked something like the following.  It creates a new `Stopwatch`, adds some custom properties and metrics, and then sends the data to the App Insights service.  This is great because we can now add logic like this across the app to track the performance of frequently run or critical processes and keep an eye on performance on a dashboard in the Azure Portal.

{% gist 1c38c2821496bfc9ff7bcf5c7f047d9a %}

The code is pretty straightforward.  Whether there are custom properties that are added or not depends on the process being tracked and the data that might be useful to view in the Azure Portal.

## Making It Reusable

I thought it would be nice to create reusable helper that removes the boilerplate code so that as we add tracking across the app, it's easier to apply.  Here is the `TimedEventLogger`.

{% gist 73c5e08b65cbb1e48fd53720284cecbd %}

It makes use of IDisposable to wrap a section of code that will be measured.  It also has methods to add custom properties or metrics if the process warrants either of those.  It also allows you to keep how you define your properties and metrics like DurationBucket or DurationSec in one place.  You might find that over time you want to add a new property globally, or change from logging in seconds to milliseconds.  That all happens in one place now.

Here you can see the usage.  This is a much cleaner experience and quicker to use as it gets plugged in across your app.
{% gist 8e919ba26c62d41ebd9fa7e30778fef7 %}

## Results in the Azure Portal

These charts show what custom metrics can look like when configured to show up as a graph on an Azure Portal dashboard.  I removed the names on each graph to protect the innocent.

![_config.yml]({{ site.baseurl }}/images/AppInsightsCharts.png)
