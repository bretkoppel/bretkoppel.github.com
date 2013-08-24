---
layout: post
title: "Project Euler Problem 1"
description: "Tackling Project Euler's Problem 1 in C# and Clojure"
category: articles
tags: [project euler, clojure, c#]
#image:
  #feature: so-simple-sample-image-2.jpg
  #credit: Michael Rose
  #creditlink: http://mademistakes.com
comments: false  
---

Lately I've been exploring [Project Euler](http://projecteuler.net/ "Project Euler") as a way to get started with Clojure. This post will feature solutions to [Project Euler Problem 1](http://projecteuler.net/problem=1) in C# and Clojure, so if you're trying to avoid spoilers then now's your chance to [look at kittens instead](https://encrypted.google.com/search?tbm=isch&q=kittens&tbs=imgo:1). Problem 1 is defined as:

> If we list all the natural numbers below 10 that are multiples of 3 or 5, we get 3, 5, 6 and 9. The sum of these multiples is 23.
>
> Find the sum of all the multiples of 3 or 5 below 1000.

What I love about the problems is that each one is as hard as you want to make it. Problem 1, for instance, is doable as a one-liner in C#:

{% highlight csharp %}
// Slow
// Why doesn't range just take actual boundaries? The world may never know.
Console.WriteLine(
  Enumerable.Range(3, 997)
  .Where(m => m % 3 == 0 || m % 5 == 0)
  .Sum()
);
{% endhighlight %}

Easy, right? I wonder if a swap to Aggregate will speed it up at all.

{% highlight csharp %}
// Faster
Console.WriteLine(
  Enumerable.Range(3, 997)
  .Aggregate(0, (acc, item) => acc + ((item % 3 == 0 || item % 5 == 0) ? item : 0))
);
{% endhighlight %}

Neither of these really appeal to me, though. Why generate 999 elements when we're only ever going to use a few of 'em?

{% highlight csharp %}
// Fastest
var sum = 0;
var multiplier = 1;
while (multiplier*3 < 1000)
{
  sum += multiplier*3;
  var five = multiplier*5;
  if (five < 1000 && five%3 != 0)
    sum += five;

  multiplier += 1;
}
{% endhighlight %}

Alright, let's see how these compare. Here's how the benchmarks work out over 1000 runs in the VM I'm using:

<div id="Chart1" class="chart" title="C# Benchmarks" data-chart-type="column" data-x-categories="Solution 1, Solution 2, Solution 3" data-y-title-text="Runtime(ms per 1000 calls)">
  <span class="chart-data" data-name="" data-series="17,14,2"/>
</div>

Aggregate provides a small advantage over filtering and summing, but neither comes close to just calculating what we need. Not a bad speed increase if you're willing to write a few extra lines of code, right? Let's move on to Clojure. Having started looking at Clojure only a few days ago, my solutions aren't going to be idiomatic or super efficient and I'm open to feedback. With that caveat out of the way, here's my initial solution:

{% highlight clojure %}
; The range function here behaves as I would expect it to. Hooray!
(reduce +
  (filter 
    #(or (= 0 (mod % 3)) (= 0 (mod % 5)))
    (range 3 1000)))
{% endhighlight %}

You can plug this and future examples into the online [REPL][1] to verify them if you don't happen to have Clojure installed. I have to admit, the lisp syntax is growing on me. The first time I looked at all of those parentheses I found myself walking to the kitchen for a glass of wine, but now I'm starting to find a certain beauty in them. Anyway, this first solution is analagous to the slowest C# solution above, which we know isn't an efficient method for dealing with the problem. Let's see if we can do any better:

{% highlight clojure %}
(defn threes-and-fives [limit]
  (letfn [(multiplier [n acc]
    (let [three (* 3 n) five (* 5 n)]
      (if (< three limit)        
        (recur (inc n) (if (or (>= five limit) (zero? (mod five 3))) (+ three acc) (+ three five acc)))
        acc
      )))]
    (multiplier 1 0)))

(threes-and-fives 1000)
{% endhighlight %}

This looks and feels ghoulish so I'm going to assume it's far from idiomatic(the names alone are too long given the Clojure I've seen). However, it benchmarks a heck of a lot better. Speaking of which, I had no idea how to benchmark Clojure, but some quick [DDG][2]ing yielded the [Criterium][3] library which was easy to get up and running even for a total novice. Assuming you have [Leiningen 2.0+][4] but haven't set up any global plugins yet, you can just:

{% highlight bash %}
echo {:user {:plugins [[criterium "0.4.1"]]}} > ~/.lein/profiles.clj
{% endhighlight %}

Then to benchmark the solutions:

{% highlight clojure %}
(use 'criterium.core)
(bench (reduce + (filter #(or (= 0 (mod % 3)) (= 0 (mod % 5))) (range 3 1000))))
(bench (threes-and-fives 1000))
{% endhighlight %}

Anyway, on to the benchmarks:

<div id="Chart2" class="chart" title="Clojure Benchmarks" data-chart-type="column" data-x-categories="Solution 1,Solution 2" data-y-title-text="Runtime({units} per call)" data-unit="μs">
  <span class="chart-data" data-name="" data-series="269,68"/>
</div>

Note that the benchmarks aren't comparable between languages as they're running on totally different machine configurations. As expected, the second solution is much quicker than the first. I'm sure there's a way to improve on it more. If you have one(or any other comments), let me know on [Twitter](https://twitter.com/bretkoppel).

[1]:	http://tryclj.com/ "Try Clojure"
[2]:	https://duckduckgo.com/?q=clojure+benchmark+criterium "DuckDuckGo"
[3]:	https://github.com/hugoduncan/criterium "Criterium"
[4]:  http://leiningen.org/ "Leiningen"