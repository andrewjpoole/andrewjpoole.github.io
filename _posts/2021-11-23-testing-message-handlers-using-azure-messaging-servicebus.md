---
layout: post
title: Testing Service Bus Handlers using Azure.Messaging.ServiceBus
image: /assets/images/bust_tilt_testing.jpg
published: false
tags:
  - testing
  - service bus
  - azure
  - component tests
---

# Testing Azure Service Bus Handlers using `Azure.Messaging.ServiceBus` Client Library

TL/DR - At ClearBank we love testing and the new `Azure.Messaging.ServiceBus` package makes testing Service Bus handlers easy at last! Check out how we did it.

## Background

In April 2020 Microsoft brought out a new client library for Azure Service Bus called `Azure.Messaging.ServiceBus`, this new library is based on the open Advanced Message Queuing Protocol (AMQP) standard, from the Microsoft website:
>[AMQP enables you to build cross-platform, hybrid applications using a vendor-neutral and implementation-neutral, open standard protocol. You can construct applications using components that are built using different languages and frameworks, and that run on different operating systems. All these components can connect to Service Bus and seamlessly exchange structured business messages efficiently and at full fidelity.](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-amqp-overview)

The class library documentation hints at this library being testable! This is great because one of the frustrations of using previous Service Bus client libraries was that absolutely everything was sealed or protected, meaning the only way to test client code was to wrap up and abstract away the service bus infrastructure. The situation might have been helped by a local emulator, but despite being a highly requested and up-voted feature request, an emulator was never officially implemented. So any improvements to the testability of Service Bus client code would be very welcome indeed!

The test support here is basically a few classes which are public and not sealed, a sprinkling of overridable methods and a factory class for creating models. 

There are no official samples yet and very few (if any) blog posts covering how to test, but after a bit of trial and error we managed to get some tests working that we are quite pleased with, hence sharing here so that others might also benefit. This article covers testing the very basic handling of messages, although it should also be a useful starting point for testing more advanced features.

## Component tests

At ClearBank we are favouring 'component tests' over reams of unit tests, I'm not sure who coined the name 'component test', but I see them as a kind of internal integration test. So we run some kind of test host, using real concrete objects, with the exception of anything which touches the network, these objects are mocked. We test as much of the service as possible i.e. including the `Startup` class and the `Program` class if possible. The tests include happy and sad paths, enough to give sufficiently high line and branch coverage of the tested services. These kind of tests take longer to setup than pure unit tests making TDD a bit harder to start with, but they run locally, they run much faster and are less brittle than integration/end-to-end tests, giving the same confidence that the business logic is proven in a smaller number of tests than traditional unit tests. Of course you still need a couple of integration tests but these need only prove that the integration is working, rather than all of the paths through the logic.

## Testing message handlers

The main requirement when testing a service bus message handler is to be able to create a `ServiceBusReceivedMessage`. This class doesn't have a public constructor, but there is `ServiceBusModelFactory` class with a `ServiceBusReceivedMessage()` method especially for the purpose! This needs to be at the heart of our test approach.

When we create the artificial `ServiceBusReceivedMessage` it needs to be packaged up in a `ProcessMessageEventArgs` however, if we want to spy on what happens to the message, we can inherit from this class and override `CompleteMessageAsync`, `DeadLetterMessageAsync` and `AbandonMessageAsync` methods and set flags accordingly. We called this class `TestableProcessMessageEventArgs`.

### Here is our `TestableProcessMessageEventArgs` class

```CSharp
public class TestableProcessMessageEventArgs : ProcessMessageEventArgs
{
    public bool WasCompleted{ get; private set; }
    public bool WasDeadLettered{ get; private set; }
    public DateTime Created{ get; }

    public TestableProcessMessageEventArgs(ServiceBusReceivedMessage message) : base(message, null, CancellationToken.None)
    {
        Created = DateTime.Now;
    }

    public override Task CompleteMessageAsync(ServiceBusReceivedMessage message,
        CancellationToken cancellationToken = new CancellationToken())
    {
        WasCompleted = true;
        return Task.CompletedTask;
    }

    public override Task DeadLetterMessageAsync(ServiceBusReceivedMessage message, string deadLetterReason,
        string deadLetterErrorDescription = null, CancellationToken cancellationToken = new())
    {
        WasDeadLettered = true;
        return Task.CompletedTask;
    }

    public override Task AbandonMessageAsync(ServiceBusReceivedMessage message, IDictionary<string, object> propertiesToModify = null,
        CancellationToken cancellationToken = new CancellationToken())
    {
        return Task.CompletedTask;
    }
}
```

The next thing we need to mock is the `ServiceBusProcessor` class, this is returned by the `ServiceBusClient` and communicates with the service bus, passing messages to the handler for a specific queue or topic. We could try to use Moq but its easier to inherit from `ServiceBusProcessor` and override its `StartProcessingAsync()` method, this stops it from realising that it isn't connected to a real service bus. We called this class `TestableServiceBusProcessor`.

