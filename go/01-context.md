---
layout: post
title: 01 Context
date: 2023-01-17 00:47:00
giscus_comments: true
description: What is the context package for?
tags: context
# categories: sample-posts
---


The context package in Go allows us to take action(mostly stop work) when the response to an API call is slow. This is one of the most frequent use case. 

Let's consider a scenario where a third party API call(`fetchThirdPartyDataWhichCanBeSlow`) takes about five seconds to get the response. 

{% highlight go linenos %}

package main

import (
	"context"
	"fmt"
	"log"
	"time"
)

func main() {
	start := time.Now()
	ctx := context.Background()
	userID := 10
	val, err := fetchUserData(ctx, userID)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("result: ", val)
	fmt.Println("took : ", time.Since(start))
}

func fetchUserData(ctx context.Context, userID int) (int, error) {
	val, err := fetchThirdPartyDataWhichCanBeSlow()
	if err != nil {
		return 0, err
	}
	return val, nil
}

func fetchThirdPartyDataWhichCanBeSlow() (int, error) {
	time.Sleep(time.Millisecond * 500)
	return 666, nil
}

{% endhighlight %}

If we run the above code, the result would be like this.

{% highlight shell %}
% go run context-1.go
result:  666
took :  500.288031ms
{% endhighlight %}

Suppose that we want to control the response time limited to less than two seconds. To do so, we can utilize the context package as following.

{% highlight go linenos %}

package main

import (
	"context"
	"fmt"
	"log"
	"time"
)

func main() {
	start := time.Now()
	ctx := context.Background()
	userID := 10
	val, err := fetchUserData(ctx, userID)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("result: ", val)
	fmt.Println("took : ", time.Since(start))
}

type Response struct {
	value int
	err   error
}

func fetchUserData(ctx context.Context, userID int) (int, error) {
	ctx, cancel := context.WithTimeout(ctx, time.Millisecond*200)
	defer cancel()
	respch := make(chan Response)

	go func() {
		val, err := fetchThirdPartyDataWhichCanBeSlow()
		respch <- Response{
			value: val,
			err:   err,
		}
	}()

	select {
	case <-ctx.Done():
		return 0, fmt.Errorf("fetching data from third party took too long")
	case resp := <-respch:
		return resp.value, resp.err
	}
}

func fetchThirdPartyDataWhichCanBeSlow() (int, error) {
	time.Sleep(time.Millisecond * 500)
	return 666, nil
}

{% endhighlight %}

Give your focus on line 29 and 42 where the context magic takes place. On line 29 the context begins measuring time while the third party API call(`fetchThirdPartyDataWhichCanBeSlow`) is triggered as a **goroutine** on line 33. At the `select` statement on line 41, it is determined which `case` should be returned. In this case, `ctx.Done()` is triggered earlier than the API call, the code will return 0 with the error message. Below is the proof.

{% highlight shell %}
% go run context.go
2023/01/17 00:07:34 fetching data from third party took too long
exit status 1
{% endhighlight %}

If we change the API call's response time from five seconds to one on line 50, it will return the API response without triggering the context's timeout.

{% highlight shell %}
% go run context.go
result:  666
took :  100.677819ms
{% endhighlight %}


*I reference Anthony GG's YouTube video to create this post. Please visit his channel for more detailed explanation.*

<br>

<p style="text-align:center;">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/kaZOXRqFPCw" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
    <div>
        <p style="text-align:center;">How To Use The Context Package In Golang?</p>
    </div>
</p>

<br><br>
