---
layout: post
title: CSharp Sql Tests
image: /images/csharpsqltests.jpg
published: false
---

TL/DR You know you should be testing stored procedures/complex queries etc, if you're underwhelmed by the T-Sql based frameworks available you might like to use a nice fluent C# framework which makes it easy to run tests against a temporary localDb instance, with a DacPac deployed, in a handy SqlTransaction per test and using markdown table syntax to define data! If so read on...

## Background

At work we are currently perfering Dapper to Entity Framework, this means we are writing more stored procedures and naturally want to cover those stored procedures with tests. At first we tried using xUnit tests, which worked perfectly well but the setup and teardown of test data proved a little cumbersome. We then tried switching approach to the TSqlt framework, which again worked perfectly well and had the added advantage of running against a temporary db instance with a DacPac project deployed and running each test in its own Sql transaction. However as developers spoilt by the lovely syntax of modern languages like C#, we found the TSqlt tests quite unpleasant to both read and write!

I thought that there must be a better way which combined the best bits of both worlds and this Sql test framework is what I came up with. Hopefully it will be useful to other people too.

## In a nutshell

In a nutshell the framework allows tests to be written in C# using familiar test frameworks (XUnit in the examples), against a temporary instance of localDb, where a DacPac has optionally been deployed, each test given a SqlConnection and a SqlTransaction (which will be rolled back afterwards), with a nice fluent api and an easy way to define tabular data either for test data setup and/or assertions.

A test looks like this:

```csharp
[Fact]
public void test_asserting_using_contains()
{
    _context.RunTest((connection, transaction) =>
    {
        var order = @"
        | Id | Customers_Id | DateCreated | DateFulfilled  | DatePaid | ProductName | Quantity | QuotedPrice | Notes       |
        | -- | ------------ | ----------- | -------------- | -------- | ----------- | -------- | ----------- | ----------- |
        | 23 | 1            | 2021/07/21  | 2021/08/02     | null     | Apples      | 21       | 5.29        | emptyString |";

        Given.UsingThe(_context)
            .TheFollowingSqlStatementIsExecuted("ALTER TABLE Orders DROP CONSTRAINT FK_Orders_Customers;")
            .And().TheFollowingDataExistsInTheTable("Orders", order);

        When.UsingThe(_context)
            .TheStoredProcedureIsExecutedWithReader("spFetchOrderById", ("OrderId", 23));

        Then.UsingThe(_context)
            .TheReaderQueryResultsShouldContain(@"| Id |
                                                  | -- |
                                                  | 23 |");

    });
}
```

## Leveraging existing work

