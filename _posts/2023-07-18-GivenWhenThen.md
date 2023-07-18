# Given, When and Then helper classes for testing

## TL/DR

This post introduces a method of organising test code into shared, reusable chunks, enabling complex tests to be expressed in a terse english-like fashion, which looks like this:
``` csharp
[Fact]
public void Return_badRequest_when_request_is_invalid()
{
    Given.UsingThe(apiWebApplicationFactory)
        .TheTransactionsApiCreateEndpointWillReturn(HttpStatusCode.OK)
        .And.WeHaveAnInvalidPaymentRequest(out var invalidPaymentRequest, currency)
        .And.TheServerIsStarted();

    When.UsingThe(apiWebApplicationFactory)
        .WeWrapThePaymentRequestInHttpRequestMessage(invalidPaymentRequest, out var httpRequestMessage)
        .And.WeSendTheMessageToTheApi(httpRequestMessage, out var response);
    
    using (new AssertionScope())
    {
        Then.UsingThe(apiWebApplicationFactory)
            .TheResponseCodeShouldBe(response, HttpStatusCode.BadRequest)
            .And.TheBodyShouldNotBeEmpty(response, out var body)
            .And.TheBodyShouldContainAProblemDetails(body, details => details.Errors.Count.Should().Be(1))
            .And.APaymentRecordWasNotSaved()
            .And.NoPaymentStatusRecordsWereSaved();
    }
}
```
Hopefully you agree that while still outlining exactly what is being setup, executed and asserted against, this test is easy to understand, in fact you could probably show it to your Product Owner without requiring too much explanation.

## Background

In our team we favour using Component tests to create the majority of our code coverage, filling in with Unit tests where necessary. Then we add minimal Integration tests to prove out integrations with external services and finally end-to-end tests to test flows from start to finish. I guess this roughly aligns to the testing trophy idea as opposed to the testing triangle.

Our Component tests are like internal Integration tests, we we aim to exercise as much of the app as possible, just mocking the external dependencies at the outermost point. So for example, we use WebApplicationFactory<TEntryPoint> to test from the Program onwards, setting up Mocks and switching them in place of external service interfaces, such as database or service bus connections. The rest of the real services/components under test obviously do not know that the external dependencies are mocked. 

This kind of test provides lots of value, including a fairly realistic local debugging experience and finding issues that traditional unit testing would not be expected to find. For instance if an injected service was not registered in the IoC container or a required config value was missing, these tests would not pass.

There are downsides to this approach; there is a fair amount of code to initially set everything up and this in turn makes it hard to do TDD. The tests themselves can also become quite large unless you have a strategy for organising and sharing code across tests…

## Given, When and Then classes

In order to solve some of those problems, we have started separating out the arrange, act and assert parts of a test into `Given`, `When` and `Then` classes, every method on each of these classes returns itself, which results in a nice fluent interface making the tests shorter and improving readability. This approach encourages code sharing and consistency across tests.

### The `Given` class 

is responsible for the 'arrange' part of the test, it contains methods for setting up behaviour on the Mocks, creating payloads and/or other bits of test data. When setting up Mocked behaviours, I like to name methods using the future tense e.g. `TheTransactionsApiWillReturnTheStatusCode(HttpStatusCode.BadRequest)`

### The `When` class 

is responsible for the 'act' part of the test and typically contains methods for kicking off the process being tested, so sending a request to an API, sending a service bus message, inserting data into a database or creating events in an event store etc. 

### The `Then` class 

is responsible for the 'assert' part of the test and contains methods for making assertions. One useful trick is to have a method which takes an Action to allow the tests to do their own assertions, for example you might have a method named `TheHttpResponseContainsABody(HttpResponseMessage response, Action<string> assertAgainstBody)` this can then be used like this: `And.TheHttpResponseContainsABody(response, body => body.Should().Be("Alive!"))`

In the above example, the whole `Then` section is wrapped in a `using (new AssertionScope())` this means that also assertions will be evaluated before any failed assertions cause the test to fail, so all of the failed assertions will be reported, rather than just the first one.

### Shared stuff

All three classes share an instance of whatever owns the Mocks and useful properties or fields, lets call this the context. Each class has a static method `UsingThe()` which assigns this shared state object and returns a new `Given`, `When` or `Then`. Each class also has a dummy `And` property which just returns itself meaning the tests read more like english sentances.

### Out variables!

So no-one likes out variables, myself included, but here they offer a way of defining state inline without breaking the flow/readability of the tests, which I think really works well. It also makes the variables easy to trace through the test steps.

### Some code

