---
layout: post
title:  "Mocking HttpMessageHandler with Moq/AutoMock"
date:   2019-06-05 12:16:00
categories: programming
tags: dotnet-core c-sharp automock unit-testing httpclient httpmessagehandler
featured-img: Spiderman.jpg
---

A common use case when writing microservices at my day job is to call WebApis via HTTP. I've tried a few different libraries but I've been using [RestSharp](http://restsharp.org/) a lot lately. Unfortunately, I ran into an issue with it lately (not relevant here) so I decided to fallback to HttpClient. If you don't know, .NET Core has added some cool new features to HttpClient that make it easier to use dependency injection.

One side effect of my switch was that I had to modify my unit tests to mock `HttpClient` instances instead of `IRestClient` interfaces. We utilize [Moq](https://github.com/moq/moq4) to auto-generate mock objects and [AutoMock](https://www.nuget.org/packages/Autofac.Extras.Moq/) to make our object construction a little easier (and less susceptible to breaking changes). I thought I'd call out what I considered the best way to unit test `HttpClient` using this library.

<!-- more -->

## Google is your Friend

I started off this process in a very familiar place, looking for code I could ~~rip off of~~ learn from on StackOverflow. I found [this](http://hamidmosalla.com/2017/02/08/mock-httpclient-using-httpmessagehandler/) blog post by Hamid Mosalla that was a good launching point, but doesn't use the AutoMock library. Using a subclass to expose a public function to mock didn't work for me but I didn't need to anyway. I decided to just mock the `HttpMessageHandler` class directly and that worked like a charm.

Here's a close approximation to what I ended up doing:

{% highlight csharp %}
[TestMethod]
public async Task DoTheThingAsync_HappyPath_CallsHttpClientWithCorrectUrl()
{
    using (var am = AutoMock.GetLoose())
    {
        // Mock out the HttpMessageHandler abstract class. Since the SendAsync
        // method is protected, we need to use the Moq.Protected namespace to do a
        // setup. Note the specific parameters that must match in order to stub out
        // the right method. You'll get a runtime error if they don't.
        am.Mock<HttpMessageHandler>().Protected()
            .Setup<Task<HttpResponseMessage>>("SendAsync",
                ItExpr.IsAny<HttpRequestMessage>(),
                ItExpr.IsAny<CancellationToken>())
            .ReturnsAsync(new HttpResponseMessage()
            {
                StatusCode = System.Net.HttpStatusCode.OK,
                Content = new StringContent(JsonConvert.SerializeObject(expectedData))
            });
        
        // Provide a specific instance of HttpClient that uses the mock message handler
        am.Provide(new HttpClient(am.Mock<HttpMessageHandler>().Object)
        {
            BaseAddress = new Uri("http://www.example.com")
        });

        // Construct our service that depends on HttpClient
        var service = am.Create<DependsOnHttpClientService>();

        // Call your method
        await service.DoTheThingAsync();

        // You can even verify that the http client was called correctly using Moq:
        am.Mock<HttpMessageHandler>().Protected()
            .Verify("SendAsync",
                Times.Once(),
                ItExpr.Is<HttpRequestMessage>(r => r.RequestUri.AbsoluteUri.EndsWith("asdf")),
                ItExpr.IsAny<CancellationToken>());
    }
}
{% endhighlight %}

I hope somebody finds this helpful. Happy coding.