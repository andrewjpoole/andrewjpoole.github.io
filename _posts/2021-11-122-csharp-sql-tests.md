---
layout: post
title: CSharp Sql Tests
published: true
---

# C# Sql Tests

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

Recently I have started separating out the arrange, act and assert parts of a test into `Given`, `When` and `Then` classes, this gives a nice fluent interface and makes the tests nice a readable.

Here is the code for the 

## Performance and efficiency

Starting a temporary localDb and deploying a DacPac are both jobs that take a long time, so its best to do these jobs once for a set of tests, various test frameworks achieve this in different ways, in XUnit its the `IClassFixture`