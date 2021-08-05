---
comments: true
layout: post
title:  "Go reflect.DeepEqual pitfall - time.Time"
date:   2018-07-06 19:00:00 +0800
tags:
  - Go
---

I have always been using `reflect.DeepEqual()` for checking outputs in functions to be exactly matching what is expected in testings. Recently I encountered a bug that took me a long time to find out what happened.


After carefully assigning my expected object intance, I still couldn't get `DeepEqual()` to return `true` for my expected value and returned value.

Here is my `struct`

{% highlight go %}
type User struct {
	Username     string
	Email        string
	Bio          string
	RegDate      time.Time
	HashedPw     []byte
}
{% endhighlight %}

I tried [spew.Dump()][spewdumpdoc] to visually find any mismatch but no result, they are exactly the same. I also tried [deep.Equal()][deepequaldoc] to see if other library could give me a hint on what I was wrong but still no result.

It took me a while I finally found that it is the `time.Time` object.

They look exactly the same through `fmt.Println()`.

{% highlight go %}
2018-07-06 10:46:29.675 +0000 UTC
2018-07-06 10:46:29.675 +0000 UTC
//UnixNano()
1530873989675000000
1530873989675000000
{% endhighlight %}

### Time

Let's have a look at `time.Time` struct first

{% highlight go %}
type Time struct {
	// wall and ext encode the wall time seconds, wall time nanoseconds,
	// and optional monotonic clock reading in nanoseconds.
	//
	// From high to low bit position, wall encodes a 1-bit flag (hasMonotonic),
	// a 33-bit seconds field, and a 30-bit wall time nanoseconds field.
	// The nanoseconds field is in the range [0, 999999999].
	// If the hasMonotonic bit is 0, then the 33-bit field must be zero
	// and the full signed 64-bit wall seconds since Jan 1 year 1 is stored in ext.
	// If the hasMonotonic bit is 1, then the 33-bit field holds a 33-bit
	// unsigned wall seconds since Jan 1 year 1885, and ext holds a
	// signed 64-bit monotonic clock reading, nanoseconds since process start.
	wall uint64
	ext  int64

	// loc specifies the Location that should be used to
	// determine the minute, hour, month, day, and year
	// that correspond to this Time.
	// The nil location means UTC.
	// All UTC times are represented with loc==nil, never loc==&utcLoc.
	loc *Location
}
{% endhighlight %}

So there are three fields in `time.Time` class, and to dig out more, let's print them all out.

{% highlight go %}
fmt.Println(reflect.ValueOf(&timeA).Elem().Field(0))
fmt.Println(reflect.ValueOf(&timeA).Elem().Field(1))
fmt.Println(reflect.ValueOf(&timeA).Elem().Field(2))
fmt.Println(reflect.ValueOf(&timeB).Elem().Field(0))
fmt.Println(reflect.ValueOf(&timeB).Elem().Field(1))
fmt.Println(reflect.ValueOf(&timeB).Elem().Field(2))
{% endhighlight %}

Since `reflect.Value` has defined a String() function and `time.Time` will just be printed in usual timestamp format, so I have to print them out one by one.

Result:
{% highlight go %}
675000000
63666470789
<nil>
675000000
63666470789
&{Local [{UTC 0 false}] [{-9223372036854775808 0 false false}] -9223372036854775808 9223372036854775807 0xc420156a40}
{% endhighlight %}

The first three lines is from `timeA` and the latter three is from `timeB`. As you can see, they are identical for the first two fields but different at the last.

If the location field is `nil`, it is UTC timezone. My machine is using UTC timezone so Local also represents UTC timezone.

The answer is found and I can finally sleep well at night. My tests failed because `time.Time` allows different representations for timezone, not because of other more horrible reasons.

### Solution
However, it is very easy to solve. 

You just make both of them UTC time specifically.

{% highlight go %}
check := reflect.DeepEqual(timeA.UTC(), timeB.UTC())
{% endhighlight %}

Hope you enjoy my writing. Feel free to comment.

[spewdumpdoc]: https://godoc.org/github.com/davecgh/go-spew/spew#Dump
[deepequaldoc]: https://godoc.org/github.com/go-test/deep#Equal