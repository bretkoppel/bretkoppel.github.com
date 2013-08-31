---
layout: post
title: "Cache Your XML Serializers"
description: "Cache Your XML Serializers to Boost Performance by 3x"
comments: false
category: articles
tags: [c#]
---
"The Site is Slow"
---
We'd been getting reports about "slow" performance for a short time, but our feature list kept us from really looking into it. Finally, a primary client contact complained triggering a series of performance optimizations. In this post, I'll explore one of the culprits: failing to cache our XML serializers.

Finding the Problem
---
A quick profile of network traffic on our pages revealed a group of sluggish AJAX calls. Generally this points to a database issue, but profiling the database didn't back that up - Everything was coming back quickly, but wasn't making it to the UI for ages. Something was going wrong in our code.

ANTS Marching
---
My first stop was the built-in [Visual Studio Profiling Tools](http://msdn.microsoft.com/en-us/library/z9z62c29.aspx). Unfortunately, they [don't work out of the box with IIS Express](http://blogs.msdn.com/b/profiler/archive/2011/05/02/how-to-profile-iis-express-with-visual-studio-2010-sp1.aspx). I fiddled with setup for a bit, but in the end I went with [ANTS Profiler](http://www.red-gate.com/products/dotnet-development/ants-performance-profiler/) because they're clear and easy to use out of the box. A few runs through ANTS pointed me to our serialization methods.

Everything looked kosher in the code. Some quick searches told me to [use Sgen](http://msdn.microsoft.com/en-US/library/bk3w6240(v=vs.100).aspx) or switch to another serializer, like [protobuf](https://developers.google.com/protocol-buffers/). Sgen didn't provide the boost I was hoping for, and swapping serializers was going to be a headache. Frustrated, I shut down for the night and went home.

Cache Everything
---
One of my searches eventually lead me to [Scott Hanselman's post](http://www.hanselman.com/blog/XmlSerializerMadnessTheGutsAndTheConclusion.aspx) on caching XML serializers. Could something that plagued us in .NET 1.1 still be around in .NET 4? Sure enough, a little reflecting showed that the current XML serializer still only caches calls to two constructors: [XmlSerializer(Type)](http://msdn.microsoft.com/en-us/library/71s92ee1.aspx) and [XmlSerializer(Type, String)](http://msdn.microsoft.com/en-us/library/kw0f5wee.aspx). Unfortunately, we use the [XmlSerializer(Type, Type[])](http://msdn.microsoft.com/en-us/library/e5aakyae.aspx) constructor extensively.

A bunch of refactoring later, I ended up with the following:
{% highlight csharp %}
public static XmlSerializer GetSerializer(Type type, Type[] extraTypes = null)
{
  if (type == null)
  {
    throw new ArgumentNullException("type");
  }

  int hash = 17;
  unchecked
  {
    hash = (hash * 17) + type.GetHashCode();
    if (extraTypes != null)
    {
      foreach (var extraType in extraTypes)
      {
          hash = (hash * 17) + extraType.GetHashCode();
      }
    }
  }

  var cacheName = "XmlSerializer-" + hash;
  var cachedSerializer = MemoryCache.Default[cacheName];
  if (cachedSerializer != null)
  {
    return (XmlSerializer)cachedSerializer;
  }

  var serializer = (extraTypes == null || extraTypes.Length == 0) ? new XmlSerializer(type) : new XmlSerializer(type, extraTypes);
  var cachePolicy = new CacheItemPolicy { SlidingExpiration = TimeSpan.FromHours(1) };
  MemoryCache.Default.Set(cacheName, serializer, cachePolicy);
  return serializer;
}
{% endhighlight %}

This actually ends up *over-caching*, which will need to be cleaned up. If we don't have any extra types, the serializer will cache itself so there's no point in me taking up extra memory.

Results
---
The results were pretty impressive:

<div id="Chart1" class="chart" title="Serializer Caching: Before and After" data-chart-type="column" data-x-categories="Fresh Load,Subsequent Loads" data-y-title-text="Runtime({units})" data-unit="s">
  <span class="chart-data" data-name="Uncached" data-series="108,27"/>
  <span class="chart-data" data-name="Cached" data-series="90,8"/>
</div>

Less than a third of the time? I'll take it!

Comments? Find me on [Twitter](https://twitter.com/bretkoppel).
