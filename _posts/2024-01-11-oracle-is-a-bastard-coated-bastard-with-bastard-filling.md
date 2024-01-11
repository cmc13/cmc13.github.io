---
layout: post
title:  "Design Decisions in OracleConnection"
date:   2024-01-11 14:00:00
categories: programming
tags: dotnet-core c-sharp asp-net-core oracle design
---

Ran into a questionable design decision in the `Oracle.ManagedDataAccess.Client` library.

<!-- more -->

## The Problem

I had re-written some data access logic in a dashboard app a few days ago. Unit tests passing, no runtime exceptions, but users were noticing that a particular update wasn't happening anymore. Code looked something like this:

{% highlight csharp %}
public async Task UpdateOrderQuantity(int orderId, int detailId, int quantity)
{
    const string sql = """
                    UPDATE detail_table SET
                    QUANTITY = :quantity
                    WHERE order_id = :orderId AND detail_id = :detailId
                    """;
    await using var conn = _dbContextFactory.CreateConnection();

    await conn.OpenAsync();
    try
    {
        // Extension method
        await conn.ExecuteNonQueryAsync(sql, new { salesorderId, detailId, quantity });
    }
    finally
    {
        await conn.CloseAsync();
    }
}
{%endhighlight%}

After some testing, a co-worker of mine figured out that the issue was the order of parameters didn't match the order in which they were declared in the SQL. My thought process, having worked much more with SQL Server than Oracle lately, why the hell would that matter? What is the point of having named parameters if the names aren't even used to match them up with the **named** parameters in the SQL code?

Some searching turned up some articles in [Oracle's documentation](https://docs.oracle.com/cd/E85694_01/ODPNT/CommandBindByName.htm) that highlight two things:

1. This is a deliberate design decision by Oracle
2. There is a way to override this behavior on the `OracleCommand` itself.

Overriding this behavior isn't a very clean solution since the extension method lives inside an external library that knows nothing about Oracle, just the `System.Data.Common` abstraction.

## What's the Big Deal?

I don't like this behavior for several reasons. I think *having* named parameters implies that those named are being used by default. The ODBC syntax for SQL parameters is just to use a question mark to delineate them. So the SQL would look like this:

{% highlight sql %}
UPDATE detail_table SET
QUANTITY = ?
WHERE order_id = ? AND detail_id = ?
{%endhighlight%}

Here it's obvious that order would matter since the parameters are identical from a code standpoint. The oracle syntax (above) has names, so why are those names not used or, at the very least, validated? I don't think the people who wrote the Oracle Data Access Components are lazy, but it gives off laziness vibes for sure.

The second issue I have is for future proofing this code. If I need to modify the SQL and add, say, another parameter somewhere in the middle of the code, like so:

{% highlight sql %}
UPDATE detail_table SET
QUANTITY = :quantity,
DATE_MODIFIED = :dateModified
WHERE order_id = :orderId AND detail_id = :detailId
{%endhighlight%}

Now in my code, I need to make sure I'm adding the parameter in the right spot because order matters. I think there are a lot of programmers (*cough*) who wouldn't think twice about just copy-pasting a block at the end of the existing code (now who's being lazy?). Imagine there are dozens or more parameters being passed and you need to make sure they're all in the exact right order. Ugh.

## Workarounds

I came up with 2 potential ways to mitigate this issue.

### Decorating the OracleConnection

`OracleConnection` is a sealed class, but we can just use an adapter to accomplish pretty much the same thing:

{% highlight csharp %}
private class BindByNameOracleConnection : DbConnection
{
    private readonly OracleConnection oracleConnection;

    public BindByNameOracleConnection(OracleConnection oracleConnection)
    {
        this.oracleConnection = oracleConnection;
    }

    // Forward all properties and methods to oracleConnection

    protected override DbCommand CreateDbCommand()
    {
        var command = oracleConnection.CreateCommand();
        command.BindByName = true;
        return command;
    }
}
{%endhighlight%}

Then wrap every instance of `OracleConnection` inside the `BindByNameOracleConnection`.

### Unit Test for Parameter Ordering

If you don't like monkeying with the data access itself, you could setup a few mocks to verify the order of the parameters to make sure they're set in the expected order. Here's a snippet of the kind of thing I was thinking about:

{% highlight csharp %}
private readonly Mock<DbProviderFactory> _mockDbProviderFactory = new();
private readonly Mock<DbConnection> _mockDbConnection = new();
private readonly Mock<DbDataReader> _mockDbDataReader = new();
private readonly Mock<DbCommand> _mockDbCommand = new();
private readonly Mock<DbParameter> _mockParam = new();
private readonly Mock<DbParameterCollection> _mockDbParamCollection = new();
private readonly List<string> _dbParameters = [];

// ...

_mockDbProviderFactory.Setup(f => f.CreateConnection()).Returns(_mockDbConnection.Object);
_mockDbConnection.Protected().Setup<DbCommand>("CreateDbCommand").Returns(_mockDbCommand.Object);
_mockDbCommand.Protected().Setup<Task<DbDataReader>>("ExecuteDbDataReaderAsync", It.IsAny<CommandBehavior>(), It.IsAny<CancellationToken>()).ReturnsAsync(_mockDbDataReader.Object);
_mockDbCommand.Protected().Setup<DbParameter>("CreateDbParameter").Returns(_mockParam.Object);
_mockDbCommand.Protected().SetupGet<DbParameterCollection>("DbParameterCollection").Returns(_mockDbParamCollection.Object);
_mockDbCommand.SetupProperty(c => c.CommandText);
_mockParam.SetupProperty(p => p.ParameterName);
_mockDbParamCollection.Setup(c => c.Add(It.IsAny<DbParameter>())).Callback<object>(param => _dbParameters.Add(((DbParameter)param).ParameterName));

// ...

CollectionAssert.AreEqual(new string[] { "quantity", "dateModified", "orderId", "detailId" }, _dbParameters);
{%endhighlight%}

## Conclusion

I haven't decided yet what I'm going to do, if anything. Maybe writing this down and venting will jog my memory in the future.