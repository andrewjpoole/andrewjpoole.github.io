---
layout: post
title: Orchestration Solution Architecture
image: /images/chain-shiny-portrait.jpg
published: false
---

# Orchestration Solution Architecture

## TD/DR:

Take DDD / clean / Onion architecture a step further by locating multiple related flows together in one c# class. Using async method chaining, express orchestration flows in way which is both easy to read and navigate. Consider the following code which describes what should happen when an API request is recieved, ending in a call to a service _AND_ the subsequent flow which describes what should happen when the resulting service bus message is recieved back from that service!

```csharp
public class XyzPaymentOrchestrator : IZxyPaymentHandler, IEventHandler<ScreeningPassedEvent>
{
    // Constructor and fields removed for brevity.

    public async Task<OneOf<Success, Failure>> HandleApiPostRequest(XyzPaymentRequest request, IXyzPaymentRequestValidator validator)
    {
        logger.LogInformation("Post Request received");

        return await XyzPaymentDetails.FromPostRequest(request)
            .Then(validator.ValidateRequest)
            .Then(paymentPersistenceService.InsertOrFetch)
            .Then(transactionTasks.CreateDebitTransaction)
            .Then(transactionTasks.CreateCreditTransaction, onFailure: orchestratorTasks.CancelDebitTransaction)
            .Then(screeningTasks.RequestScreening)
            .ToResult();
    }

    public async Task HandleEvent(ScreeningPassedEvent screeningPassedEvent)
    {
        logger.LogInformation("ScreeningPassedEvent received");

        var result = await XyzPaymentDetails.FromScreeningEvent(screeningPassedEvent)
            .Then(paymentPersistenceService.Fetch)
            .Then(transactionTasks.SettleDebitTransaction)
            .Then(transactionTasks.SettleCreditTransaction)
            .Then(outboxService.CompletePayment);
    }    
}
```
If you like the look of that, read on!

## Async method Chaining

This concept is both enabled by and a natural evolution of my previous blog post, which explains how to use some fairly straight forward extension methods to enable async methods to be chained together with a single return path. Once we started using that method for API handlers, we soon wondered if the same could be used for service bus message handlers, then we wondered if the flows could be combined...

## DDD Solution Architectures

We roughly follow a DDD / clean solution architecture. There are many explanations and great examples of this all over the internet, with the famous onion diagram. 

At the centre is the domain layer which contains enterprise level Entities and ValueObjects which are not expected to change very much. 

Around the domain is the Application layer which contains business logic for our component's use-cases.

Around this is the Infrastructure layer which contains client code for external API's, implementations of service interfaces defined in Application, code which connects to databases, service bus, configuration, telemetry etc.

Here is a diagram depicting various projects in a solution and the flow of dependencies to achieve the above.

diagram here...

## Orchestration

Consider a scenario where we receive an API request, perform some tasks and then call another API which does some async work resulting in a message on one or more service bus queues. Maybe in our contrived WeatherReport scenario, we need to send the generated weather report to be modelled by some other system. We can now create something like the following:
```csharp
public class WeatherReportOrchestrator: IWeatherReportOrchestrator,
            EventHandler<WeatherReportModellingSucceededEvent>, EventHandler<WeatherReportModellingFailedEvent>
{
    public async Task<OneOf<WeatherReport, Failure>> HandleApiPostRequest(string requestedRegion, DateTime requestedDate)
    {
        return await WeatherReportDetails.FromRequest(requestedRegion, requestedDate)
            .Then(regionValidator.ValidateRegion)
            .Then(dateChecker.CheckDate)
            .Then(CheckCache)
            .Then(weatherForecastGenerator.Generate)
            .Then(reportPersistenceService.Save)
            .Then(weatherReportModellingClient.RequestModelling())
    }

    public async Task HandleEvent(WeatherReportModellingSucceededEvent modellingSucceededEvent)
    {
        logger.LogModellingSucceededEventReceived();

        var result = await WeatherReportDetails.FromModellingEvent(modellingSucceededEvent)
            .Then(reportPersistenceService.Fetch)
            .Then(reportPersistenceService.UpdateStatusModellingSucceeded)
            .Then(weatherReportPublisher.Publish);

        if (result.IsT1)
            throw new Exception($"Something went wrong while handling {nameof(WeatherReportModellingSucceededEvent)}");
    }

    public async Task HandleEvent(WeatherReportModellingFailedEvent modellingFailedEvent)
    {
        logger.LogModellingFailedEventReceived();

        var result = await WeatherReportDetails.FromModellingEvent(modellingFailedEvent)
            .Then(reportPersistenceService.Fetch)
            .Then(details => reportPersistenceService.UpdateStatusModellingFailed(details, modellingFailedEvent.GetFailureReason()));

        if (result.IsT1)
            throw new Exception($"Something went wrong while handling {nameof(WeatherReportModellingSucceededEvent)}");
    }
}
```

### Steps / rules

Use `Failure : OneOfBase<ApiFailure, MessageHandlerFailure>` etc

Details / routing slip class, success using a record. Shared or one per flow?

### Organisation

orchestrator per vertical slice or usecase or version etc

## Pros and Cons

### Pros

* Really easy to see whats going on and across synchonous and asynchonous flows i.e. synchronous API handler => asynchronous message bus handler etc.
* Really easy to navigate around the code `[ctrl]+[f12]` straight into any chained method.
* Really easy to debug by placing breakpoints inside any chained method, or step through the `Then` extension method for the thourough version!
* Ability to split a flow e.g. in two in order to scale out a component identified as a bottleneck.

### Cons

* Orchestrators could become a large collection of dependencies, like controllers used to be.
* Care needed not to force un-needed dependencies on various components sharing an orchestrator, e.g. API validation registration/config now required by a message handling background worker.
