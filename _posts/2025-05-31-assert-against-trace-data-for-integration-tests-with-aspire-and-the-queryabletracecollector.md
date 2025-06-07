---
layout: post
title: Assert against trace data for integration tests with .Net Aspire and the QueryableTraceCollector
image: /images/testing-ai-generated-1.png
published: true
---

# Integration Tests: How Not to Hate Them ❤️

### TL/DR;

Recently while preparing for a talk on integration tests, I tried a few things and discovered something I think is potentially game changing; asserting against open telemetry trace data for integration tests! This makes the tests less complex and less brittle, easier to run locally and will probably make your systems easier to support by temping you to add better telemetry earlier on in the dev loop! Basically a win-win-win-win...🧐

## Background

I have always disliked integration tests because they expose just how hard it can be to run a modern distributed app locally, especially in a team scenario where people have differing preferences of tooling/setup/even OS! So we often give up on running integration tests locally altogether and end up relying on the integration tests passing in an environment... after a release... after a CI build... after we have checked in our change and hoped for the best! This is not an ideal feedback loop and could easily take 30mins+ ☹️😫😪😴

The other thing which annoys me about integration tests is having to assert in several ways against various different things to ensure the expected behaviour happened, E.g. using database connections to query or asb queues for assertions. Its possible of course, but think of the case of a message broker like Azure Service Bus, we need to peek at a queue to ensure first no old messages exist, then again at the right moment to assert that the expected message has appeared, then after the correct amount of time has passed we check again to see that the message has dissapeared, but we also need to check that it didn't appear in the dead letter queue! Even after that we still aren't certain that it was processed unless we check some other outcome like a domain event was persisted or other db update occured etc. The dance of timing here certainly plays a part in these kinds of tests being notoriously brittle. There must be a better way...

Clearly the effort required to get integration tests running locally is worthwhile and my talk set out to make that case and provide a few tips.

## Enter .Net Aspire for Exceptional Local Development Experience!

