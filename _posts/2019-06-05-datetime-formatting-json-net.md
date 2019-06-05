---
layout: post
title:  "DateTime Formatting with Json.NET"
date:   2019-06-05 13:29:00
categories: programming
tags: dotnet-core c-sharp newtonsoft json datetime
featured-img: Clocks.jpg
---

I thought I'd share some learnings I had with respect to the DateTime struct in .NET and the [Json.NET](https://www.newtonsoft.com/json) library. This all came about due to my worst bout with time zone irregularities coupled with various systems passing dates back and forth with no attached time zone data.

<!-- more -->

## Some Background

I'll admit that I hadn't thought about time zones and the problems they present because it simple hadn't been an issue thus far. Our database stores times in the local time zone (I know, I know...) and that's where this comedy of errors begins. Local dates aren't that bad on their own, but when your database connection library selects dates out with an unspecified `DateTimeKind`, some weirdness is bound to happen.

Dates in .NET core can have 3 "kinds". They can be local, UTC, or unspecified. With either local or UTC, you know what that date means because it carries some extra information that tells you what that time means. Unspecified lacks this information and you must guess whether you're talking about local times or universal ones, and that means potential errors in your code.

So, I had this date that only partially gave me some information and I needed to pass it along to some legacy code via Json. The framework that parses Json back into usable data to call this legacy logic, unbeknownst to me, assumes all dates are UTC. This is a problem because this date was actually local, but didn't know it so the framework parsed it as UTC which was off by 5 hours and, lo and behold, the record that it was supposed to match couldn't be found. I spoke with another developer who had also dealt with this problem recently (because he had worked on the legacy logic that I needed to call and ran into the same issue). The pieces fell into place quickly after that (though not quickly enough to forego the hours of debugging that preceded this conversation).

## Dates in Json.NET

The de-facto standard for representing dates is ISO 8601. This format looks like one of the following:

   * 2019-06-05T13:53:07+00:00
   * 2019-06-05T13:53:07Z
   * 20190605T135307Z

The first format is a "local" time with a time zone offset. The second represents strict UTC with a "Zulu" designator at the end. The last one is the same thing without all the non-digit junk we put in there to make it readable (I don't see this one very often but it's part of the standard). The Json.NET library will display dates in both the local and UTC formats depending on what "kind" you specify as the date:

{% highlight csharp %}
public static void Main()
{
    Console.WriteLine(JsonConvert.SerializeObject(new DateTime(2019, 1, 1, 12, 59, 59, DateTimeKind.Local)));
	// "2019-01-01T12:59:59+00:00"
	Console.WriteLine(JsonConvert.SerializeObject(new DateTime(2019, 1, 1, 12, 59, 59, DateTimeKind.Utc)));
	// "2019-01-01T12:59:59Z"
}
{% endhighlight %}

So, if you have an "unspecified" type of date, what would that look like? Behold:

{% highlight csharp %}
Console.WriteLine(JsonConvert.SerializeObject(new DateTime(2019, 1, 1, 12, 59, 59, DateTimeKind.Unspecified)));
// "2019-01-01T12:59:59"
{% endhighlight %}

Hmmm, technically NOT ISO 8601. This is similar behavior to the `ToString("o")` behavior of the DateTime object itself (though not exactly) so it seems reasonable for the Json.NET library to follow the same pattern.

## DateTimeZoneHandling Manipulation

If you want to override this behavior, Json.NET provides a setting that you can override to make sure that your dates are formatted how you expect in all cases. There are 4 possible values:
   * Local: This would mean you want the library to always serialize dates in the local format with a time zone offset explicitly stated after the date itself
{% highlight csharp %}
Console.WriteLine(JsonConvert.SerializeObject(new DateTime(2019, 1, 1, 12, 59, 59, DateTimeKind.Utc),
    new JsonSerializerSettings()
	{
		DateTimeZoneHandling = DateTimeZoneHandling.Local
	}));
// "2019-01-01T12:59:59+00:00"
{% endhighlight %}

   * Utc: The date should always be converted to the UTC format and have the Zulu designator at the end
{% highlight csharp %}
Console.WriteLine(JsonConvert.SerializeObject(new DateTime(2019, 1, 1, 12, 59, 59, DateTimeKind.Local),
    new JsonSerializerSettings()
	{
		DateTimeZoneHandling = DateTimeZoneHandling.Utc
	}));
// "2019-01-01T12:59:59Z"
{% endhighlight %}

   * Unspecified: The date should carry no time zone information with it (y tho?)
{% highlight csharp %}
Console.WriteLine(JsonConvert.SerializeObject(new DateTime(2019, 1, 1, 12, 59, 59, DateTimeKind.Local),
    new JsonSerializerSettings()
	{
		DateTimeZoneHandling = DateTimeZoneHandling.Unspecified
	}));
// "2019-01-01T12:59:59"
{% endhighlight %}

   * RoundtripKind (the default): This means the DateTimeKind drives the output format of the date. This is what we've already been looking at.

So, you can use the DateTimeZoneHandling setting to control your output if you need precise control over how your dates look when converted to JSON. And, as always, please store dates in UTC.