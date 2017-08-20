---
layout: post
title:  "Learning Things By Reinventing the Wheel Series - Part 1"
date:   2017-08-20 02:48:00 +0200
categories: jekyll update
---

## Builder Design Pattern with UrlBuilder


### Resolving Problems on the Go

So, I was working on my Android application which communicates with an API using HTTP requests.
Since there were more than one HTTP request to be made, my laziness told me to make at least a
helper method which has a task to build an URL from the information given. The method declaration
was looking something like this:

{% highlight java %}

public static String urlBuilder(String... urlComponents) {
    // ...
}

{% endhighlight %}

Where `urlComponents` were everything one URL was made of. There was one requirement though,
that components must be in a specific order:

1. Hostname, or base URL (in this specific case the API URL)
2. Resource, exclusively one (you can already see a problem with this)
3. A list of query parameters which had to be prepared before (one parameter key/value pair, one string)

At first, it seemed like it would satisfy my minimalistic needs, since I had pretty much simple API
requests, so I didn't bother changing it until I realized that it won't. You can see, at least,
one problem with resources path which I had pointed, but there's one more, maybe little more subtle which I will soon address.
The problem with resources emerges from potential need for nested resources and something like this:

    http://www.example.com/resource/nested_resource
    
Or even getting resource by id from the collection:

    http://www.example.com/resource/1
    
I could have easily save the day by adding one more for loop in the method implementation,
but that had its own flaws by complicating string concatenations, and introducing one more method parameter,
which indicates nesting level so it could easily determine how to split urlComponents array in two arrays, one with resources, other with parameters.

The thing that made me completely give up on one method idea was the way query parameters were supposed to be treated.
At first, I thought that prepared key/value string as a method argument will be sufficient, but that would made my method pretty cumbersome for usage.
Then, I thought to provide another helper method for forming a parameter string, which led to same problem.
At this point, incorporating parameter forming logic into existing method would led to a lot of spaghetti code,
and I simply didn't like the course method implementation has took.

While I was searching for a way to introduce default parameter values in Java code so that could simplify my method usage in terms of nesting,
I stumbled upon *Builder pattern* as a way of providing them.
Since I don't program in Java that much anymore, I had nearly abandoned whole design pattern concept finding it often as an overkill solution for my everyday problems.
While I still think that design patterns are great, they have a time and place for their usage. It's simply something that you won't use on everyday basis. 
This particular problem was one that had what it takes for a design pattern, the Builder pattern was exactly what I was looking for.

### The Basics of the Builder Pattern

