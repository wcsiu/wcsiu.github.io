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
Before we start doing any programming, we need to create an app on Telegram first For details, click [here][createTelegramApp]. After this, you should have your `app_id` and `app_hash`.

## 2 - The Client application
To build a telegram client, we need to use a library called [TDLib][tdlibGithub]. It is library written in C++ and so we can leverage [cgo][cgo] to write our client in Go.

[go-tdlib][go-tdlib] is the goto Go library for TDLib. It is being actively maintained. This tutorial will be using it.

### Coding part:
First, we need a TDLib client.

{% highlight go %}
client := tdlib.NewClient(tdlib.Config{
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
func getChatList(client *tdlib.Client, limit int) error {
	if !haveFullChatList && limit > len(allChats) {
		offsetOrder := int64(math.MaxInt64)
		offsetChatID := int64(0)
		var chatList = tdlib.NewChatListMain()
		var lastChat *tdlib.Chat

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
		chats, err := client.GetChats(chatList, tdlib.JSONInt64(offsetOrder),
			offsetChatID, int32(limit-len(allChats)))
		if err != nil {
			return err
		}
		if len(chats.ChatIDs) == 0 {
			haveFullChatList = true
			return nil
		}

		for _, chatID := range chats.ChatIDs {
			// get chat info from tdlib
			chat, err := client.GetChat(chatID)
			if err == nil {
				allChats = append(allChats, chat)
			} else {
				return err
			}
		}
		return getChatList(client, limit)
	}
	return nil
}
{% endhighlight %}

Combine them all the source code would be like this.

{% highlight go %}
package main

import (
	"fmt"
	"math"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/Arman92/go-tdlib"
)

var allChats []*tdlib.Chat
var haveFullChatList bool

func main() {
	tdlib.SetLogVerbosityLevel(1)
	tdlib.SetFilePath("./errors.txt")

	// Create new instance of client
	client := tdlib.NewClient(tdlib.Config{
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
	ch := make(chan os.Signal, 2)
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

	// Wait while we get Authorization Ready!
	// Note: See authorization example for complete auhtorization sequence example
	currentState, _ := client.Authorize()
	for ; currentState.GetAuthorizationStateEnum() != tdlib.AuthorizationStateReadyType; currentState, _ = client.Authorize() {
		time.Sleep(300 * time.Millisecond)
	}

	// get at most 1000 chats list
	getChatList(client, 1000)
	fmt.Printf("got %d chats\n", len(allChats))

	for _, chat := range allChats {
		fmt.Printf("Chat title: %s \n", chat.Title)
	}

	for {
		time.Sleep(1 * time.Second)
	}
}

// see https://stackoverflow.com/questions/37782348/how-to-use-getchats-in-tdlib
func getChatList(client *tdlib.Client, limit int) error {

	if !haveFullChatList && limit > len(allChats) {
		offsetOrder := int64(math.MaxInt64)
		offsetChatID := int64(0)
		var chatList = tdlib.NewChatListMain()
		var lastChat *tdlib.Chat

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
		chats, err := client.GetChats(chatList, tdlib.JSONInt64(offsetOrder),
			offsetChatID, int32(limit-len(allChats)))
		if err != nil {
			return err
		}
		if len(chats.ChatIDs) == 0 {
			haveFullChatList = true
			return nil
		}

		for _, chatID := range chats.ChatIDs {
			// get chat info from tdlib
			chat, err := client.GetChat(chatID)
			if err == nil {
				allChats = append(allChats, chat)
			} else {
				return err
			}
		}
		return getChatList(client, limit)
	}
	return nil
}
{% endhighlight %}

## 3 - Dockerfile
To build our Go application with TDLib, we would need all the dependencies. To save your time, I have already prepared the [base image][wcsiu/tdlib] and the Dockerfile is [here][wcsiu/tdlib-Dockerfile].

First, for base images, we need the dependencies and Go compiler.

{% highlight docker %}
FROM wcsiu/tdlib:1.7.0 AS base
FROM golang:1.15 AS golang
{% endhighlight %}

Then, we put the dependencies in places as `Arman92/go-tdlib` wants them to be.

{% highlight docker %}
COPY --from=base /usr/local/include/td /usr/local/include/td
COPY --from=base /usr/local/lib/libtd* /usr/local/lib/
COPY --from=base /usr/lib/x86_64-linux-gnu/libssl.a /usr/local/lib/libssl.a
COPY --from=base /usr/lib/x86_64-linux-gnu/libcrypto.a /usr/local/lib/libcrypto.a
COPY --from=base /usr/lib/x86_64-linux-gnu/libz.a /usr/local/lib/libz.a
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

Final step, run the executable.

{% highlight docker %}
ENTRYPOINT [ "/demo-runner" ]
{% endhighlight %}

The complete Dockerfile:
{% highlight docker %}
FROM wcsiu/tdlib:1.7.0 AS base
FROM golang:1.15 AS golang

COPY --from=base /usr/local/include/td /usr/local/include/td
COPY --from=base /usr/local/lib/libtd* /usr/local/lib/
COPY --from=base /usr/lib/x86_64-linux-gnu/libssl.a /usr/local/lib/libssl.a
COPY --from=base /usr/lib/x86_64-linux-gnu/libcrypto.a /usr/local/lib/libcrypto.a
COPY --from=base /usr/lib/x86_64-linux-gnu/libz.a /usr/local/lib/libz.a

WORKDIR /demo

COPY go.mod go.sum ./
RUN go mod download
COPY . .

RUN go build --ldflags "-extldflags '-static -L/usr/local/lib -ltdjson_static -ltdjson_private -ltdclient -ltdcore -ltdactor -ltddb -ltdsqlite -ltdnet -ltdutils -ldl -lm -lssl -lcrypto -lstdc++ -lz'" -o /tmp/demo-exe main.go

FROM gcr.io/distroless/base:latest
COPY --from=golang /tmp/demo-exe /demo-runner
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
    volumes:
      - "./dev:/demo"
    working_dir: /demo
    stdin_open: true
    tty: true
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
docker attach telegram-client-demo
{% endhighlight %}

Just enter your account phone number with country code and press enter. Then follow the instruction until it is set.

## 6 - Debug
Besides the `docker logs`, there is error log file in `./dev/` with name `errors.txt`.

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