We can then call `base.OnProcessMessageAsync(args)` on the `TestableServiceBusProcessor` passing in a an artificial message created using the 
`ServiceBusModelFactory.ServiceBusReceivedMessage()` mentioned above. This will present the artificial message to the handler, as if it were connected to a real service bus receiving a real message.

We can add a couple of helpful features to the `TestableServiceBusProcessor` such as a collection of `TestableProcessMessageEventArgs` called `MessageDeliveryAttempts` to assert against and a generic method for sending messages, we can also simulate retries in order to test sad paths and/or exponential back off policies etc.

### Here is our `TestableServiceBusProcessor` class

```CSharp
public class TestableServiceBusProcessor : ServiceBusProcessor
{
    public List<TestableProcessMessageEventArgs> MessageDeliveryAttempts = new();

    public async Task SendMessageWithRetries<T>(T payload, int maxDeliveryCount = 5)
    {
        for (var attempt = 1; attempt <= maxDeliveryCount; attempt++)
        {
            // Don't send retry if the message was sent already and completed
            if (MessageDeliveryAttempts.Any() && MessageDeliveryAttempts.Last().WasCompleted)
                return;

            await SendMessage(payload, attempt);

            // Simulate the message being deadlettered if max delivery count is hit
            if (attempt == maxDeliveryCount)
                MessageDeliveryAttempts.Last().WasDeadLettered = true;

        }
    }

    public async Task SendMessage<T>(T payload, int attempt = 1)
    {
        var args = CreateMessageArgs(payload, attempt);
        MessageDeliveryAttempts.Add((TestableProcessMessageEventArgs)args);
        await base.OnProcessMessageAsync(args);
    }

    public ProcessMessageEventArgs CreateMessageArgs<T>(T payload, int deliveryCount = 1)
    {
        var payloadJson = JsonSerializer.Serialize(payload);

        var message = ServiceBusModelFactory.ServiceBusReceivedMessage(
            body: BinaryData.FromString(payloadJson),
            deliveryCount: deliveryCount);

        var args = new TestableProcessMessageEventArgs(message);

        return args;
    }

    public override async Task StartProcessingAsync(CancellationToken cancellationToken = default)
    {
    }
}
```

## How to use them

So, to use these classes in a component test, we can use the excellent `TestHost` from the `Microsoft.AspNetCore.TestHost` package, this enables us to run a service in memory, calling the `ConfigureServices` method as normal, but then overriding some of the configurations i.e. the ones that touch the network that we need to mock. Consider the following code:

```CSharp
public class TestHost : IDisposable
{
    private readonly IHost _server;
    public Mock<ServiceBusSender> MockTestQueue2Sender { get; } = new();
    public TestableServiceBusProcessor TestableQueue1MessageProcessor { get; } = new();
    public Mock<IWidget> MockWidgetService { get; } = new();
    public IServiceProvider Services => _server.Services;
    public int NumberOfSimulatedServiceBusMessageRetries = 5;
    public int InitialRetryDelayForServiceBusMessageRetriesInMs = 100;

    public TestHost()
    {
        var builder = new HostBuilder()
            .ConfigureWebHost(webHost =>
            {
                webHost.UseTestServer()
                    .ConfigureServices((context, services) => Program.ConfigureHost(services, context.Configuration))
                    .ConfigureTestServices((services) =>
                    {
                        var client = new Mock<ServiceBusClient>();

                        client.Setup(t => t.CreateSender(It.Is < string > (s => s == "testQueue2")))
                            .Returns(MockTestQueue2Sender.Object);

                        client.Setup(t => t.CreateProcessor(It.Is<string>(s => s == $"testQueue1")))
                            .Returns(TestableQueue1MessageProcessor);

                        services.AddSingleton(client.Object);

                        // register other test services here...
                        services.AddSingleton(MockWidgetService.Object);

                    }).Configure(app => { });
            });

        _server = builder.Build();
        _server.StartAsync();
    }

    public void Dispose()
    {
        _server?.StopAsync().GetAwaiter().GetResult();
        _server?.Dispose();
    }
}
```

So we're using Moq to create a Mock `ServiceBusClient`, this will be registered into the DI container and when it's `CreateProcessor()` method is called with the correct queue name, an instance of our `TestableServiceBusProcessor` will be returned. We are also setting up a Mock `ServiceBusSender` which we can use later to Assert whether messages were sent to a different queue.
Finally we have a Mock `IWidget` service, which our message handler calls and we can use to throw exceptions or simulate processing delays etc.

