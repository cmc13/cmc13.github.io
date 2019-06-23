---
layout: post
title:  "Using the New Microsoft.FeatureManagement Library with LaunchDarkly"
date:   2019-06-23 12:00:00
categories: programming
tags: dotnet-core c-sharp feature-management launchdarkly
featured-img: flag.png
---

With Microsoft's recent announcement of their own flavor of feature flag management, I was curious about utilizing it with another, external feature management service, namely the [LaunchDarkly](https://launchdarkly.com/) feature flag service.

What's nice about LaunchDarkly is the ability to separate your feature flags from code and configuration. So, you can toggle your flags with the push of a button and not have to restart or re-deploy your app.

<!-- more -->

## LaunchDarkly Setup

I'm not planning to go too in depth on LaunchDarkly's functionality as there are other great resources for that, but just to level set, I'm installing the `LaunchDarkly.Client` NuGet package and setting up the `LdClient` service for dependency injection.

{% highlight csharp %}
services.AddSingleton<ILdClient>(x => new LdClient(LaunchDarkly.Client.Configuration
    .Default(Configuration["LAUNCHDARKLY_KEY"])
    //.WithLoggerFactory(x.GetService<ILoggerFactory>()) // optional if you want LaunchDarkly logging
    .WithStreamUri(Configuration["LAUNCHDARKLY_RELAY"])));
{% endhighlight %}

## FeatureManagement Setup

The FeatureManagement library is similarly simple and there are already numerous docs and blog posts that talk about this. More or less, install the `Microsoft.FeatureManagement` library (currently in preview state) and then add it to your services.

{% highlight csharp %}
services.AddFeatureManagement();
{% endhighlight %}

Now you can inject the `IFeatureManager` service anywhere you need to feature switch your code.

## And Now for the Magic

So, now I needed to create a custom feature filter that links up with LaunchDarkly. It's a pretty simple implementation, and lacks some functionality of LD.

{% highlight csharp %}
// The Alias will be used in the appsettings configuration
// that we'll be setting up soon
[FilterAlias(nameof(LaunchDarklyFeatureFilter))]
public class LaunchDarklyFeatureFilter : IFeatureFilter
{
    private readonly ILdClient ld;

    public LaunchDarklyFeatureFilter(ILdClient ld)
    {
        this.ld = ld;
    }

    public bool Evaluate(FeatureFilterEvaluationContext context)
    {
        // Just about as simple as it gets. Could have specified a default
        // value as well, but pretty trivial to modify to your exact needs
        return ld.BoolVariation(context.FeatureName, new User("default"));
    }
}
{% endhighlight %}

This is only half the battle. We also need to enable the custom filter in the appsettings.json (or wherever your FeatureManagement configuration is set up). Note that the Feature Alias we set up **must** match the name below.

{% highlight csharp %}
{
  "FeatureManagement": {
    "FeatureFlagName": {
      "EnabledFor": [
        {
          "Name": "LaunchDarklyFeatureFilter",
          "Parameters": {}
        }
      ]
    }
  }
}
{% endhighlight %}

Finally, we actually need to use the flag.

{% highlight csharp %}
public class MyService
{
    private readonly IFeatureManager featureManager;

    public MyService(IFeatureManager featureManager)
    {
        this.featureManager = featureManager;
    }

    public void DoMyServiceThing()
    {
        /* ... */

        /* this should call out to LaunchDarkly to get the current
            state of the feature switch */
        if (featureManager.IsEnabled("FeatureFlagName"))
        {
            /* do my feature stuff */
        }

        /* ... */
    }
}
{% endhighlight %}

So, that's it in a nutshell. Toggling your flag is as easy as clicking a checkbox on a web page.