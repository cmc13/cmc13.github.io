---
layout: post
title:  "Abusing Reflection for Fun and Profit"
date: 2019-04-14 20:24:06
categories: programming
tags: dotnet-core c-sharp reflection
featured-img: Reflection.jpeg
---
There aren't a lot of things that make you feel like more of a badass .NET developer than finding a solid use for reflection. When you're stuck and you have that epiphany moment that you can whip out the perfect tool and solve your problem with a couple simple lines of code, it's a great feeling. Or maybe I just need some better hobbies.

<!-- more -->

{% include figure.html name="https://media.giphy.com/media/A6H1A9rhetsXK/giphy.gif" alt="Haters Gonna Hate" caption="Haters Gonna Hate" %}

## The Problem

I found myself in the situation where I needed to use a library in a way for which it wasn't designed. The library relied heavily on a single abstraction and I wanted to create my own implementation using the same interface. The problem is, implementing the interface required both accessing properties and a constructor that wasn't public. Here's an example of such a scenario.

{% highlight csharp %}
public interface IRequest
{
    Message ToMessage();
    void FromMessage(Message msg);
}

public class Message
{
    // Can't access this property
    internal string Prop { get; set; }

    // Good luck instantiating me
    internal Message() {}
}
{% endhighlight %}

I didn't really want to modify the library as it worked perfectly for all of our use cases except this one (which happened to be a non-production testing tool). So, after banging my head against a wall for a while and nearly giving up and submitting a pull request to modify the library anyway, it hit me. I could use reflection and get around these (deliberate) restrictions.

## The Solution

Here's the heart of the solution. Using reflection, I can query the class for its non-public members and even invoke them as though they were public. I'm not sure if that's cool, scary, or both (probably both).

{% highlight csharp %}
public class MyCustomRequest : IRequest
{
    public Message ToMessage()
    {
        // Get a reference to the constructor using reflection
        var constructor = typeof(Message).GetConstructor(BindingFlags.NonPublic | BindingFlags.Instance,
            null, new Type[0], null);

        // Samesies with the internal property
        var prop = typeof(Message).GetProperty("Prop", BindingFlags.NonPublic | BindingFlags.Instance);

        // Invoke the constructor to instantiate the object
        var msg = (Message)constructor.Invoke(new { });

        // Set the value of the property
        prop.SetValue(msg, "whatever");

        return msg;
    }

    public void FromMessage(Message msg)
    {
        var prop = typeof(Message).GetProperty("Prop", BindingFlags.NonPublic | BindingFlags.Instance);
        var propValue = prop.GetValue(msg);
        // do something with this value
    }
}
{% endhighlight %}

## Final Thoughts

There are a number of things you can take away from this. First and foremost, always keep reflection in your back pocket as a useful tool (even if you don't think this is a good application of it). From the other side, as a library creator/maintainer, oftentimes you make design decisions to influence how someone uses your library. As it turns out, these restrictions are more of a suggestion. Everything is ultimately public if someone is determined enough.