![Sequence diagram](/assets/images//testing_azure_service_bus_message_handlers_using_azure_messaging_servicebus_image1.png)

## The actual tests

The following code shows the actual tests, we decided to split the setup, execution and assertion into a set of helpers named `Given`, `When` and `Then`, this allows for a nice fluent interface for the tests and makes them nice and readable.

```CSharp
[Fact]
public void a_message_sent_to_the_queue_is_handled_and_completed()
{
    using var host = new TestHost();

    Given.OnThe(host)
        .ATestPayloadIsGenerated(out var testPayload);

    When.OnThe(host)
        .AMessageIsSentToTestQueue1(testPayload);

    Then.OnThe(host)
        .TheWidgetServiceWasCalled(times: 1)
        .And().AMessageWasSentToTestQueue2()
        .And().TheMessageWasCompleted();
}

[Fact]
public void a_message_sent_to_the_queue_is_handled_and_when_permanentException_is_thrown_is_dead_lettered()
{
    using var host = new TestHost();

    Given.OnThe(host)
        .ATestPayloadIsGenerated(out var testPayload)
        .And().TheWidgetWillThrowAPermanentException();

    When.OnThe(host)
        .AMessageIsSentToTestQueue1(testPayload);

    Then.OnThe(host)
        .AMessageWasSentToTestQueue2(times: 0)
        .And().TheMessageWasDeadLettered();
}

[Fact]
public void a_message_sent_to_the_queue_is_handled_and_when_transientException_is_thrown_is_retried_and_is_eventually_completed()
{
    using var host = new TestHost();

    Given.OnThe(host)
        .ATestPayloadIsGenerated(out var testPayload)
        .And().TheWidgetWillThrowANumberOfTransientExceptions(times:3);

    When.OnThe(host)
        .AMessageIsSentToTestQueue1(testPayload, simulateRetries: true);

    Then.OnThe(host)
        .TheMessageWasRetried(times:4)
        .And().TheWidgetServiceWasCalled(times:4)
        .And().TheRetriedMessagesHadIncreasingDelays()
        .And().AMessageWasSentToTestQueue2(times:1)
        .And().TheMessageWasCompleted();
}
```

## Given, When and Then test helpers

Here is the code for the `When` class which simulates the message appearing on the queue

```CSharp
public class When
{
    private readonly TestHost _host;

    public When(TestHost host)
    {
        _host = host;
    }

    public static When OnThe(TestHost host) => new(host);
    public When And() => this;

    public When AMessageIsSentToTestQueue1(string testPayload, bool simulateRetries = false)
    {
        var payload = new TestQueueMessage(testPayload);

        if (simulateRetries)
            _host.TestableQueue1MessageProcessor.SendMessageWithRetries(payload, _host.NumberOfSimulatedServiceBusMessageRetries).GetAwaiter().GetResult();
        else
            _host.TestableQueue1MessageProcessor.SendMessage(payload).GetAwaiter().GetResult();

        return this;
    }
}
```
And here is the code for the `Then` class which demonstrates how to use the `TestableServiceBusProcessor` and its collection of `MessageDeliveryAttempts`, each of which contains a `TestableProcessMessageEventArgs` that be can used to make assertions. We are using the `FluentAssertions` package.

```CSharp
public class Then
{
    private readonly TestHost _host;

    public Then(TestHost host)
    {
        _host = host;
    }

    public static Then OnThe(TestHost host) => new(host);
    public Then And() => this;

    public Then TheMessageWasCompleted()
    {
        _host.TestableQueue1MessageProcessor.MessageDeliveryAttempts.Last().WasCompleted.Should().BeTrue();
        return this;
    }

    public Then TheMessageWasDeadLettered()
    {
        _host.TestableQueue1MessageProcessor.MessageDeliveryAttempts.Last().WasDeadLettered.Should().BeTrue();
        return this;
    }

    public Then TheMessageWasRetried(int times)
    {
        _host.TestableQueue1MessageProcessor.MessageDeliveryAttempts.Count.Should().Be(times);
        return this;
    }

    public Then TheWidgetServiceWasCalled(int times) 
    {
        _host.MockWidgetService.Verify(x => x.DoSomething(It.IsAny<string>()), Times.Exactly(times));
        return this;
    }

    public Then AMessageWasSentToTestQueue2(int times = 1)
    {
        _host.MockTestQueue2Sender.Verify(x => x.SendMessageAsync(It.IsAny<ServiceBusMessage>(), It.IsAny<CancellationToken>()), Times.Exactly(times));
        return this;
    }

    public Then TheRetriedMessagesHadIncreasingDelays()
    {
        var delays = new List<TimeSpan>();
        for (var i = 0; i < _host.TestableQueue1MessageProcessor.MessageDeliveryAttempts.Count - 1; i++)
        {
            delays.Add(_host.TestableQueue1MessageProcessor.MessageDeliveryAttempts[i + 1].Created -
                        _host.TestableQueue1MessageProcessor.MessageDeliveryAttempts[i].Created);
        }

        for (var i = 0; i < delays.Count - 1; i++)
        {
            Assert.True(delays[i + 1] > delays[i], $"the interval of retry{i+1}({delays[i+1].TotalMilliseconds}ms) was not larger than the previous retry({delays[i].TotalMilliseconds}ms)");
        }

        return this;
    }
}
```

And that is how we are testing message handling code which uses the `Azure.Messaging.Servicebus` client library. Thankyou for reading and hope you find it useful.

All of the code from this post can be found [here](https://github.com/andrewjpoole/azure-messaging-servicebus-handler-tests)

The title image is from [here](https://rarehistoricalphotos.com/double-decker-buses-tilt-testing-1933/)