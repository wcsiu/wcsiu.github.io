---
comments: true
layout: post
title:  "Go fail to iterate through an array"
date:   2019-04-01 16:04:46 +0800
tags:
  - Go
---
Assume you wrote code like this.

``` go
type something struct {
	content string
}

func main() {
	var stringSlice = []something{something{content: "a"}, something{content: "b"}, something{content: "c"}}

	for _, v := range stringSlice {
		go Print(&v)
	}

	time.Sleep(3 * time.Second)

}

func Print(s *something) {
	fmt.Println(s.content)
}
```

#### Output

```
c
c
c
```

#### Expected result
```
a
b
c
```

Why wouldn't the code print all the values in the slice. It may seems untrivial for people not having a ton of golang experience or tangle with the concept of pointer.

After spending sometime digging through the golang spec by myself, I found [this][rangeClause].

> The iteration variables may be declared by the "range" clause using a form of short variable declaration (:=). In this case their types are set to the types of the respective iteration values and their scope is the block of the "for" statement; they are re-used in each iteration. If the iteration variables are declared outside the "for" statement, after execution their values will be those of the last iteration.

The iteration values are reused which means the pointer to the iteration values you passed to the function will change as the for loop progresses. In the above case, the loop reached the end before the goroutines had the chance to start so all the printed out values are the last item of the slice.

It would be easier to be understood with the code below.

``` go
type something struct {
	content string
}

func main() {
	var stringSlice = []something{something{content: "a"}, something{content: "b"}, something{content: "c"}}

	for _, v := range stringSlice {
		go Print(&v)
	}

	time.Sleep(3 * time.Second)

}

func Print(s *something) {
	fmt.Println(s.content)
	fmt.Printf("[Print()] addr: %p\n",s)
}
```

#### Output

```
c
[Print()] addr: 0x40c128
c
[Print()] addr: 0x40c128
c
[Print()] addr: 0x40c128
```

As you can see, the printed pointer addresses are all the same.

[rangeClause]: https://golang.org/ref/spec#RangeClause