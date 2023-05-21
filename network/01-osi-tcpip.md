---
layout: post
title: 01 OSI vs TCP/IP Model
date: 2023-05-20 18:00:00
giscus_comments: true
description: 
tags: OSI TCP/IP Networking
# categories: sample-posts
---


## The OSI Model

1. **Physical** - what types of cables do I use?
2. **Data Link** - anything that works with a MAC address(network cards, switches)
3. **Network** - has to do with logical addresses(IP addresses)(routers)
4. **Transport** - disassembles packets but make sure the packets get to the other system in good order
5. **Session** - connects a server to a client on a remote system(i.e. TCP connection), defines what's taking place in terms of how that connectivity really works
6. **Presentation** - used to convert data into a format that your application can read
7. **Application** - the smarts in the applications that make the network aware

<br>

## The TCP/IP Model

1. **Network Interface** - covers all the physical cables, MAC addresses, network cards... tied to the physical & data link layer for the OSI
2. **Internet** - routers and anything it has to do with an IP address works at the Internet layer
3. **Transport** - has to do with connection to the other system and does all assembly and disassembly(i.e. TCP and UDP)
4. **Application** - everything that has to do with the application itself works at this layer(the OSP application, presentation, and session layer) (i.e. email, FTP, or telnet)

<br>

## Meet the Frame

Devices on a network send and receive data in discrete chunks called **frames**n (or **packets**).

A frame is generated inside the network card. Data comes down into the network card, which creates a frame and shoots it out to the network. Equally, as a frame comes into a network card, the data is pulled away sent up to whatever software needs it.

Frames are **a maximum of 1,500 bytes**(around 10,000 bits) in size and they have a discrete beginning and a discrete end.

<br>

## The MAC address













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
