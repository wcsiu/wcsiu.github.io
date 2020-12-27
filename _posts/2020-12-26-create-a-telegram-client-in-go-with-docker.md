---
comments: true
layout: post
title: "Create a Telegram client in Go using TDLib with Docker"
date: 2020-12-26 22:36:22 +0800
tags:
  - Go
  - Telegram
  - TDLib
  - Docker
---

After this tutorial, you will be able to build a telegram client with API to retrieve chat list from it in Go. This tutorial assume you have a basic understanding of Docker, how it works and most importantly having it installed in you machine. If you don't, it would be great for you go learn [some basic concepts about it][dockerConcepts] first.

# TL;DR
Github demo code: [https://github.com/wcsiu/telegram-client-demo][demoRepo]

# Let's begin.

## 1 - Create a Telegram application
Before we start doing any programming, we need to create an app on Telegram first. For details, click [here][createTelegramApp]. After this, you should have your `app_id` and `app_hash`.

## 2 - The Client application
To build a telegram client, we need to use a library called [TDLib][tdlibGithub]. It is library written in C++ and so we can leverage [cgo][cgo] to write our client in Go.

[go-tdlib][go-tdlib] is the goto Go library for TDLib. It is being actively maintained. This tutorial will be using it.

### Coding part:
First, we need a TDLib client.

{% highlight go %}
client = tdlib.NewClient(tdlib.Config{
	APIID:               "FILL YOUR API ID HERE",
	APIHash:             "FILL YOUR API HASH HERE",
	SystemLanguageCode:  "en",
	DeviceModel:         "Server",
	SystemVersion:       "1.0.0",
	ApplicationVersion:  "1.0.0",
	UseMessageDatabase:  true,
	UseFileDatabase:     true,
	UseChatInfoDatabase: true,
	UseTestDataCenter:   false,
	DatabaseDirectory:   "./tdlib-db",
	FileDirectory:       "./tdlib-files",
	IgnoreFileNames:     false,
})
{% endhighlight %}

You will have to fill in your `app_id` and `app_hash` you get from the previous section. The rest we can leave it as default.

Then, we need to authorize our client.

{% highlight go %}
go func() {
	for {
		var currentState, _ = client.Authorize()
		if currentState.GetAuthorizationStateEnum() == tdlib.AuthorizationStateWaitPhoneNumberType {
		fmt.Print("Enter phone: ")
		var number string
		fmt.Scanln(&number)
		_, err := client.SendPhoneNumber(number)
		if err != nil {
			fmt.Printf("Error sending phone number: %v\n", err)
		}
		} else if currentState.GetAuthorizationStateEnum() == tdlib.AuthorizationStateWaitCodeType {
		fmt.Print("Enter code: ")
		var code string
		fmt.Scanln(&code)
		_, err := client.SendAuthCode(code)
		if err != nil {
			fmt.Printf("Error sending auth code : %v\n", err)
		}
		} else if currentState.GetAuthorizationStateEnum() == tdlib.AuthorizationStateWaitPasswordType {
		fmt.Print("Enter Password: ")
		var password string
		fmt.Scanln(&password)
		_, err := client.SendAuthPassword(password)
		if err != nil {
			fmt.Printf("Error sending auth password: %v\n", err)
		}
		} else if currentState.GetAuthorizationStateEnum() == tdlib.AuthorizationStateReadyType {
		fmt.Println("Authorization Ready! Let's rock")
		break
		}
	}
}()
{% endhighlight %}

The phone number should include country code.The above authorization code snippet allows you to authorize through password or security code in your client.

To get all the chat names, we need to get the list of chat IDs and get chat details one by one.

{% highlight go %}
func getChatList(client *tdlib.Client, limit int) ([]*tdlib.Chat, error) {
	var allChats []*tdlib.Chat
	var offsetOrder = int64(math.MaxInt64)
	var offsetChatID = int64(0)
	var chatList = tdlib.NewChatListMain()
	var lastChat *tdlib.Chat

	for len(allChats) < limit {
		if len(allChats) > 0 {
			lastChat = allChats[len(allChats)-1]
			for i := 0; i < len(lastChat.Positions); i++ {
				//Find the main chat list
				if lastChat.Positions[i].List.GetChatListEnum() == tdlib.ChatListMainType {
					offsetOrder = int64(lastChat.Positions[i].Order)
				}
			}
			offsetChatID = lastChat.ID
		}

		// get chats (ids) from tdlib
		var chats, getChatsErr = client.GetChats(chatList, tdlib.JSONInt64(offsetOrder),
			offsetChatID, int32(limit-len(allChats)))
		if getChatsErr != nil {
			return nil, getChatsErr
		}
		if len(chats.ChatIDs) == 0 {
			return allChats, nil
		}

		for _, chatID := range chats.ChatIDs {
			// get chat info from tdlib
			var chat, getChatErr = client.GetChat(chatID)
			if getChatErr == nil {
				allChats = append(allChats, chat)
			} else {
				return nil, getChatErr
			}
		}
	}

	return allChats, nil
}
{% endhighlight %}

Finally, we add a HTTP router and HTTP handler for `GET /getChats`.

{% highlight go %}
http.HandleFunc("/getChats", getChatsHandler)
http.ListenAndServe(":3000", nil)
{% endhighlight %}

{% highlight go %}
func getChatsHandler(w http.ResponseWriter, req *http.Request) {
	var allChats, getChatErr = getChatList(client, 1000)
	if getChatErr != nil {
		w.WriteHeader(http.StatusInternalServerError)
		w.Write([]byte(getChatErr.Error()))
		return
	}

	var retMap = make(map[string]interface{})
	retMap["total"] = len(allChats)

	var chatTitles []string
	for _, chat := range allChats {
		chatTitles = append(chatTitles, chat.Title)
	}

	retMap["chatList"] = chatTitles

	var ret, marshalErr = json.Marshal(retMap)
	if marshalErr != nil {
		w.WriteHeader(http.StatusInternalServerError)
		w.Write([]byte(marshalErr.Error()))
		return
	}

	io.WriteString(w, string(ret))
}
{% endhighlight %}

Combine them all the source code would be like this.

{% highlight go %}
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"math"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/Arman92/go-tdlib"
)

