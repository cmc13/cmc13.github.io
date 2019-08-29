---
layout: post
title:  "Cancelling Requests in ASP.NET Core"
date:   2019-08-29 12:00:00
categories: programming
tags: dotnet-core c-sharp asp-net-core cancellation
featured-img: cancel.png
---

In creating a new realtime process, the idea of cancellation of requests became an interesting idea to me. As it turns out, .NET Core supports this functionality in a first-class way.

<!-- more -->

## Cancellation Token

.NET has long had the idea of a cancellation token and it has been integrated tightly with the TPL library. There are numerous `Task`-related APIs that accept and honor a cancellation token. When you `async`-ify your Web API controllers, this is no different. There are additional steps to ensure your API understands what you're trying to do, however.

## Startup

If you're modifying an existing Asp.NET core app, you may need to add the following call to `SetCompatibilityVersion`:

{% highlight csharp %}
public void ConfigureServices(IServiceCollection services)
{
    // The CompatibilityVersion will enable cancellation
    services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
}
{%endhighlight%}

## Passing the CancellationToken to Your Action

Now the only thing left to do is pass the cancellation token and use it:

{% highlight csharp %}
[Route("api/[controller]")]
[ApiController]
public class ValuesController : ControllerBase
{
    // GET api/values/5
    [HttpGet("{delay}")]
    public async Task<ActionResult<string>> Get(int delay, CancellationToken cancellationToken)
    {
        try
        {
            await Task.Delay(delay, cancellationToken).ConfigureAwait(false);
            return "done!");
        }
        catch (TaskCanceledException ex) when (ex.CancellationToken == cancellationToken)
        {
            return "cancelled!";
        }
    }
}
{%endhighlight%}

Testing this example is pretty easy. Run the app, and navigate to http://localhost:#####/api/values/n where ##### is replaced by the port set in your debug session and n is some value large enough that you can cancel the request in your browser. You can set a breakpoint on the catch clause to verify that it was actually cancelled when you stop the web request. Pretty cool.

In the real world, you'd want to pass your cancellation token around to whatever potentially-long-running async method is running behind the scenes and watch the token for cancellation. This would cause you to use fewer resources in the event of a timeout. In my case, I have a realtime process that needs a response within 12 seconds or else the API may not even bother. I felt it was right for the calling process to figure out its own timeout and just cancel the request if the API exceeded it. On the API side, I pass the cancellation token to various `HttpClient`s and database handlers so they can stop as soon as the request is cancelled.