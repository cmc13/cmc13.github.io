---
layout: post
title:  "Unit Testing with the LinkGenerator Class"
date:   2019-06-21 12:00:00
categories: programming
tags: dotnet-core c-sharp linkgenerator unit-testing
featured-img: www.png
---

I had some difficulty today with unit testing what seemed like a simple post action on a controller. I wanted to verify that, on success, it returned a 201 status code. What could be easier? Well, trying to generate the location header proved to be a bit of a learning curve.

In past iterations of ASP.NET, you used the UrlHelper to generate links to other actions (or controllers). .NET Core has introduced an injectable object for this purpose, the `LinkGenerator`.

<!-- more -->

## Usage ##

The usage of this new class is pretty straightforward. 

{% highlight csharp %}
using Microsoft.AspNetCore.Routing;

[Route("api/[controller]")]
public class MyController : ControllerBase
{
    private readonly LinkGenerator linkGenerator;

    public MyController(LinkGenerator linkGenerator)
    {
        this.linkGenerator = linkGenerator;
    }

    [HttpGet]
    public IActionResult Get() { /* ... */ }

    [HttpPost]
    public IActionResult Post(InputModel request)
    {
        /* do action-y things */

        var model = new OutputModel();

        return Created(linkGenerator.GetPathByAction(HttpContext, nameof(Get),
            values: new { /* route values */ }), model);
    }
}
{% endhighlight %}

## Testing

I decided I wanted to test the happy path of to make sure it would return a created status (201). I hadn't yet even thought about testing the location, but I was having some sadness...

{% highlight csharp %}
[TestMethod]
public async Post_HappyPath_ReturnsCreatedStatus()
{
    using (var am = AutoMock.GetLoose())
    {
        var fixture = new Fixture();

        /* set up mocks */

        /* Instantiate controller */
        var ctrl = am.Create<MyController>();

        /* Run the action method */
        var response = await ctrl.Post(fixture.Create<InputModel>());

        /* test the output */
        Assert.IsNotNull(response);
        Assert.IsInstanceOfType(response, typeof(CreatedResult));
    }
}
{% endhighlight %}

## Working the Problem

I tried just about every method of calling `LinkGenerator` methods with various issues. I kept getting a 500 status instead of my expected 201. Eventually, things started to become clear. First of all, within the test, there's no `HttpContext`. After that, I needed to mock the `LinkGenerator` method (note that most of the functionality of `LinkGenerator` is in extension methods) to return...something (anything).


{% highlight csharp %}
[TestMethod]
public async Post_HappyPath_ReturnsCreatedStatus()
{
    using (var am = AutoMock.GetLoose())
    {
        var fixture = new Fixture();

        /* set up mocks */
        /* note that this isn't the same method that I'm calling on the controller.
            Looking at the source code on github (yay open source), this is the
            underlying method that needs to be mocked. */
        am.Mock<LinkGenerator>().Setup(g => g.GetPathByAddress(It.IsAny<HttpContext>(),
            It.IsAny<RouteValuesAddress>(), It.IsAny<RouteValueDictionary>(),
            It.IsAny<RouteValueDictionary>(), It.IsAny<PathString?>(),
            It.IsAny<FragmentString>(), It.IsAny<LinkOptions>()))
                    .Returns("/");

        /* Instantiate controller */
        var ctrl = am.Create<MyController>();
        
        /* Also create a dummy HttpContext */
        ctrl.ControllerContext = new ControllerContext()
        {
            HttpContext = new DefaultHttpContext()
        };

        /* Run the action method */
        var response = await ctrl.Post(fixture.Create<InputModel>());

        /* test the output */
        Assert.IsNotNull(response);
        Assert.IsInstanceOfType(response, typeof(CreatedResult));
    }
}
{% endhighlight %}

This worked. It's my best effort (so far) after striking out on Google results for unit tests involving `LinkGenerator`. Some helpful things I did find were the [Microsoft Docs for the GetPathByAction extension method](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.routing.controllerlinkgeneratorextensions.getpathbyaction?view=aspnetcore-2.2) and also the source code for the extension method on [GitHub](https://github.com/aspnet/Mvc/blob/release/2.2/src/Microsoft.AspNetCore.Mvc.Core/Routing/ControllerLinkGeneratorExtensions.cs).

Happy Coding.