var client *tdlib.Client

func main() {
	tdlib.SetLogVerbosityLevel(1)
	tdlib.SetFilePath("./errors.txt")

	// Create new instance of client
	client = tdlib.NewClient(tdlib.Config{
		APIID:               "FILL YOUR API ID HERE",
		APIHash:             "FILL YOUR API HASH HERE",
		SystemLanguageCode:  "en",
		DeviceModel:         "Server",
		SystemVersion:       "1.0.0",
		ApplicationVersion:  "1.0.0",
		UseMessageDatabase:  true,
		UseFileDatabase:     true,
		UseChatInfoDatabase: true,
		UseTestDataCenter:   false,
		DatabaseDirectory:   "./tdlib-db",
		FileDirectory:       "./tdlib-files",
		IgnoreFileNames:     false,
	})

	// Handle Ctrl+C , Gracefully exit and shutdown tdlib
	var ch = make(chan os.Signal, 2)
	signal.Notify(ch, os.Interrupt, syscall.SIGTERM)
	go func() {
		<-ch
		client.DestroyInstance()
		os.Exit(1)
	}()

	go func() {
		for {
			var currentState, _ = client.Authorize()
			if currentState.GetAuthorizationStateEnum() == tdlib.AuthorizationStateWaitPhoneNumberType {
				fmt.Print("Enter phone: ")
				var number string
				fmt.Scanln(&number)
				var _, err = client.SendPhoneNumber(number)
				if err != nil {
					fmt.Printf("Error sending phone number: %v\n", err)
				}
			} else if currentState.GetAuthorizationStateEnum() == tdlib.AuthorizationStateWaitCodeType {
				fmt.Print("Enter code: ")
				var code string
				fmt.Scanln(&code)
				var _, err = client.SendAuthCode(code)
				if err != nil {
					fmt.Printf("Error sending auth code : %v\n", err)
				}
			} else if currentState.GetAuthorizationStateEnum() == tdlib.AuthorizationStateWaitPasswordType {
				fmt.Print("Enter Password: ")
				var password string
				fmt.Scanln(&password)
				var _, err = client.SendAuthPassword(password)
				if err != nil {
					fmt.Printf("Error sending auth password: %v\n", err)
				}
			} else if currentState.GetAuthorizationStateEnum() == tdlib.AuthorizationStateReadyType {
				fmt.Println("Authorization Ready! Let's rock")
				break
			}
		}
	}()

	// Wait while we get Authorization Ready!
	// Note: See authorization example for complete auhtorization sequence example
	var currentState, _ = client.Authorize()
	for ; currentState.GetAuthorizationStateEnum() != tdlib.AuthorizationStateReadyType; currentState, _ = client.Authorize() {
		time.Sleep(300 * time.Millisecond)
	}

	http.HandleFunc("/getChats", getChatsHandler)
	http.ListenAndServe(":3000", nil)
}

