---
layout: post
title:  "Moq Callbacks: How to Make a Failing Unit Test Look Like it Passes"
date:   2019-04-02 10:59:49 -0500
categories: programming
tags: dotnet-core c-sharp moq unit-testing
featured-img: Testing.png
---

As I was getting ready to head home from work on Friday, I compiled, ran a quick smoke test, and verified my unit tests are working. I committed and pushed my changes to the server ~~and went home~~.

To my dismay, after pulling up the build, I noticed a dreaded red X in a place where a green checkmark clearly should have been. Curious, I re-compiled and re-ran my unit tests. Nope, all green. Next thing I know, I’m 2 hours into figuring out why the hell my unit tests pass locally but not on the build server. 

<!-- more -->

{% include figure.html name="https://media.giphy.com/media/kHU8W94VS329y/giphy.gif" alt="Level of frustration: high" caption="via GIPHY" %}

## Show Me the Code

All right, so here’s what I was doing (wrong, obviously). I was unit testing a service whose dependency expects a callback function. Here’s a stripped down version of such a service:

{% highlight csharp %}
// Here's an example of the dependency that expects a callback
public interface ICallbackService
{
    void Register(Func<Task<int>> cb);
}

// And here's a (really dumb) service that uses the dependency
public class Service
{
    public Service(ICallbackService cb)
    {
        // Pseudo-async function, could also be the real deal.
        cb.Register(() => Task.FromResult(5));
    }
}
{% endhighlight %}

And here’s a unit test that I would expect to fail. This is where the problem is, and it’s probably easy to spot the issue for you unit testing experts out there. I’m using a few libraries to be aware of, namely [Moq](https://github.com/moq/moq4) and [AutoMock](https://www.nuget.org/packages/Autofac.Extras.Moq/) (Autofac.Extras.Moq), part of the [Autofac](http://autofac.org/) dependency injection framework.

{% highlight csharp %}
[TestMethod]
public void TestConstructor_CallbackFunctionIsCalled_CorrectValueIsReturned()
{
    using (var am = AutoMock.GetLoose())
    {
        // Setup the register function to fire a callback when it's called.
        // This, in turn, fires the callback parameter passed and tests
        // the return value to make sure it's what we expect.
        am.Mock<ICallbackService>()
            .Setup(r => r.Register(It.IsAny<Func<Task<int>>>()))
            .Callback<Func<Task<int>>>(async cb =>
            {
                var result = await cb();

                Assert.AreEqual(4, result);
            });

        // Generate mocks for all dependencies and instantiate service
        // This should fire the Register function
        am.Create<Program.Service>();
    }
}
{% endhighlight %}

One quick note, while I omitted any code to do synchronization between the callback and the main thread in the test, that’s not the issue we’re looking at here. You should definitely consider synchronization in both your code and tests in the real world, and it was handled correctly in my case, that concern was omitted here in the interest of brevity. 

Seemed like a reasonable test at the time. Little did I know what horrors awaited. So, when I run this test, the expectation would be a failure. Visual Studio, however, reports success.

{% include figure.html name="SuccessfulTest.png" alt="&quot;Successful&quot; Unit Tests" caption="Success! Or Not" %}

Running the code through debug, however, shows the (expected) failure, but it doesn’t propagate to the Test Explorer:

{% include figure.html name="VisualStudioDebugUnitTestCallback.png" alt="Debugging Unit Test" caption="Debugging the “Successful” test" %}

## What About the Command Line?

Yes, well, you might think the command line would be the savior here, mostly because that’s what the build server is going to run, rather than the Visual Studio GUI Test Explorer. By and large, you’d be right, though after several runs I got a mixed result (mostly failures, but some successes, possibly due to the lack of synchronization code mentioned above):

<pre>
C:\Users\admin\source\repos\ConsoleApp4>dotnet test
 Build started, please wait…
 Skipping running test for project C:\Users\admin\source\repos\ConsoleApp4\ConsoleApp4\ConsoleApp4.csproj. To run tests with dotnet test add "true" property to project file.
 Build completed.
 Test run for C:\Users\admin\source\repos\ConsoleApp4\TestProject\bin\Debug\netcoreapp2.1\TestProject.dll(.NETCoreApp,Version=v2.1)
 Microsoft (R) Test Execution Command Line Tool Version 15.9.0
 Copyright (c) Microsoft Corporation.  All rights reserved.
 Starting test execution, please wait…
 The active test run was aborted. Reason: Unhandled Exception: Microsoft.VisualStudio.TestTools.UnitTesting.AssertFailedException: Assert.AreEqual failed. Expected:&lt;4%gt;. Actual:&lt;5&gt;.
    at Microsoft.VisualStudio.TestTools.UnitTesting.Assert.HandleFail(String assertionName, String message, Object[] parameters)
    at Microsoft.VisualStudio.TestTools.UnitTesting.Assert.AreEqual[T](T expected, T actual, String message, Object[] parameters)
    at TestProject.UnitTest1.&lt;&gt;c.&lt;b__0_1&gt;d.MoveNext() in C:\Users\admin\source\repos\ConsoleApp4\TestProject\UnitTest1.cs:line 23
 --- End of stack trace from previous location where exception was thrown ---
    at System.Threading.ExecutionContext.RunInternal(ExecutionContext executionContext, ContextCallback callback, Object state)
 --- End of stack trace from previous location where exception was thrown ---
    at System.Threading.ThreadPoolWorkQueue.Dispatch()
 Test Run Aborted.
</pre>

## The Fix is In

So, the fix was relatively easy, once I started to recognize what was going on and did some Googling. Brian Lachniet [wrote](http://blachniet.com/blog/assertions-in-moq-callbacks/) about this exact problem as well, which would have been nice were I expecting to encounter such weirdness today. As it turns out, using a Moq callback coupled with the asynchronous nature of my own callback ended up biting me. In the end, I simply decided to move my assert outside the callback:

{% highlight csharp %}
[TestMethod]
public void TestConstructor_CallbackFunctionIsCalled_CorrectValueIsReturned()
{
    using (var am = AutoMock.GetLoose())
    {
        int result = 0;

        am.Mock<ICallbackService>()
            .Setup(r => r.Register(It.IsAny<Func<Task<int>>>()))
            .Callback<Func<Task<int>>>(async cb =>
            {
                result = await cb();
            });

        var service = am.Create<Program.Service>();

        Assert.AreEqual(4, result);
    }
}
{% endhighlight %}

Now my tests fail, which they should in this contrived example:

{% include figure.html name="FailingUnitTest.png" alt="Failing Unit Test" caption="Yay! Well, at least it’s doing what it’s supposed to." %}

That’s only half the battle, of course, now I needed to turn the test green. As it turns out, in my case, the test itself was wrong rather than the unit under test. I fixed it, verified that the project now built on the server, and could finally go home for the weekend.

## Lessons Learned

So, my advice is to never trust the Test Explorer and always run your unit tests on the command line before you commit and push, right? Well, I don’t know if I’m willing to go that far. I probably won’t change my workflow for this edge case (although admittedly this is a common pattern for the services I'm writing at work at the moment). I do a lot of my .NET Core coding in VS Code these days so the command line gets run plenty as it is. Writing, running, and debugging unit tests is still easier in VS proper, though, I must admit. This is one of those lessons I had to learn the hard way and hopefully will commit this to memory (and this blog post).