Rather than pointing at a Github repo, simplified example `Given`, `When` and `Then` classes are included below to get you going.

Here is an example of a Given class with methods for setting up testing an API:
```csharp
public class Given
{
    private readonly WeatherReportApiWebApplicationFactory apiWebApplicationFactory;

    public Given(WeatherReportApiWebApplicationFactory apiWebApplicationFactory)
    {
        this.apiWebApplicationFactory = apiWebApplicationFactory;
        apiWebApplicationFactory.MockWeatherReportFetcher.Invocations.Clear();
    }

    public static Given UsingThe(WeatherReportApiWebApplicationFactory apiWebApplicationFactory) => new(apiWebApplicationFactory);
    public Given And => this;
    
    public Given TheWeatherReportFetcherApiEndpointWillReturn(HttpStatusCode statusCode, Refit.ProblemDetails? problemDetails = null)
    {
        apiWebApplicationFactory.MockWeatherReportFetcher
            .SetupRequest(HttpMethod.Get, r => r.RequestUri == someUri)
            .ReturnsResponse(statusCode, new StringContent(JsonSerializer.Serialize(problemDetails), Encoding.UTF8));

        return this;
    }    

    public Given WeHaveAWeatherReportRequestPayload(out WeatherForecastRequest requestPayload, string currency)
    {
        paymentRequest = new WeatherForecastRequest(
            // populate request here...
        );

        return this;
    }    

    public Given TheServerIsStarted()
    {
        apiWebApplicationFactory.Start();
        return this;
    }
}
```

Here is an example of a When class with methods for testing an API:
```csharp
public class When
{
    private readonly WeatherReportApiWebApplicationFactory apiWebApplicationFactory;

    public When(WeatherReportApiWebApplicationFactory apiWebApplicationFactory)
    {
        this.apiWebApplicationFactory = apiWebApplicationFactory;
    }

    public static When UsingThe(WeatherReportApiWebApplicationFactory apiWebApplicationFactory) => new(apiWebApplicationFactory);
    public When And => this;

    public When WeWrapTheRequestInHttpRequestMessage(WeatherForecastRequest requestPayload, out HttpRequestMessage httpRequest)
    {        
        httpRequest = new HttpRequestMessage(HttpMethod.Post, DomesticOrchestratorApiWebApplicationFactory.DomesticPaymentUri);
        httpRequest.Headers.Add("AdditionalRequiredHeaderName", JsonSerializer.Serialize(someValue));
        httpRequest.Content = new StringContent(JsonSerializer.Serialize(requestPayload), Encoding.UTF8, "application/json");

        return this;
    }

    public When WeSendTheMessageToTheApi(HttpRequestMessage httpRequest, out HttpResponseMessage response)
    {
        response = apiWebApplicationFactory.HttpClient.SendAsync(httpRequest).GetAwaiter().GetResult();
        return this;
    }
}
```

#### Async methods

The methods on these classes are nessecarily all ordinary synchronous methods, otherwise the await keyword would prevent the method chaining. So to execute some asynchronous process like calling an API we do use `.GetAwaiter().GetResult();` The reason I'm Ok with this is that the test code being synchronous actually simplifies things. If the test code and the code-under-test were both async, I think there would be more chance of race conditions and more complex code to prevent them. This way, if we need to wait for an async process to complete, e.g. for a service bus handler to finish handling a message, we can add a method `Then... And.WeWaitSomeTimeForProcessingToComplete(timeToWaitInMilliSeconds: 500)`


Here is an example of a Then class with methods for testing an API:
```csharp
public class Then
{
    private readonly WeatherReportApiWebApplicationFactory apiWebApplicationFactory;

    public Then(WeatherReportApiWebApplicationFactory apiWebApplicationFactory)
    {
        this.apiWebApplicationFactory = apiWebApplicationFactory;
    }

    public static Then UsingThe(WeatherReportApiWebApplicationFactory apiWebApplicationFactory) => new(apiWebApplicationFactory);
    public Then And => this;

    public Then TheResponseCodeShouldBe(HttpResponseMessage response, HttpStatusCode code)
    {
        response.Should().NotBeNull();
        response.StatusCode.Should().Be(code);
        return this;
    }

    public Then TheBodyShouldNotBeEmpty(HttpResponseMessage response, Action<string> assertAgainstBody)
    {
        var body = response.Content.ReadAsStringAsync().GetAwaiter().GetResult();
        assertAgainstBody(body);

        return this;
    }
}
```


## In conclusion

So there it is, a simple strategy for organising your test code with, so lightweight it doesn't even constitute a framework! Hope you find it helpful and thanks for taking the time to read :)
