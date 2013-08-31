---
layout: post
title: "Project Euler Problem 3"
description: "Tackling Project Euler's Problem 3 in C# and Clojure"
category: articles
tags: [project euler, c#, clojure]
comments: false  
---

Lately I've been exploring [Project Euler](http://projecteuler.net/ "Project Euler") as a way to get started with Clojure. This post will feature solutions to [Project Euler Problem 3](http://projecteuler.net/problem=1) in C# and Clojure, so if you're trying to avoid spoilers then now's your chance to [look at kittens instead](https://encrypted.google.com/search?tbm=isch&q=kittens&tbs=imgo:1).

Problem 3 is defined as:

> The prime factors of 13195 are 5, 7, 13 and 29.
> 
> What is the largest prime factor of the number 600851475143 ?

You can brute force this pretty easily using a pre-generated [prime list](http://primes.utm.edu/lists/small/1000.txt). If you know nothing else about prime numbers, you can reason that since 2 and 3 are both prime, then given an odd number *n* the largest factor must be $$ \leq{n \over 3} $$. Using our list of primes as a sanity check, we can run the following and then try our matches until Project Euler lets us through.

{% highlight csharp %}
var primes = new[] {2,3,5,7,11,13,17,19,23,29,31,37,41,43,47,53,59,61,67,71,73,79,83,89,97,101,103,107,109,113,127,131,137,139,149,151,157,163,167,173,179,181,191,193,197,199,211,223,227,229,233,239,241,251,257,263,269,271,277,281,283,293,307,311,313,317,331,337,347,349,353,359,367,373,379,383,389,397,401,409,419,421,431,433,439,443,449,457,461,463,467,479,487,491,499,503,509,521,523,541,547,557,563,569,571,577,587,593,599,601,607,613,617,619,631,641,643,647,653,659,661,673,677,683,691,701,709,719,727,733,739,743,751,757,761,769,773,787,797,809,811,821,823,827,829,839,853,857,859,863,877,881,883,887,907,911,919,929,937,941,947,953,967,971,977,983,991,997,1009};
const long limit = 600851475143;
var ceiling = (long) Math.Ceiling(limit/3.0);

for (long tester = 3; tester < ceiling; tester+=2)
{
  if (limit%tester == 0 && primes.All(m => tester % m > 0))
    Console.WriteLine("Prime Candidate: " + tester);
}
{% endhighlight %}

If you're planning to run this, I'd also plan on taking a break from the computer. A *long* break. Crunching this took 25 minutes on my setup and, frankly, the pre-generated prime list feels a little like cheating. Now the fun can begin. What can we do to speed this up?

* If we had a [primality test](https://en.wikipedia.org/wiki/Primality_test) then we could use that instead of relying on our pre-generated list of primes. We could also verify that the number isn't prime before going through all of our calculations.
* If we know a number *n* isn't prime, we can use simple [trial division](https://en.wikipedia.org/wiki/Trial_division) and limit our search to numbers less than $$ \sqrt n $$.

C# doesn't include a built in primality test, so our first task is to build that out(or grab an [existing implementation](https://duckduckgo.com/?q=c%23+primality+test) ). I chose to build out the [Miller-Rabin test](https://en.wikipedia.org/wiki/Miller-Rabin):

{% highlight csharp %}
public static bool IsPrimeByMillerRabin(BigInteger candidate, int numberOfRounds = 5)
{
  if (candidate <= 3)
    return true;

  if (candidate % 2 == 0)
    return false;

  var s = 0;
  var d = candidate - 1;
  while (d % 2 == 0)
  {
    s++;
    d = d / 2;
  }
  
  var rand = new Random();
  for (int i = 0; i < numberOfRounds; i++)
  {
    var a = rand.Next(2, candidate > int.MaxValue ? int.MaxValue - 2 : (int)candidate - 2);
    var c = BigInteger.ModPow(a, d, candidate);
    if (c == 1 || c == candidate-1)
      continue;

    for (int r = 0; r < s; r++)
    {
      c = BigInteger.ModPow(c, 2, candidate);
      if (c == 1)
        return false;

      if (c == candidate - 1)
        break;

      if(r == s-1)
        return false;
    }
  }

  return true;
}
{% endhighlight %}

With our prime test in place, we can move on to implementing trial division:

{% highlight csharp %}
public static void Problem3_TrialDivision()
{
  Console.WriteLine(PrimeFactorsByTrialDivision(600851475143).Max());
}

public IEnumerable<BigInteger> PrimeFactorsByTrialDivision(BigInteger value)
{
  if (IsPrimeByMillerRabin(value, 5))
    return new[] {value};

  var primeFactors = new HashSet<BigInteger>();
  while (value % 2 == 0)
  {
    primeFactors.Add(2);
    value /= 2;
  }

  var ceiling = new BigInteger(MathHelpers.Sqrt(value)) + 1;
  for (BigInteger tester = 3; tester < ceiling; tester += 2)
  {
    if (value % tester == 0 && IsPrimeByMillerRabin(tester, 5))
    {
      value /= tester;
      primeFactors.Add(tester);
      if (IsPrimeByMillerRabin(value))
      {
          primeFactors.Add(value);
          break;
      }

      tester -= 2;
    }
  }

  return primeFactors;
}
{% endhighlight %}

This quickly yields our answer without any of the nasty hacks we had in the first(very slow) solution. Before we go any further, let's see what this might look like in Clojure. Java provides a convenient [isProbablePrime method](http://docs.oracle.com/javase/7/docs/api/java/math/BigInteger.html#isProbablePrime(int)) so we can do the following:

{% highlight clojure %}
(defn prime-factors-by-trial-division
  "Gets the frequencies of each prime factor for value."
  [value] 
  (loop [v value testers (range 3 value 2) primes []]
    (let [t (first testers)]
      (cond
        (< v 3) (frequencies (conj primes v))
        (.isProbablePrime (BigInteger. (str v)) 3) (frequencies (conj primes v))
        (zero? (mod v 2)) (recur (/ v 2) testers (conj primes 2))
        (and (.isProbablePrime (BigInteger. (str t)) 3) (zero? (mod v t))) (recur (/ v t) testers (conj primes t))
        :else (recur v (next testers) primes)
      )
    )
  )
)
(println "Max Factor: " (first (sort-by - (keys (prime-factors-by-trial-division 600851475143)))))
{% endhighlight %}

Trial division on a number of this size is plenty fast, but I wanted to go a little further down the rabbit hole. After all, looking ahead at the Project Euler problem list makes it seem like some additional prime-related methods might come in handy. Looking up the pages on [integer factorization](https://en.wikipedia.org/wiki/Integer_factorization) and [prime numbers](https://en.wikipedia.org/wiki/Prime_numbers) introduces the idea of sieves. Would it be faster to generate all primes up to a number and then test only those values? I took a stab at recreating the [sieve of Atkin](https://en.wikipedia.org/wiki/Sieve_of_Atkin) to find out.

{% highlight csharp %}
public static ICollection<BigInteger> PrimesByAtkin(BigInteger limit)
{
  var primes = new HashSet<BigInteger> { 2, 3, 5 };
  if (limit <= 5)
    return primes.Where(m => m <= limit).ToArray();

  var cachedPrimes = MemoryCache.Default.Get("LargestAtkin");
  if (cachedPrimes != null)
  {
    var castPrimes = (KeyValuePair<BigInteger, ICollection<BigInteger>>)cachedPrimes;
    if (limit == castPrimes.Key)
      return castPrimes.Value.ToArray();
    
    if (limit < castPrimes.Key)
      return castPrimes.Value.Where(m => m <= limit).ToArray();
  }

  var flip1 = new BigInteger[] { 1, 5 };
  var flip2 = new BigInteger[] { 7 };
  var flip3 = new BigInteger[] { 11 };

  // no built in sqrt, but MSDN offers this(http://msdn.microsoft.com/en-us/library/dd268263.aspx)
  var sqrt = new BigInteger(MathHelpers.Sqrt(limit));
  var numbers = new HashSet<BigInteger>();
  Action<BigInteger> flipNumbers = (n) =>
  {
    if (!numbers.Add(n))
      numbers.Remove(n);
  };

  for (BigInteger x = 1; x <= sqrt; x++)
  {
    for (BigInteger y = 1; y <= sqrt; y++)
    {
      BigInteger n = (4 * (x * x)) + (y * y);
      if (n <= limit && (flip1.Contains(n % 12)))
        flipNumbers(n);

      n = (3 * (x * x)) + (y * y);
      if (n <= limit && (flip2.Contains(n % 12)))
        flipNumbers(n);

      n = (3 * (x * x)) - (y * y);
      if (x > y && n <= limit && (flip3.Contains(n % 12)))
        flipNumbers(n);
    }
  }

  var sortedNumbers = numbers.OrderBy(m => m);
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

  MemoryCache.Default["LargestAtkin"] = new KeyValuePair<BigInteger, ICollection<BigInteger>>(limit, primes);
  return primes;
}

public IEnumerable<BigInteger> PrimeFactorsByAtkin(BigInteger value)
{
  // verify the number isn't prime
  if (IsPrimeByMillerRabin(value, 10))
    return new[] { value };

  var firstPrimes = PrimesByAtkin(100).OrderBy(m => m);
  var primes = firstPrimes.Where(m => value % m == 0).ToList();
  var primeFactors = new HashSet<BigInteger>(primes);
  do
  {
    value = primes.Aggregate(value, (current, prime) => current/prime);

    if (IsPrimeByMillerRabin(value, 5))
    {
        primeFactors.Add(value);
        return primeFactors;
    }
    primes = firstPrimes.Where(m => value % m == 0).ToList();
  } while (primes.Any());

  return primeFactors.Union(PrimeFactorsByTrialDivision(value));
}
{% endhighlight %}

This was about 50% faster than the previous version and there's still room for improvement. I might post a follow-up about some of the other solutions I've implemented - including one that uses a [rational sieve](https://en.wikipedia.org/wiki/Rational_sieve) - but this seems like plenty for now. Let me know about your great solutions(or bugs in my code) by contacting me on [Twitter](https://twitter.com/bretkoppel).