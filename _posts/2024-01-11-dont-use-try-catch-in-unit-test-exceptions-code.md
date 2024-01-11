---
layout: post
title:  "Don't Use Try-Catch in Unit Test Code that Throws Exceptions"
date:   2024-01-11 13:00:00
categories: programming
tags: dotnet-core c-sharp asp-net-core unit-testing exceptions
---

In cleaning up some legacy code today, I ran across an issue with a unit test that used a bad pattern to test for exceptions being thrown.

<!-- more -->

## Try-Catch in Unit Test Code

In several places in our code base at my job, there were unit tests that utilized a try-catch block to ensure that a particular piece of code threw an exception. These blocks looked something like this:

{% highlight csharp %}
public void TestThatMethodThrowsException
{
    var sut = new SystemUnderTest();

    try
    {
        sut.MethodThatShouldThrowException();
        Assert.Fail();
    }
    catch
    {
        // verify that expected things happened
    }
}
{%endhighlight%}

If the issue isn't immediately apparent to you, welcome to the club I found myself in earlier today. On the outset, it seems like it *should* work and, indeed, the test was passing. However, when I changed the code (out of matter of preference), the test actually failed. Here's an example of the preferred syntax for unit testing exceptions:

{% highlight csharp %}
public void TestThatMethodThrowsException
{
    var sut = new SystemUnderTest();

    Assert.ThrowsException<Exception>(() => sut.MethodThatShouldThrowException());

    // verify that expected things happened
}
{%endhighlight%}

Then, of course, when I ran the test, it failed. Cue head scratching. I hadn't modified the code in the system under test, just the test itself. Why would I be getting different results. I ran the test through the debugger for my sanity's sake and sure enough, no exception being thrown. I then stashed the code changes and ran the old test through the debugger and was hit in the face with the truth. The old code in the system under test didn't throw an exception, either. Which is obvious since it's the *same* code. That's when the underlying behavior of unit tests was (re-)revealed to me.

## Under the Hood of Unit Tests

Something I had seen on occasion, but is largely an implementation detail of unit tests is that, for MSTest at least, unit tests work by throwing an exception, then capturing that output and displaying it, either in the terminal when running a `dotnet test` or in the Test Explorer in Visual Studio (or in whatever IDE you happen to prefer). So, in the above code the catch block and subsequent validation ran when the `Asset.Fail()` throw an `AssertFailedException`. The validations in the catch block were accurate whether an exception was thrown or not, which wasn't helpful.

## Corrections to the Test

As a matter of preference, I didn't like the try-catch syntax in unit tests because it seems more clunky to me. The `Assert.ThrowsException` (and `Assert.ThrowsExceptionAsync`) syntax seemed more purposeful in terms of what is meant to be tested. As I've also now learned, it can be misleading which is another reason I'll be striving to re-write any tests I see using that pattern. I'm sure someone will read this and wonder also why the `ExpectedExceptionAttribute` wouldn't work, and it would, but I think I just prefer the Assert methods. You assert everything else, why not exceptions, too? 