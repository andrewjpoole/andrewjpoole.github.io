---
layout: post
title: "Introducing the Aspire ASB Emulator UI"
categories: [Aspire, Azure Service Bus, Blazor, .NET]
tags: [aspire, azure-service-bus, blazor, tools]
image: /images//AsbEmulatorUiScreenshot.gif
published: true
---

TL/DR - If you're using the Azure Service Bus emulator with .NET Aspire and missing the ability to peek at messages or send test messages like you can in the Azure Portal or ServiceBusExplorer, I've built a tool for you! The **Aspire ASB Emulator UI** is a Blazor-based web UI that plugs right into your Aspire AppHost.

## The Problem

.NET Aspire makes it incredibly easy to spin up local resources, including the [Azure Service Bus Emulator](https://learn.microsoft.com/azure/service-bus-messaging/test-locally-with-service-bus-emulator). This is fantastic for local development as it allows us to test against a real(ish) service bus without needing an internet connection or costing a penny.

However, one thing I really missed from the "real" Azure Service Bus experience was a UI to visualise what's going on, typically the Azure Portal or one of the various standalone apps. Being able to jump into a queue, see how many messages are sitting there, peek at the dead-letter queue, or manually send a test message to trigger a workflow is invaluable. However these tools use the ASB Admin interface, which the emulator explicitly doesn't support.

With the emulator running in a container, you're often left flying blind or writing throwaway console apps just to push a message onto a queue.

Luckily through the magic of Aspire we can connect to the backend SQL database and rummage around for a table which exposes the configured entities. I wasn't able to accurately query message counts for all entity types via the database, but once we have the list of entities to parse, we can cheat and use the ordinary client to peek and get message counts that way.

## The Solution: Aspire ASB Emulator UI

I decided to build a tool to fill this gap. **Aspire ASB Emulator UI** is basically Postman for the ASB emulator!

It's a Blazor Interactive Server application that you can wire up directly in your Aspire AppHost. It connects to your emulator instance and gives you a nice UI to explore and interact with your entities.

![Aspire ASB Emulator UI Screenshot](/images//AsbEmulatorUiScreenshot.gif)

### Key Features

Here is what you can do with it right now:

* Entity Explorer**: View all your queues and topics. See real-time message counts for both active and dead-letter queues.
* Message Sender**: A full-featured message sender with a Monaco Editor (VS Code style). You can set broker properties like `MessageId`, `CorrelationId`, and `ScheduledEnqueueTime`.
* Message Viewer**: Peek or receive messages from your queues to see what's actually going on inside them.
* Canned Messages**: Save frequently used test messages and even use AI to generate templates.
* Placeholder Syntax**: Use dynamic values in your test messages like `~newGuid~` or `~now+5m~` so you don't have to manually edit JSON every time you send a test.

## Getting Started

Getting it running is super simple. It's designed to feel just like adding any other resource in Aspire.

First, add the hosting package to your **AppHost** project:

```bash
dotnet add package AspireAsbEmulatorUi.Hosting
```

Then, in your `Program.cs`, just wire it up to your Service Bus resource:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Add Azure Service Bus and configure it to run with the emulator
var serviceBus = builder
    .AddAzureServiceBus("myservicebus")
    .RunAsEmulator(c => c.WithLifetime(ContainerLifetime.Persistent));

// Add the ASB Emulator UI
builder.AddAsbEmulatorUi("asb-ui", serviceBus);

builder.Build().Run();
```

That's it! When you run your Aspire project, you'll see a new resource called "asb-ui" in the dashboard. Click the endpoint, and you're in.

## Why I Built It

I'm a big fan of great developer experience. Friction in local testing—like having to manually craft a JSON payload and run a script just to test a message handler—adds up over time. Tools like this help keep the flow going. It's very easy to create small tools like this and integrate them with an Aspire orchestration and also to share them. Conversely it is often _not_ very easy to deploy even simple tools like this into a real enterprise environment. 

Plus, it was a great excuse to play with **Blazor Interactive Server** and **Tailwind CSS** in a .NET 10 context!

### One interesting Aspire related rabbit hole...

When building the hosting extension, one challenge I faced was getting the SQL password for the emulator's backend database. The emulator resource has an environment variable `MSSQL_SA_PASSWORD`, but in Aspire, these values are often expressions or references that get evaluated at runtime. To get the actual value during the resource wiring phase, you can't just read the environment variable dictionary directly. Instead, you have to use `ProcessEnvironmentVariableValuesAsync` to resolve the final value.

Here is how I grabbed the password from the emulator resource to pass it to the UI:

```csharp
// Process container environment variables to extract the SQL password
await sbResource.ProcessEnvironmentVariableValuesAsync(
    context.ExecutionContext,
    async (key, unprocessedValue, processedValue, exception) =>
    {
        // Capture SQL password
        if (key == "MSSQL_SA_PASSWORD")
        {
            // processedValue contains the actual resolved password!
            context.EnvironmentVariables["asb-sql-password"] = processedValue;
        }
    },
    context.Logger,
    CancellationToken.None);
```

## Check It Out

The project is open source and available on GitHub. I'd love for you to give it a try and let me know what you think.

[View on GitHub](https://github.com/andrewjpoole/aspire-asb-emulator-ui)

Happy coding! 🚀