func getChatsHandler(w http.ResponseWriter, req *http.Request) {
	var allChats, getChatErr = getChatList(client, 1000)
	if getChatErr != nil {
		w.WriteHeader(http.StatusInternalServerError)
		w.Write([]byte(getChatErr.Error()))
		return
	}

	var retMap = make(map[string]interface{})
	retMap["total"] = len(allChats)

	var chatTitles []string
	for _, chat := range allChats {
		chatTitles = append(chatTitles, chat.Title)
	}

	retMap["chatList"] = chatTitles

	var ret, marshalErr = json.Marshal(retMap)
	if marshalErr != nil {
		w.WriteHeader(http.StatusInternalServerError)
		w.Write([]byte(marshalErr.Error()))
		return
	}

	io.WriteString(w, string(ret))
}

// see https://stackoverflow.com/questions/37782348/how-to-use-getchats-in-tdlib
func getChatList(client *tdlib.Client, limit int) ([]*tdlib.Chat, error) {
	var allChats []*tdlib.Chat
	var offsetOrder = int64(math.MaxInt64)
	var offsetChatID = int64(0)
	var chatList = tdlib.NewChatListMain()
	var lastChat *tdlib.Chat

	for len(allChats) < limit {
		if len(allChats) > 0 {
			lastChat = allChats[len(allChats)-1]
			for i := 0; i < len(lastChat.Positions); i++ {
				//Find the main chat list
				if lastChat.Positions[i].List.GetChatListEnum() == tdlib.ChatListMainType {
					offsetOrder = int64(lastChat.Positions[i].Order)
				}
			}
			offsetChatID = lastChat.ID
		}

		// get chats (ids) from tdlib
		var chats, getChatsErr = client.GetChats(chatList, tdlib.JSONInt64(offsetOrder),
			offsetChatID, int32(limit-len(allChats)))
		if getChatsErr != nil {
			return nil, getChatsErr
		}
		if len(chats.ChatIDs) == 0 {
			return allChats, nil
		}

		for _, chatID := range chats.ChatIDs {
			// get chat info from tdlib
			var chat, getChatErr = client.GetChat(chatID)
			if getChatErr == nil {
				allChats = append(allChats, chat)
			} else {
				return nil, getChatErr
			}
		}
	}

	return allChats, nil
}
{% endhighlight %}

## 3 - Dockerfile
To build our Go application with TDLib, we would need all the dependencies. To save your time, I have already prepared the [base image][wcsiu/tdlib] and the Dockerfile is [here][wcsiu/tdlib-Dockerfile].

First, for base images, we need the dependencies and Go compiler.

Go Compiler:
{% highlight docker %}
FROM golang:1.15 AS golang
{% endhighlight %}

Then, we put the dependencies in places as `Arman92/go-tdlib` wants them to be.

Dependencies:
{% highlight docker %}
COPY --from=wcsiu/tdlib:1.7.0 /usr/local/include/td /usr/local/include/td
COPY --from=wcsiu/tdlib:1.7.0 /usr/local/lib/libtd* /usr/local/lib/
COPY --from=wcsiu/tdlib:1.7.0 /usr/lib/x86_64-linux-gnu/libssl.a /usr/local/lib/libssl.a
COPY --from=wcsiu/tdlib:1.7.0 /usr/lib/x86_64-linux-gnu/libcrypto.a /usr/local/lib/libcrypto.a
COPY --from=wcsiu/tdlib:1.7.0 /usr/lib/x86_64-linux-gnu/libz.a /usr/local/lib/libz.a
{% endhighlight %}

We build the Go application.

{% highlight docker %}
RUN go build --ldflags "-extldflags '-static -L/usr/local/lib -ltdjson_static -ltdjson_private -ltdclient -ltdcore -ltdactor -ltddb -ltdsqlite -ltdnet -ltdutils -ldl -lm -lssl -lcrypto -lstdc++ -lz'" -o /tmp/demo-exe main.go
{% endhighlight %}

You might be wondering why I am adding such a long value for `ldflags`. It is linker flag for C compiler. The flag provides locations of the C libraries we need to `cgo` to compile the application.

`-static` is a very important flag here. It tells the compiler to include the c libraries into the final Go executable instead of just linking the Go executable to them. It makes the executable can run without extra dependencies and our next step on minimizing the image size possible.

{% highlight docker %}
FROM gcr.io/distroless/base:latest
COPY --from=golang /tmp/demo-exe /demo-runner
{% endhighlight %}

We move our client executable into a very minimal base image, [distroless][distrolessGithub]. With this extra stage, our image size shrink significantly.

We then expose port 3000 of the container for the HTTP router to be able to receive external call.

{% highlight docker %}
EXPOSE 3000
{% endhighlight %}

Final step, run the executable.

