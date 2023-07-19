---
layout: post
title: Async method chaining in C#
image: /images/chain-shiny.jpg
published: false
---

# Async Method Chaining in C#

## TL/DR

We recently had a large amount of fairly procedural logic performing multiple tasks in an API request handler, we wanted a way to split it up, but maintaining a really obvious flow and ease of code navigation. This is what we came up with:
```csharp
public async Task<OneOf<WeatherReport, Failure>> Handle(string requestedRegion, DateTime requestedDate)
{
    return await WeatherReport.Create(requestedRegion, requestedDate)
        .Then(report => regionValidator.ValidateRegion(report))
        .Then(report => dateChecker.CheckDate(report))
        .Then(report => CheckCache(report))
        .Then(report => weatherForecastGenerator.Generate(report));
}
```
This code is wired-up directly a minimal API, where the `OneOf<WeatherReport, Failure>` result is converted to an HttpResponse and returned. If you like the look of that, read on!

## Background

### No exceptions

OneOf library, already using minimal API and [FromServices] Attribute to find the request handler.
Convert to Http Response at the last minute
thin API project + Application + Domain etc

### Cognitive Load

WWF, functional
MediatR - hard to navigate + lots of mapping to different but almost identical command objects

## How it works

full version of the contrieved WeatherReportRequestHandler example shown above:
```csharp
public class GetWeatherReportRequestHandler : IGetWeatherReportRequestHandler
{
    private readonly IRegionValidator regionValidator;
    private readonly IDateChecker dateChecker;
    private readonly IWeatherForecastGenerator weatherForecastGenerator;

    public GetWeatherReportRequestHandler(IRegionValidator regionValidator, IDateChecker dateChecker, IWeatherForecastGenerator weatherForecastGenerator)
    {
        this.regionValidator = regionValidator;
        this.dateChecker = dateChecker;
        this.weatherForecastGenerator = weatherForecastGenerator;
    }

    public async Task<OneOf<WeatherReport, Failure>> Handle(string requestedRegion, DateTime requestedDate)
    {
        return await WeatherReport.Create(requestedRegion, requestedDate)
            .Then(report => regionValidator.ValidateRegion(report))
            .Then(report => dateChecker.CheckDate(report))
            .Then(report => CheckCache(report))
            .Then(report => weatherForecastGenerator.Generate(report));
    }

    public async Task<OneOf<WeatherReport, Failure>> CheckCache(WeatherReport report)
    {
        // Check and populate from a cache etc...
        // Methods from anywhere can be chained as long as they have the correct signature...
        return report;
    }
}
```

### Routing Slip

### All in the method signature

### What if something goes wrong?

## Conclusion

So if you also have multiple tasks to perform off the back of an API request or when handling a Service Bus message etc, maybe consider this method of chaining methods together. Maintain a single return path, place the methods wherever they best fit and make life easy for your team by expressing declaratively the flow of tasks with super-easy code navigation. 

This code is avilable in a Nuget package, but its basically 3 extension methods, so copy/paste away!

Hope you found this interesting, thanks for reading :)
