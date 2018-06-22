---
comments: true
layout: post
title:  "Generate random negative and positive integers in range in Golang"
date:   2018-06-21 18:12:32 +0800
tags:
  - Golang
---

Golang built-in package `math/rand` provides random integer generation in  [0,n). With a little modification we can make it works for negative integers too.

{% highlight go %}
func RandInt(lower, upper int) int {
	rand.Seed(time.Now().UnixNano())
	rng := upper - lower
	return rand.Intn(rng) + lower
}
{% endhighlight %}

True random number generation is hard. What `math/rand` does is pseudo-random number generation which is generating a sequence of numbers based on a provided key and it is called seed here. It also means anyone can generate the same sequence of "random" integers if the seed value is known. (For more details, click [Here][wiki-true-vs-pseudo]) Even though `math/rand` is not truely random but we can make it close enough by using a constantly changing key and what we are using here is current Unix time.

#### Warning:
* `rand.Intn()` doesn't accept anything less than zero so lower and upper should not be the same and upper should always be larger than lower.

[wiki-true-vs-pseudo]: https://en.wikipedia.org/wiki/Random_number_generation#%22True%22_vs._pseudo-random_numbers