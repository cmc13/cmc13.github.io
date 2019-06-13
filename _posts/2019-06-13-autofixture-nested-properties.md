---
layout: post
title:  "Building Test Objects with Nested Properties Using AutoFixture"
date:   2019-06-13 13:29:00
categories: programming
tags: dotnet-core c-sharp autofixture unit-testing
---

One of the tools I use in unit testing is [AutoFixture](https://github.com/AutoFixture/AutoFixture). This allows me to quickly and easily create test objects as input for the unit-under-test. I decided to use the fluent builder syntax to initialize an object that contained nested properties. I knew the syntax seemed like it might be problematic but I decided to give it a try.

<!-- more -->

Here's the original code:

{% highlight csharp %}
var input = fixture.Build<Models.WM>()
    .With(i => i.PackingDocumentPrint.SONumber, "123")
    .With(i => i.PackingDocumentPrint.StationID, "123")
    .With(i => i.PackingDocumentPrint.OrderType, "ERP")
    .With(i => i.PackingDocumentPrint.DONumber, "1234-1234-1234-1234")
    .Create();
{% endhighlight %}

Running the test, I received the following exception:

> System.ArgumentException: The expression contains access to a nested property or field. Configuration API doesn't support this feature, therefore please rewrite the expression to avoid nested fields or properties.

A Google search was less than helpful. I received 3 results, all of them links to the source code of the AutoFixture library. After spending a minute to think about the problem and the exception message, I settled on the following modification:

{% highlight csharp %}
var input = fixture.Build<Models.WM>()
    .With(i => i.PackingDocumentPrint, fixture.Build<Models.WMPackingDocumentPrint>()
        .With(i => i.SONumber, "123")
        .With(i => i.StationID, "123")
        .With(i => i.OrderType, "ERP")
        .With(i => i.DONumber, "1234-1234-1234-1234")
        .Create())
    .Create();
{% endhighlight %}

This worked like a charm. I hope this saves someone else a little time.