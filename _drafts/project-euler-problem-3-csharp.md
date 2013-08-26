---
layout: post
title: "Project Euler Problem 3(C#)"
description: "Tackling Project Euler's Problem 3 in C#"
category: articles
tags: [project euler, c#]
#image:
  #feature: so-simple-sample-image-2.jpg
  #credit: Michael Rose
  #creditlink: http://mademistakes.com
comments: false  
---

Lately I've been exploring [Project Euler](http://projecteuler.net/ "Project Euler") as a way to get started with Clojure. This post will feature solutions to [Project Euler Problem 3](http://projecteuler.net/problem=1) in C#, so if you're trying to avoid spoilers then now's your chance to [look at kittens instead](https://encrypted.google.com/search?tbm=isch&q=kittens&tbs=imgo:1). I've decided to split this post due to length. Problem 3 is the Project Euler problem that really hooked me, but it takes some math to explain why. I'll be posting my Clojure solutions later.

Problem 3 is defined as:

> The prime factors of 13195 are 5, 7, 13 and 29.
> 
> What is the largest prime factor of the number 600851475143 ?

The basic solution to this isn't too rough(at least, not on the mind). If you know nothing else about prime numbers, you can reason that since 2 is the smallest possible prime factor, the largest must be value / 2. Knowing this, we can run the following:

{% highlight csharp %}
const long limit = 600851475143;
var ceiling = (long) Math.Ceiling(limit/2.0);

// since 2 is the smallest prime, it's largest factor can't be any larger
long largestPrime = 0;
for (long tester = 2; tester < ceiling; tester++)
{
  if (limit%tester == 0)
    largestPrime = tester;
}
Console.WriteLine(largestPrime);
{% endhighlight %}

If you're planning to run this, I'd also plan on taking a break from the computer. Crunching this took hours on my setup. Now the fun can begin. What can we do to speed this up? Time to brush up on our math! Checking [Wikipedia](https://en.wikipedia.org/wiki/Prime_number), it looks like we can use a couple of tricks to speed this up:

* Employ a sieve to find all of our potential primes. We could use the sieve of [Eratosthenes](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes) or the [sieve of Atkin](https://en.wikipedia.org/wiki/Sieve_of_Atkin), for instance.
* Employ a sieve meant specifically for finding prime factors, like the [rational sieve](https://en.wikipedia.org/wiki/Rational_sieve).

With that in mind, we can go about setting up our sieves. I decided on the sieve of Atkin since it's faster than the sieve of Eratosthenes. Here's what I came up with:

{% highlight csharp %}
public static ICollection<long> PrimesByAtkin(long limit)
{
  if (limit < 2)
    return new HashSet<long>();
  else if (limit < 3)
    return new HashSet<long> { 2 };
  else if (limit < 5)
    return new HashSet<long> { 2, 3 };

  var flip1 = new long[] { 1, 13, 17, 29, 37, 41, 49, 53 };
  var flip2 = new long[] { 7, 19, 31, 43 };
  var flip3 = new long[] { 11, 23, 47, 59 };
  var sqrt = Math.Sqrt(limit);
  var numbers = new HashSet<long>();
  Action<long> flipNumbers = (n) =>
  {
    if (!numbers.Add(n))
      numbers.Remove(n);
  };

  for (long x = 1; x <= sqrt; x++)
  {
    for (long y = 1; y <= sqrt; y++)
    {
      long n = (4 * (x * x)) + (y * y);
      if (n <= limit && (flip1.Contains(n % 60)))
        flipNumbers(n);

      n = (3 * (x * x)) + (y * y);
      if (n <= limit && (flip2.Contains(n % 60)))
        flipNumbers(n);

      n = (3 * (x * x)) - (y * y);
      if (x > y && n <= limit && (flip3.Contains(n % 60)))
        flipNumbers(n);
    }
  }

  var sortedNumbers = numbers.OrderBy(m => m);
  var primes = new HashSet<long> { 2, 3, 5 };

  foreach (var sieveNumber in sortedNumbers)
  {
      if (!numbers.Contains(sieveNumber))
          continue;

      primes.Add(sieveNumber);
      var squareSieve = sieveNumber * sieveNumber;
      var multiplier = 1;
      while (multiplier * squareSieve <= limit)
      {
          numbers.Remove(multiplier * squareSieve);
          multiplier += 1;
      }
  }

  return primes;
}
{% endhighlight %}

With our Atkin sieve in place, we can work on building our Rational Sieve. We'll need two support methods: One to calculate whether the number is [B-Smooth](https://en.wikipedia.org/wiki/Smooth_number); Another to calculate whether the number is [*probably* prime](https://en.wikipedia.org/wiki/Primality_test#Probabilistic_tests); and a third to calculate the [Greatest Common Factor](https://en.wikipedia.org/wiki/Greatest_common_divisor) for two numbers.

{% highlight csharp %}
public static bool IsBSmooth(long value, long bound)
{
  var primes = PrimesByAtkin(value);
  return !primes.Any(m => value % m == 0 && m > bound);
}

public static bool IsPrimeByMillerRabin(long candidate, int numberOfRounds = 1)
{
  if (candidate % 2 == 0 || Math.Abs((Math.Sqrt(candidate) % 1) - 0) < Double.Epsilon || PrimesByAtkin(1000).Any(m => candidate % m == 0))
    return false;

  var s = 0;
  var d = candidate - 1;
  while (d % 2 == 0)
  {
    s++;
    d = d / 2;
  }

  var rand = new Random();
  var probablePrime = true;
  for (int i = 0; i < numberOfRounds; i++)
  {
    var a = rand.Next(2, candidate > int.MaxValue ? int.MaxValue : (int)candidate);
    var potentialPrime = false;
    for (int r = 0; r <= s; r++)
    {
      if (Math.Abs(Math.Pow(a, Math.Pow(2, r) * d) % candidate - (candidate - 1)) < Double.Epsilon)
      {
        potentialPrime = true;
        break;
      }
    }

    if (potentialPrime == false)
    {
      probablePrime = false;
      break;
    }
  }

  return probablePrime;
}

public static long GreatestCommonFactor(long x, long y)
{
  if (x == y)
    return x;

  var divisor = x > y ? y : x;
  while (divisor > 0 && (x % divisor != 0 || y % divisor != 0))
  {
    divisor -= 1;
  }
  return divisor;
}
{% endhighlight %}

Finally, we can build out our Rational sieve:

{% highlight csharp %}
public IEnumerable<long> PrimeFactorsByRationalSieve(long value)
{
  return PrimeFactorsByRationalSieve(value, (long)Math.Ceiling(Math.Sqrt(value)));
}

public IEnumerable<long> PrimeFactorsByRationalSieve(long value, long bound)
{
  var primes = PrimesByAtkin(bound);
  var suitablePrimes = primes.Where(m => value % m == 0);
  if (suitablePrimes.Any())
  {
    var product = suitablePrimes.Aggregate((long)1, (accumulator, current) => accumulator * current);
    return suitablePrimes.Union(this.PrimeFactorsByRationalSieve(value / product, bound));
  }

  if (IsPrimeByMillerRabin(value, 3) && PrimesByAtkin(value).Contains(value))
    return new[] { value };

  var zValues = new Dictionary<int, List<List<int>>>();
  for (int i = 1; i < value && zValues.Count < (primes.Count + 3); i++)
  {
    if (!IsBSmooth(i, bound) || !IsBSmooth(i + value, bound))
      continue;

    var leftList = new List<int>(primes.Count);
    var rightList = new List<int>(primes.Count);
    foreach (var prime in primes)
    {
      var exponent = 1;
      while (i % Math.Pow(prime, exponent) == 0)
      {
          exponent += 1;
      }
      leftList.Add(exponent - 1);

      exponent = 1;
      while ((i + (double)value) % Math.Pow(prime, exponent) == 0)
      {
          exponent += 1;
      }
      rightList.Add(exponent - 1);
    }
    zValues.Add(i, new List<List<int>> { leftList, rightList });
  }

  if (zValues.Any() == false)
    return suitablePrimes;

  var allEvens = zValues.Values.FirstOrDefault(m => m[0].All(n => n % 2 == 0) && m[1].All(n => n % 2 == 0));
  if (allEvens == null)
  {
    for (int i = 0; i < zValues.Count && allEvens == null; i++)
    {
      for (int n = i + 1; n < zValues.Count && allEvens == null; n++)
      {
        var leftList = new List<int>(primes.Count);
        var rightList = new List<int>(primes.Count);
        for (int j = 0; j < primes.Count; j++)
        {
          var leftAddition = zValues.ElementAt(i).Value[0][j] + zValues.ElementAt(n).Value[0][j];
          var rightAddition = zValues.ElementAt(i).Value[1][j] + zValues.ElementAt(n).Value[1][j];
          if (leftAddition == rightAddition)
          {
              leftList.Add(0);
              rightList.Add(0);
          }
          else
          {
              leftList.Add(leftAddition);
              rightList.Add(rightAddition);
          }
        }

        if (leftList.All(m => m % 2 == 0) && rightList.All(m => m % 2 == 0))
            allEvens = new List<List<int>> { leftList, rightList };
      }
    }
  }

  double leftProduct = 1;
  double rightProduct = 1;
  for (int i = 0; i < primes.Count; i++)
  {
      leftProduct *= Math.Pow(primes.ElementAt(i), allEvens[0][i]);
      rightProduct *= Math.Pow(primes.ElementAt(i), allEvens[1][i]);
  }

  var leftSquare = Math.Sqrt(leftProduct);
  var rightSquare = Math.Sqrt(rightProduct);
  return suitablePrimes.Union(new[] { GreatestCommonFactor(Math.Abs((int)leftSquare - (int)rightSquare), value), GreatestCommonFactor((int)leftSquare + (int)rightSquare, value) });
}
{% endhighlight %}

With this, I can run the function without having to leave my desk and go for a walk. It seems silly not to take the other low-hanging fruit here, though: Caching the Atkin sieve. We only need to cache the largest value we've seen so far since we can extrapolate smaller values based on that, so:

{% highlight csharp %}
private static KeyValuePair<long, ICollection<long>> _cachedPrimes = new KeyValuePair<long, ICollection<long>>();

/// <summary>
/// Gets all of the prime numbers up to and including the limit using the sieve of Atkin(https://en.wikipedia.org/wiki/Sieve_of_atkin).
/// </summary>
/// <param name="limit">The limit.</param>
public static ICollection<long> PrimesByAtkin(long limit)
{
  if (limit < 2)
    return new HashSet<long>();
  else if (limit < 3)
    return new HashSet<long> { 2 };
  else if (limit < 5)
    return new HashSet<long> { 2, 3 };

  if (limit <= _cachedPrimes.Key)
    return _cachedPrimes.Value.Where(m => m <= limit).ToArray();

.......

  _cachedPrimes = new KeyValuePair<long, ICollection<long>>(limit, primes);
  return primes;
}
{% endhighlight %}

Of course, nearly all of the methods we've written can be cached. Clojure actually gives you a [method](http://clojure.github.io/clojure/clojure.core-api.html#clojure.core/memoize) to do exactly that quickly and easily, but in .NET we can use [aspects](https://en.wikipedia.org/wiki/Aspect_oriented_programming). [PostSharp](http://www.postsharp.net/) is where I turn when I need AOP in .NET:

{% highlight csharp %}
[PSerializable]
public sealed class CachingAttribute : OnMethodBoundaryAspect
{
  public override void OnEntry(MethodExecutionArgs args)
  {
    var key = GetKey(args);
    var value = MemoryCache.Default.Get(key);
    if (value == null)
        args.MethodExecutionTag = key;
    else
    {
        args.ReturnValue = value;
        args.FlowBehavior = FlowBehavior.Return;
    }
  }

  public override void OnSuccess(MethodExecutionArgs args)
  {
    var cacheKey = (string)args.MethodExecutionTag;
    MemoryCache.Default[cacheKey] = args.ReturnValue;
  }

  private string GetKey(MethodExecutionArgs args)
  {
    var key = new StringBuilder();
    key.Append(args.Method.DeclaringType);
    key.Append("-");
    key.Append(args.Method.Name);
    key.Append("-");
    foreach (var arg in args.Arguments)
    {
        key.Append(arg);
        key.Append("|");
    }
    return key.ToString();
  }
}
{% endhighlight %}

While the Atkin sieve needs more complex behavior, we can throw this on all of our other support functions. Where does this get us? **Mind the logarathmic scale.**

<div id="Chart1" class="chart" title="C# Benchmarks" data-chart-type="column" data-x-categories="Uncached, Atkin Caching, All Caching" data-y-title-text="Runtime(s per 1000 calls)" data-y-scale-type="logarithmic">
  <span class="chart-data" data-name="" data-series="260,230000,230"/>
</div>

Not too shabby! We could improve further, of course: Parallelization and faster sieves are both possibilities. If you end up making improvements(or have any other comments), let me know on [Twitter](https://twitter.com/bretkoppel).