[.NET Aspire](https://github.com/dotnet/aspire) has been developed to solve the exact problem of running distributed apps locally and it was on my list to try anyway. Aspire allows us to define our services in terms of executable apps, services, configuration and the relationships between them all. This is also done in one single file - amazing! There are lots of excellent resources on what Aspire is and how to use it to orchestrate a distributed app, from here I will concentrate specifically how it can help with integration tests.

## Open telemetry

Aspire also wires up all of the resources to send [Open Telemetry](https://opentelemetry.io/) data to the dashboard where it can be viewed and filtered while the app is running locally. The value of this can not be understated. Once we start using this data to see whats happening and inspect what's gone before, it becomes addictive, we start having realisations like "if we could add that piece of data to this trace, then we'd save x minutes looking it up from 3 db queries during an incident if scenario y ever happens". To have _this_ kind of view of an app, at _this_ early a stage in the development loop is amazing! See below for code to add custom trace data to a process or to bridge a process gap.

![trace data in Aspire dashboard](/images//open-telemetry-trace-data-aspire-dashboard.png)

## Aspire Tests

Aspire also supports testing a distributed app using the same orchestration code, meaning that when an Aspire test is run, the same dependencies get started, the same service discovery and configuration happens and we get the opportunity to create clients to our services and call APIs or send messages etc. These tests take quite a while to run due to the nature of whats actually being spun up, but they are much faster than waiting for the check-in -> CI build -> release to env -> run tests loop.

## A Eureka moment!💡

So at this moment, while experimenting with Aspire tests, it dawned on me that all that lovely trace data would be absolutely perfect for asserting that expected behaviour ocurred as expected, its a forensic single-source-of-truth timeline of everything that happened since the app started!

Alas I also realised a few things:

There are some differences between Aspire tests the Aspire localdev 'F5' experience, one being that the dashboard doesn't run during tests. The dashboard is the component that collects open telemetry data, all other orchestrated services are configured to send their telemetry data to the dashboard via gRPC. If the dashboard doesn't run during tests, then nothing is listening to collect the trace data☹️

Now it is possible to configure that dashboard to run during tests, but there are no public endpoints available with which to query the data. Again it is possible to use reflection to get access to the private view model, but I was unable to get that to work.

## Asserting against trace data!🚀

So, thinking through the requirements:
- We need something that all of the resources can send data to, and given each resource runs in its own process, that likely means a new resource
- Various official OTEL backends exist, but they all seemed to be quite heavywieght for this use-case
- We need an easy way to assert against the collected trace data
- Ideally we should be able to filter just the interesting trace data, otherwise its a firehose!
- We need to be able to clear collected trace data between tests
- Ideally it should be simple to setup and in-keeping with Aspire and Aspire tests

So, after some experiementation, the simplest solution I could come up with is:
- A simple minimal API with some methods and an in-memory collection
- An implementation of `OpenTelemetry.BaseExporter<Activity>` which will send filtered trace data to the API, packaged up as an Aspire Client package
- A lite version of Activity to serialise and pass around
- A container image of the API in DockerHub
- An Aspire integration package to easily add the API as a container resource

## Filling in some trace data gaps⛳

I noticed that in a coupld of places I was missing vital info about what had happened, I was looking at this from a 'what would be ideal to assert against' point of view, but 'what would be helpful when investigating an incident' is another equally valid and useful one.

### NotificationService

In my dummy Notification Service, I did not have any trace data showing me when the notification is 'made', because being a dummy app, it was not doing anything which is automatically traced (i.e. making an API call, sending a message or calling the database), so I added the following code:

```csharp
using (var activity = Activity.StartActivity("User Notication Sent", ActivityKind.Producer))
{
    activity?.SetTag("user-notification-event.body", @event.Body);
    activity?.SetTag("user-notification-event.reference", @event.Reference);
    activity?.SetTag("user-notification-event.timestamp", @event.Timestamp.ToString("o"));

    sentNotifications.Add(@event);

    logger.LogInformation("User Notification Sent! {Reference}\nBody: {Body}\n@{Timestamp}", @event.Body, @event.Reference, @event.Timestamp);

    return;
}
```

### Outbox

Also in my Outbox implementation, because the `OutboxDispatcherHostedService` runs in a separate process, when it picks up an `OutboxItem` to dispatch, there is no trace data from the context which originally persisted the item, so I ended up serialising the telemetry context and saving it in a json string in the database so it could be rehydrated when dispatching, this means that the span persists across both phases before the `OutboxItem` is persisted and then afterwards when it is dispatched.

Code to serialise the telemetry context when persisting an individual `OutboxItem`, found in the [OutboxRepository](https://github.com/andrewjpoole/event-sourced-but-flow-driven-example/blob/main/src/WeatherApp.Infrastructure/Outbox/OutboxRepository.cs):

```csharp

private static readonly ActivitySource Activity = new(nameof(OutboxRepository));
private static readonly TextMapPropagator Propagator = Propagators.DefaultTextMapPropagator;

public async Task<long> Add(OutboxItem outboxItem, IDbTransactionWrapped? transaction = null)
{
    //... prepare update query
    using (var activity = Activity.StartActivity("Outbox Item Insertion", ActivityKind.Producer))
    {
        Dictionary<string, string> telemetryDictionary = new();
        if(activity != null)
            Propagator.Inject(new PropagationContext(activity.Context, Baggage.Current), telemetryDictionary, (carrier, key, value) => 
            {
                carrier.Add(key, value);
            });

        parameters.Add("@SerialisedTelemetry", JsonSerializer.Serialize(telemetryDictionary));

        var insertedOutboxItemId = await connection.QuerySingleOrDefault<int>(sql, parameters, transaction);

        return insertedOutboxItemId;
    }
}
```

Code to rehydrate just before sending an individual outbox item, found in the [OutboxDispatcherhostedService](https://github.com/andrewjpoole/event-sourced-but-flow-driven-example/blob/main/src/WeatherApp.Infrastructure/Outbox/OutboxDispatcherHostedService.cs):

```csharp
// Hydrate the telemetry context from item.SerialisedTelemetry...
var parentContext = Propagator.Extract(default, item.SerialisedTelemetry, (serialisedTelemetry, key) => 
{
    var telemetryDictionary = JsonSerializer.Deserialize<Dictionary<string, string>>(serialisedTelemetry);

    if (telemetryDictionary == null || telemetryDictionary.Count == 0)
        return Enumerable.Empty<string>();

    if(telemetryDictionary.TryGetValue(key, out var value))
        return new List<string> { telemetryDictionary[key] };

    return Enumerable.Empty<string>();
});
Baggage.Current = parentContext.Baggage;

using var activity = Activity.StartActivity("Dispatch Message", ActivityKind.Consumer, parentContext.ActivityContext);
await messageSender.SendAsync(item.SerialisedData, item.MessagingEntityName, cancellationToken);
await outboxRepository.AddSentStatus(OutboxSentStatusUpdate.CreateSent(item.Id));
logger.LogDispatchedOutboxItem(item.Id);
```

## Ok, how do I try it?

### Setting Up QueryableTraceCollector in your Aspire project

1. Adding QueryableTraceCollector to Your Aspire Project

add the [hosting nuget package](https://www.nuget.org/packages/AJP.Aspire.Hosting.QueryableTraceCollector) to the 
`<PackageReference Include="AJP.Aspire.Hosting.QueryableTraceCollector" Version="1.0.0" />`

```csharp
var queryableTraceCollectorApiKey = builder.Configuration["QueryableTraceCollectorApiKey"] ?? "123456789"; // I add a local secret in VSCode...
var queryabletracecollector = builder.AddQueryableTraceCollector("queryabletracecollector", queryableTraceCollectorApiKey)
    .WithExternalHttpEndpoints()
    .ExcludeFromManifest();
```
This will add a container resource which runs the API.

2. Add the [client integration nuget package](https://www.nuget.org/packages/AJP.Aspire.QueryableTraceCollector.Client/1.0.0), probably to ServiceDefaults
`<PackageReference Include="AJP.Aspire.QueryableTraceCollector.Client" Version="1.0.0" />`

Add

```csharp
public static TBuilder ConfigureOpenTelemetry<TBuilder>(this TBuilder builder) where TBuilder : IHostApplicationBuilder
{
    //... usual stuff omitted
    builder.AddOpenTelemetryExporters();

    // Filter collected trace data by DisplayName to just the bits we're interested in, these can also be wired up through config instead if you prefer...
    builder.Services.AddQueryableOtelCollectorExporter(builder.Configuration, ["Outbox Item Insertion", "User Notication Sent", "Domain Event Insertion"]); 

    return builder;
}
```

3. Assert against collected trace data in an Aspire test project

I do it with a nice framework🙂 see my test, the second one in [here](https://github.com/andrewjpoole/event-sourced-but-flow-driven-example/blob/main/tests/WeatherApp.Tests.Aspire.Integration/WeatherAppAspireIntegrationTests.cs)

```csharp
[Test]
public void PostWeatherData_EventuallyResultsIn_AUserNotificationBeingSent2()
{
    var (given, when, then) = (new Given(), new When(), new Then());

    given.WeHaveSetupTheAppHost(out var appHost)
        .And.WeRunTheAppHost(appHost, out var app, DefaultTimeout)
        .And.WeCreateAnHttpClientForTheQueryableTraceCollector(app, appHost.Configuration, out var queryableTraceCollectorClient)
        .And.WeClearAnyCollectedTraces(queryableTraceCollectorClient)
        .And.WeCreateAnHtppClientForTheAPI(app, out var apiHttpClient, DefaultTimeout)
        .And.WeHaveSomeCollectedWeatherData(out var location, out var reference, out var requestId, out var collectedWeatherData);

    when.WeWrapTheWeatherDataInAnHttpRequest(out var httpRequest, location, reference, requestId, collectedWeatherData)
        .And.WeSendTheRequest(apiHttpClient, httpRequest, out var response);

    then.TheResponseShouldBe(response, HttpStatusCode.OK);
            
    when.WeWaitWhilePollingForTheNotificationTrace(queryableTraceCollectorClient, 9, "User Notication Sent", out var traces);
        
    then.WeAssertAgainstTheTraces(traces, traces => 
    {
        traces.AssertContainsDomainEventInsertionTag("WeatherDataCollectionInitiated");
        traces.AssertContainsDomainEventInsertionTag("SubmittedToModeling");
        traces.AssertContainsDomainEventInsertionTag("ModelUpdated");
        
        traces.AssertContainsDisplayName("Outbox Item Insertion");

        traces.AssertContains(t => t.DisplayName == "User Notication Sent"
            && t.ContainsTag("user-notification-event.body", x => x == "Dear user, your data has been submitted and included in our latest model")
            && t.ContainsTag("user-notification-event.reference", x => x == reference), 
            "Didn't find the expected user notification trace with the expected tags.");            
    });        
}
```

That might seem like a large test for a blog post, but consider that it is testing an entire end-to-end flow ([described here in this readme](https://github.com/andrewjpoole/event-sourced-but-flow-driven-example/blob/main/README.md)) and then compare it to your integration tests and end-to-end tests at work, then it starts to look nice and terse! The fact that it does all of its assertions consistently against a single source of truth, without the need to lots of complex timing and waiting makes a previously complex problem trivially simple🤓 

### Bonus tip #1 - If you need to ensure that an API call is _not_ traced

You can just create a DelegatingHandler which literally nulls the current Activity. I use this to ensure that calls to QueryableTraceCollector are not themselves traced, otherwise we end up with infinaite loops, ask me how I know 🤣

```csharp
public class ClientHandlerWithTracingDisabled : DelegatingHandler
{
    public ClientHandlerWithTracingDisabled(HttpMessageHandler innerHandler) : base(innerHandler){}

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        Activity.Current = null;
        return await base.SendAsync(request, cancellationToken);
    }

    protected override HttpResponseMessage Send(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        Activity.Current = null;
        return base.Send(request, cancellationToken);
    }
}
```
### Bonus tip #2 - Customise display for demos

Recently I have been using VSCode for as much as possible and in perticular Polyglot Notebooks for demonstrations. The Http cells are great, but for querying the QueryableTraceCollector's API I have been using a CSharp cell where I can add an overriden ToString() method and customise the output using emojis like this:

```csharp
// contents of CSharp Cell in Polyglot notebook
using System.Net.Http;
using System.Text.Json;

public class TraceData
{
    public string Resource { get; set; } = string.Empty;
    public string Source { get; set; } = string.Empty;
    public string DisplayName { get; set; } = string.Empty;
    public string TraceId { get; set; } = string.Empty;
    public string SpanId { get; set; } = string.Empty;    
    public Dictionary<string, object> Tags { get; set; } = new();

    public override string ToString()
    {
        var thirdPart = string.Empty;
        if(DisplayName == "Domain Event Insertion")
            thirdPart = Tags["domain-event.eventclassName"].ToString().Split('.').LastOrDefault();

        if(DisplayName == "User Notication Sent")
            thirdPart = $"✉️ reference: {Tags["user-notification-event.reference"].ToString()}";

        if(DisplayName == "Outbox Item Insertion")
            thirdPart = $"📫 outbox id: {Tags["outbox-item.Id"].ToString()}";
                   
        return $"{Resource.PadRight(30, '-')}> {DisplayName.PadRight(30, '-')}> {thirdPart}";
    } 
        
}

var client = new HttpClient() { BaseAddress = new Uri("http://localhost:8000"), DefaultRequestHeaders = { { "X-Api-Key", "insert-api-key-here" } } };
var response = await client.GetAsync("traces");
var responseJson = await response.Content.ReadAsStringAsync();
JsonSerializer.Deserialize<TraceData[]>(responseJson, JsonSerializerOptions.Web ).Display();
```

![image](/images//queryabletracecollector-polyglotnotebook-customise-output.png)

Which makes it much easier to spot thing things that matter.

## Conclusions

Asserting against trace data with .NET Aspire and QueryableTraceCollector elevates your integration tests, not only making them easier to write but also fairly trivial to run locally _and_ less brittle😁 By helping us to move towards something like Observability-driven-development we can gain confidence that our distributed systems behave as intended—not just at the API level, but deep within its internals. Also by making the observability more useful, earlier in the loop, we end up with more supportable systems.

All of the code from this post can be found [here](https://github.com/andrewjpoole/event-sourced-but-flow-driven-example)

### Best Practices

- **Isolate tests**: If your QueryableTraceCollector persists between tests, be sure to call the DELETE /traces method to clear all trace data.
- **Use meaningful span names**: This makes querying and asserting easier.
- **Assert on what matters**: Focus on key operations, error handling, and context propagation.
