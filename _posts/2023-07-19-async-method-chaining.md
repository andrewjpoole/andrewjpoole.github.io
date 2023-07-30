---
layout: post
title: Async method chaining in C#
image: /images/chain-shiny-portrait.jpg
published: true
---

# Async Method Chaining in C#

## TL/DR

We recently had a large amount of fairly procedural logic performing multiple tasks in an API request handler, we wanted a way to split it up, but maintaining a really obvious flow and ease of code navigation. This is what we came up with applied to a contrived WeatherReport API:
```csharp
public async Task<OneOf<WeatherReport, Failure>> Handle(string requestedRegion, DateTime requestedDate)
{
    return await WeatherReport.Create(requestedRegion, requestedDate)
        .Then(regionValidator.ValidateRegion)
        .Then(dateChecker.CheckDate)
        .Then(CheckCache)
        .Then(weatherForecastGenerator.Generate);
}
```
This code is wired-up directly a minimal API, where the `OneOf<WeatherReport, Failure>` result is converted to an HttpResponse and returned. These are async methods, chained together with a single return path. If you like the look of that, read on!

## Background

### No exceptions

So in the API I mentioned earlier we are already using minimal API and the `FromServices` Attribute (`[FromServices]IXyzRequestHandler handler`) to move all processing from the API project to the Application project. This keeps the API project nice and thin.