The `LocalDbTestContext` class is responsible for setting everything up ready for a set of tests to be executed. For managing the temporary LocalDb instance, I am using the excellent `MartinCostello.SqlLocalDb` package which makes this task relatively trivial and can be found [here on github](https://github.com/martincostello/sqllocaldb) and [here on Nuget](https://www.nuget.org/packages/MartinCostello.SqlLocalDb/)

For the DacPac deployment I took inspiration from [this StackOverflow thread](https://stackoverflow.com/questions/43365451/improve-the-performance-of-dacpac-deployment-using-c-sharp)

## `LocalDbTestContext.RunTest()` method

Once the localDb instance is running and we have optionally deployed the Dacpac project, we can use the `RunTest()` method, passing in an `Action<IDbConnection, IDbTransaction>` which is the test to be executed.

The action is executed in the context of a new `SqlTransaction` and wrapped in a try finally block, which is used to tidy up any open `DataReader`s and roll back the test's individual `SqlTransaction` this ensures that the tests cannot affect eachother.

```csharp
public LocalDbTestContext RunTest(Action<IDbConnection, IDbTransaction> useConnection)
{
    try
    {
        SqlTransaction = SqlConnection.BeginTransaction();
        useConnection(SqlConnection, SqlTransaction);
    }            
    finally 
    {
        if(LastQueryResult is IDataReader lastQueryResultAsReader)
            lastQueryResultAsReader.Close(); // close any open datareaders as they are against the connection and will stuff up other tests

        SqlTransaction?.Rollback(); // leave the context untouched for the next test
    }

    return this;
}

```

## Some nice extra features

### `TabularData` class for human readable data definition

We are used to defining tabular data in Markdown tables and also Specflow's example tables, data expressed in this format is far easier for a human to 'parse' than Sql statements. So I created a class called `TabularData` which has methods for converting to and form markdown table strings and also converting to Sql statements and from `SqlDataReader`, it also has methods for evaluating whether two `TabularData` are equal and whether one contains another. The code can be found [here](https://github.com/andrewjpoole/CSharpSqlTests/blob/main/CSharpSqlTests/TabularData.cs) The example test above demonstrates its use.

### `Given`, `When` and `Then` helper clases

Recently I have started separating out the arrange, act and assert parts of a test into `Given`, `When` and `Then` classes, this gives a nice fluent interface and makes the tests nice a readable. You can write whatever C# code you like in the test and you could use the connection directly with `System.Data` like I'm doing here or via Dapper or EF etc this framework is completely un-opinionated.

The `Given` class contains methods for executing Sql statements (e.g. to remove a foreign key constraint) and methods for inserting test data into a table using a markdown table strings or an instance of a `TabularData`.

The `When` class contains methods for executing Stored Procedures which return a single value, Stored Procedures that return an `IDataReader`, Scalar queries and queries which return an `IDataReader`. The query results are stored on the context so that the `Then` class can neatly access them but there are also overloads which return the query result an Out argument.

The `Then` class has methods for asserting that query results are equal to or contain data specified using a markdown table strings or an instance of a `TabularData` passed in as an qrgument.

Here is the code for the `Given` class showing the use of `System.Data` types to utilise the connection, with some parts removed for brevity

```csharp
public class Given
{
    private readonly ILocalDbTestContext _context;
    private readonly Action<string>? _logAction;

    public Given(ILocalDbTestContext context, Action<string>? logAction = null)
    {
        _context = context;
        _logAction = logAction;
    }

    public static Given UsingThe(LocalDbTestContext context, Action<string>? logAction = null) => new(context, logAction);

    public Given And() => this;

    private void LogMessage(string message)
    {
        _logAction?.Invoke(message);
    }

    public Given TheDacpacIsDeployed(string dacpacProjectName = "")
    {
        _context.DeployDacpac(dacpacProjectName);

        return this;
    }

    public Given TheFollowingDataExistsInTheTable(string tableName, string markdownTableString)
    {
        var tabularData = TabularData.FromMarkdownTableString(markdownTableString);
        return TheFollowingDataExistsInTheTable(tableName, tabularData);
    }

    public Given TheFollowingDataExistsInTheTable(string tableName, TabularData tabularData)
    {
        try
        {
            var cmd = _context.SqlConnection.CreateCommand();
            cmd.CommandText = tabularData.ToSqlString(tableName);
            cmd.CommandType = CommandType.Text;
            cmd.Transaction = _context.SqlTransaction;

            _context.LastQueryResult = cmd.ExecuteNonQuery();

            LogMessage("TheFollowingDataExistsInTheTable executed successfully");

            return this;
        }
        catch (Exception ex)
        {
            LogMessage($"Exception thrown while executing TheFollowingDataExistsInTheTable, {ex}");
            throw;
        }
    }
    // some methods removed for brevity    
}
```

## Performance and efficiency

Starting a temporary localDb and deploying a DacPac are both quite expensive tasks, so its best to do these jobs once for a set of tests, various test frameworks achieve this in different ways, in XUnit its the `IClassFixture<T>`. A test class that implements this interface will have an instance of `T` injected into it's constructor and the `T` will disposed after any tests have been run.

Here is the class that I am injecting using `IClassFixture` which creates the localDb instance and deploys the dacpac.

```csharp
public class LocalDbContextFixture : IDisposable
{
    public LocalDbTestContext Context;

    public LocalDbContextFixture(IMessageSink sink)
    {
        Context = new LocalDbTestContext("SampleDb", log => sink.OnMessage(new DiagnosticMessage(log)));
        Context.DeployDacpac();
    }       

    public void Dispose()
    {
        Context.TearDown();
    }
}
```

In my experience the localDb takes around 10 seconds to be created and the dacpac takes another 10 seconds, but this is roughly comparable to the setup time that an equivalent T-Sqlt test would take.

So thats it, a lightweight framework which sets up your db instance and then hopefully gets out of the way, letting you test your db objects however you like but providing a nice human readable way to specify tabular data for setups and assertions. Hope you find it useful and thanks for reading!