[Builder pattern](https://en.wikipedia.org/wiki/Builder_pattern) is one of the creational design patterns.
[According to Wikipedia](https://en.wikipedia.org/wiki/Creational_pattern),
creational design patterns deal with mechanisms of object creation by providing somewhat level of controlling it.
Builder pattern constructs an object by providing a series of methods for attribute value tweaking of the newly created object.
The *Director* has access to that methods, so the one who has instantiated the *Builder* object has the way of controlling a new object properties.

![Builder pattern](https://upload.wikimedia.org/wikipedia/commons/f/f3/Builder_UML_class_diagram.svg "Builder design pattern")
<p style="text-align:center"><em>Figure 1. UML class diagram of a Builder pattern (source by Wikipedia)</em></p>

The *Product* is what *Builder* provides as a final result.
It is nothing more than a newly created object with additional attribute tweaks as before mentioned.
The *Builder* can be an interface or abstract class, and it could represent a base class/interface for one or many concrete builders.
It is common sense that those builders have to share methods provided by the *Builder* abstraction, so they must be some kind related.
Providing an additional level of abstraction is something that is purely optional, as you can see in the coming example.

### Case Study: UrlBuilder Solution Example

A clumsy written method from the beginning of an article was transformed into following code:

{% highlight java %}

public class UrlBuilder {

    private String url;
    private String query;

    public UrlBuilder hostname(String hostname) {
        if(TextUtils.isEmpty(url)) {
            url = hostname;
        } else {
            throw new RuntimeException("Already has a hostname!");
        }

        return this;
    }

    public UrlBuilder resource(String resource) {
        if(TextUtils.isEmpty(url)) {
            throw new RuntimeException("No URL hostname provided!");
        }

        url = TextUtils.join("/", new String[] { url, resource });

        return this;
    }

    public UrlBuilder parameter(String key, String value) {
        final String p = TextUtils.join("=", new String[] { key, value });
        query = TextUtils.isEmpty(query) ? p : TextUtils.join("&", new String[] { query, p });

        return this;
    }

    public String getUrl() {
        if(TextUtils.isEmpty(url)) {
            throw new RuntimeException("No URL provided!");
        }

        return TextUtils.isEmpty(query) ? url : TextUtils.join("?", new String[] { url, query });
    }

}

{% endhighlight %}

You may be questioning the benefit of the provided solution using Builder pattern.
It sure has a lot more methods and probably more code than the end method solution would have.
While both things are probably true, usage of Builder pattern in this case provides a huge benefit: flexibility.
With one method and variable number of arguments as a way of communication with it, there had to be strictly defined order of passing values.
Example call of such method would be:

{% highlight java %}

final int NESTING_LEVEL = 2;
String url = Util.urlBuilder(NESTING_LEVEL, "http://www.example.com", 
                                            "resource", "nested_resource",
                                            "parameter1=12", "parameter2=34");

{% endhighlight %}
    
This would result with an URL like this:

    http://www.example.com/resource/nested_resource?parameter1=12&parameter2=34
    
Using a provided Builder, we can achieve the same with:


{% highlight java %}

String url = new UrlBuilder.hostname("http://www.example.com")
                           .resource("resource")
                           .resource("nested_resource")
                           .parameter("parameter1", "12")
                           .parameter("parameter2", "34")
                           .getUrl();

{% endhighlight %}

You can notice method chaining going.
This is available because every build method (`hostname`, `resource` and `parameter`) return reference to the builder,
thus providing a way to call another of its methods for the provided reference.
We could have totally mixed up `resource` and `parameter` method calls and would still get same result.
The only constraint is that the `hostname` should be provided before any of the resources.
When we are pleased with a result, all we have to do is to tell our builder to give us a final product by calling `getUrl()` method.

You can also see `RuntimeException` throwing from some methods.
That is one way of handling wrong usage of builders.
There are some rules when building a new URL:

1. There can be only one hostname provided
2. Hostname must be provided before resources, because of concatenation order
3. URL must exists (with or without resources) before getting a final URL product

Other than that, methods can be called in any order, since queries are appended at the very end of product creation.

### Conclusion

I dare to say that provided example is truly where the *Builder pattern* shines.
It is used when we can't say upfront about the properties of the created object, so we want to provide a way to fully customize it before using it in our program.
*Builder pattern* is a nice way to encapsulate the process of customization by providing the builder methods.
One more positive feature is that not all attributes need to have a provided value, because we now have a way to give them default values by builder class instatiation.
There are lot of examples of *Builder pattern* usage in Java libraries, but since I'm programming lately Android application, the example will be [Android dialogs](https://developer.android.com/guide/topics/ui/dialogs.html) classes.

I hope this article provided you a sense for what can you do in real life programming using a *Builder pattern*. Oh, I almost forgot to explain why this article is beginning of a *Reinventing the Wheel* series. Java standard library already has [UrlBuilder](https://docs.oracle.com/cd/E35976_01/studio.240/studio_api_ref/com/endeca/util/UrlBuilder.html) and Apache has its [URIBuilder](http://hc.apache.org/httpcomponents-client-ga/httpclient/apidocs/org/apache/http/client/utils/URIBuilder.html). This all wouldn't have happenned if I wasn't lazy enough to check if there are other, already implemented solutions. So, I have built my own little [Snowflake](https://octo-labs.github.io/snowflake/) and I am proud of it. :)