For the return type of the `Handle()` method we use the excellent [OneOf library](https://github.com/mcintyre321/OneOf), which gives us discriminated unions in C#! In this case the return type is a `OneOf<Sucess, Failure>` where `Success` just represents success and `Failure` is a `OneOfBase` representing 3 or 4 specific Failure scenarios, these get mapped to various HttpResponses in the API. This means the method signature tells us everything we need to know about all possible outcomes and we are not resorting to using Exceptions to communicate non-exceptional behaviour. For instance, bad user input is not actually exceptional, its planned for and this approach allows us to define an `InvalidRequestFailure` which can be returned and will then be converted into a `400 BadRequest` HttpResponse. 

Also because we may be interacting with external dependencies, the Handle method must be async, so the actual return type is `Task<OneOf<Sucess, Failure>>`

### Reducing Cognitive Load

Initially, we considered using MediatR to refactor our logic, but it seemed overkill and looking through another repo where we were already using MediatR, it seemed quite hard to navigate. Each Request/Command Handler had to map its result into a new (but almost identical) Command/Request object with another Handler. So to figure out the path through the code, a fair amount of searching was nessecary.  

A long while ago I did some work with Windows Workflow Foundation, which I really enjoyed. Code would be organised into functional chunks called Activities, each having In arguments, Out arguments and sometimes InOut arguments. The Activities would be arranged in a visual workflow, so the flow and ordering was really obvious. The striking thing about it, was that new team members could instantly find their way around and once inside an Activity it was immediately obvious what the scope of the task was, what inputs were available to use and what outputs would need to be set. 

I wanted to find a way to replicate this declarative flow and a flavour of the functional isolation, because I want engineers to easily see whats going on and find their way around. Our final solution shows the flow declaratively in once place and `[ctrl]+[f12]` takes you straight to the code in Visual Studio!

## How its used

Here is the full version of the contrived WeatherReportRequestHandler example shown above:
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
A static method named `Create()` returns an initialised `WeatherReport` object, but it actually returns a `Task<oneOf<WeatherReport, Failure>>` we'll see why in a moment. The `WeatherReport` object contains any required state and crucially represents Success when returned.

To be chained, a method need only have the correct return type, in this case a `Task<OneOf<WeatherReport, Failure>>`. The chained method will probably recieve a `WeatherReport` as an argument so it can read some state, perform its task, possibly set some resulting state and then crucially it should return the `WeatherReport` object if sucessful _or_ a derrivative of `Failure` if not. 

Note the methods can live anywhere, they might be local or live on services injected from IoC, they just need the correct signature.

The chaining is possible because of an extension method named `Then` (shown below) which extends a `Task<oneOf<T, TFailure>>` in the case above the `T` is the `WeatherReport` and `TFailure` is the `Failure` class mentioned above. This is why the `Create()` method and all of the other methods in the chain must return `Task<oneOf<WeatherReport, Failure>>`. The compiler infers what the Types `T` and `TFailure` are from the first method in the chain.

In this version you can see the `WeatherReport` variable named `report` being explicitly passed into each chained method, if the chained methods only have a single `T` argument then you can use the C# feature method group conversion to delegate, making the code even more terse and easily readable: 
```csharp
.Then(report => regionValidator.ValidateRegion(report))
// becomes
.Then(regionValidator.ValidateRegion)
```
This is shown in the TLDR section at the top.

As with any lambda expressions in C#, the func containing the chained method can capture any outer variables in scope e.g. 
```csharp
var validationSettings = new ValidationSettings{...}
...
.Then(report => regionValidator.ValidateRegion(report, validationSettings))
```

Out variables can also be used to return state from a chained method and passed into subsequent chained methods or consumed in the method containing the chain.

We have had success using a record for the `T` where some properties are set initially in our `Create()` method and other nullable properties start life as null, but are later set while creating a new version of the record using the `with` keyword.

## How it works

Here is the simplest version of the extension method `Then`:
```csharp
public static async Task<OneOf<T, TFailure>> Then<T, TFailure>(
  this Task<OneOf<T, TFailure>> currentJobResult,
  Func<T, Task<OneOf<T, TFailure>>> nextJob)
{
    // Inspect result of (probably already awaited) currentJobResult, if its a TFailure return it...
    var TOrFailure = await currentJobResult;
    if (TOrFailure.IsT1)
        return TOrFailure;

    // ...Otherwise do your next job
    var currentT = TOrFailure.AsT0;
    return await nextJob(currentT);
}
```
So the first job is to inspect the result of the previous job in the chain, we await it, but are pretty sure that its already been awaited and therefore the Result is already available. Note this is fine because we are using the full blown `Task`, if it were `ValueTask` we would not be able to await a second time. The OneOf library has methods for us to determine if if the Result contains the `T` (`WeatherReport` in this case) or `TFailure` (a `Failure` in this case). `IsT0()` will return true if the object contains the first possiblity in the discriminated union, `IsT1()` will return true for the second item etc. 

If the result contains a `TFailure` from eariler in the chain we _immediately_ return that `Failure` _without_ performing our Task, all of the following `Then's` in the chain do the same. This means the first `Failure` returned by a chained task is returned and no more chained tasks are executed.

Otherwise if the Result was sucessful, which means it contained a `T` (`WeatherReport` in this case) then we should call and await the `Func` `nextJob` passing in the `T` (`Weatherreport` in this case). This is how the chain is continued.

### What if something goes wrong?

We had a case where we were creating two transactions, one after the other in the chain, but if the first succeeded and the second failed, we needed a way to cancel the first transaction. Initally this code was hidden in the second transaction creation method, but a team member pointed out that was counter-intuitive. To solve this problem a second overload of the `Then` method is provided which takes a second Func called `onFailure` as an argument. We inspect the Result of `nextJob` in the same way and it it fails, we invoke and await `onFailure` before returning the `TFailure` as before.

Here is the overload of the extension method `Then` with `onFailure`:
```csharp
public static async Task<OneOf<T, TFailure>> Then<T, TFailure>(
  this Task<OneOf<T, TFailure>> currentJobResult,
  Func<T, Task<OneOf<T, TFailure>>> nextJob,
  Func<T, TFailure, Task<TFailure>> onFailure)
{
    // Inspect result of (probably already awaited) currentJobResult, if its a TFailure return it...
        var TOrFailure = await currentJobResult;
        if (TOrFailure.IsT1)
            return TOrFailure;

        // ...Otherwise do your next job ... 
        var currentT = TOrFailure.AsT0;
        var result = await nextJob(currentT);

        // If next job returned a T (Success) return it...
        if (result.IsT0)
            return result;        
        
        // Otherwise invoke onFailure and return the final resulting TFailure...
        var finalFailure = await onFailure(currentT, result.AsT1);
        return finalFailure;
}
```
It can be used like this:
```csharp
return await PaymentDetails.Create(request)
        .Then(d => validator.ValidateRequest(d))
        .Then(d => paymentRepository.Save(d))
        .Then(d => transactionService.CreateDebitTransaction(d))
        .Then(d => transactionService.CreateCreditTransaction(d), onFailure: d, f => transactionService.CancelDebitTransaction(d, f))
        .Then(d => eventDispatcher.Dispatch(d));
```

We also have a method named `IfThen()` which takes an additional `func<T, bool>` named `condition` so that the `nextJob` will only be invoked if `condition` evaluates to True; I'm sure there are other variations which will be added. 

## Conclusion

So if you also have multiple tasks to perform off the back of an API request or when handling a Service Bus message etc, maybe consider this method of chaining methods together. Maintain a single return path, place the methods wherever they best fit and make life easy for your team by expressing declaratively the flow of tasks with super-easy code navigation. 

This code is avilable in a [Nuget package]() and in this [GitHib repository](https://github.com/andrewjpoole/OneOf.Chaining), but its basically a few small extension methods, so copy/paste away!

Hope you found this interesting, thanks for reading :)