{% highlight docker %}
ENTRYPOINT [ "/demo-runner" ]
{% endhighlight %}

The complete Dockerfile:
{% highlight docker %}
FROM golang:1.15 AS golang

COPY --from=wcsiu/tdlib:1.7.0 /usr/local/include/td /usr/local/include/td
COPY --from=wcsiu/tdlib:1.7.0 /usr/local/lib/libtd* /usr/local/lib/
COPY --from=wcsiu/tdlib:1.7.0 /usr/lib/x86_64-linux-gnu/libssl.a /usr/local/lib/libssl.a
COPY --from=wcsiu/tdlib:1.7.0 /usr/lib/x86_64-linux-gnu/libcrypto.a /usr/local/lib/libcrypto.a
COPY --from=wcsiu/tdlib:1.7.0 /usr/lib/x86_64-linux-gnu/libz.a /usr/local/lib/libz.a

WORKDIR /demo

COPY go.mod go.sum ./
RUN go mod download
COPY . .

RUN go build --ldflags "-extldflags '-static -L/usr/local/lib -ltdjson_static -ltdjson_private -ltdclient -ltdcore -ltdactor -ltddb -ltdsqlite -ltdnet -ltdutils -ldl -lm -lssl -lcrypto -lstdc++ -lz'" -o /tmp/demo-exe main.go

FROM gcr.io/distroless/base:latest
COPY --from=golang /tmp/demo-exe /demo-runner
EXPOSE 3000
ENTRYPOINT [ "/demo-runner" ]
{% endhighlight %}

We can then try to build the image and let's call the image `telegram-client-demo`.

{% highlight shell %}
docker build -fDockerfile -ttelegram-client-demo .
{% endhighlight %}

## 4 - Docker Compose
The main reason we use `docker-compose` here is just to keep the configuration in code.

{% highlight yaml %}
stdin_open: true
tty: true
{% endhighlight %}

However, I do want to talk about these two settings. They allow us to `docker attach` the running container to input through `stdin` for authorization.

The full `docker-compose.yml`.

{% highlight yaml %}
version: '3.4'

services:
  telegram-client-demo:
    build:
      context: ..
      dockerfile: ./Dockerfile
      network: host
    image: telegram-client-demo
    container_name: telegram-client-demo
    hostname: telegram-client-demo
    expose:
      - "3000"
    volumes:
      - "./dev:/demo"
    working_dir: /demo
    stdin_open: true
    tty: true
    ports:
      - 3000:3000
{% endhighlight %}

To run the application, we can just this command.

{% highlight shell %}
docker-compose -fdocker-compose.yml up
{% endhighlight %}

## 5 - Not done yet but almost!
You may find yourself stuck.

{% highlight shell %}
$ docker-compose -fdocker-compose.yml up
docker-compose -fdocker-compose.yml up
Recreating telegram-client-demo ... done
Attaching to telegram-client-demo
{% endhighlight %}

You need to `docker attach` to the running container to do your first time authorization. After that, your credentials would in the `./dev/`. 

{% highlight shell %}
$ docker attach telegram-client-demo
85233333333 #your phone number with country code
Enter code: 
94757
Authorization Ready! Let's rock
{% endhighlight %}

Just enter your account phone number with country code and press enter. Then follow the instructions until it is set.

We can now try to make a `GET` request to our container exposed port 3000.

{% highlight shell %}
$ curl -v http://localhost:3000/getChats
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 3000 (#0)
> GET /getChats HTTP/1.1
> Host: localhost:3000
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Sun, 27 Dec 2020 15:47:48 GMT
< Content-Length: 308
< Content-Type: text/plain; charset=utf-8
<
* Connection #0 to host localhost left intact
{"chatList":["a","b","c"],"total":3}* Closing connection 0
{% endhighlight %}

## 6 - Debug
Besides the `docker logs`, there is error log file in `./dev/` with name `errors.txt`.

# END

Congrats. You got a telegram client which can fetch all you chats.

If you find anything I can improve on this writting, feel free to leave your comments. I am still on my learning journey!

[demoRepo]: https://github.com/wcsiu/telegram-client-demo
[dockerConcepts]: https://www.docker.com/resources/what-container
[createTelegramApp]: https://core.telegram.org/api/obtaining_api_id
[tdlibGithub]: https://github.com/tdlib/td
[cgo]: https://golang.org/cmd/cgo
[go-tdlib]: https://github.com/Arman92/go-tdlib
[wcsiu/tdlib]: https://hub.docker.com/r/wcsiu/tdlib
[wcsiu/tdlib-Dockerfile]: https://github.com/wcsiu/tdlib
[distrolessGithub]: https://github.com/GoogleContainerTools/distroless