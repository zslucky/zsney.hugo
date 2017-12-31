---
title: "RTC and WebRTC Frontend Real Project Practice"
date: 2017-08-02T07:46:13+08:00
draft: false
categories:
  - frontend
tags:
  - rtc
  - webrtc
links:
  - title: "Baidu"
    link: "https://www.baidu.com"
  - title: "Google"
    link: "https://www.google.com"
---

This is a real project practice that I did by using `RTC` and `WebRTC`, I will Introduce the technical points and some important points that I met in development, hope this can help you to know RTC a bit more deep in frontend.

<!--more-->

## Content

- [About Project](#about-project)
- [Technical Summary](#technical-summary)
- [Technical Introducing](#technical-introducing)
  - [RTC Introducing](#rtc-introducing)
  - [WebRTC Introducing](#webrtc-introducing)

## About Project

This is a project about data visualization and real-time communication with staffs which I monitor some hardwares' status and if any abnormal occurs I need to contact the corresponding staff. So I should transfer the hardwares' status to browser in real time and if any data shows warning or danger, I must contact the corresponding staff immediately, let him to solve the warning. This is a safety production project, right?

## Technical Summary

There are 2 important technicals here:

- RTC (Real-time-communication)
- WebRTC (Web Real-time-communication)

Looks like the same? Why separate these 2 conceptions? In short, the communicatees are different, RTC I used to communicate between browser and server, WebRTC I used to communicate between browser and browser. Actually, they used different technical stack, RTC used websocket or polling like ways, but WebRTC just using the browser API.

Browser to browser communication likes P2P(peer-to-peer) communication, the directly communication is the most efficient way for this kind of communications, so WebRTC can reach it. I think RTC also can take on this work, but it should using server as a middleware, and also need some data transformation, so I think it's not a best choice here.

But we still need RTC to transfer data from server to browsers.

## Technical Introducing

So Let's begin our travel.

### RTC Introducing

There are many libraries now can help us to do real-time-communication, like [socket.io](https://github.com/socketio/socket.io), [sockjs](https://github.com/sockjs) etc... Here I will simply introduce the backend implemention, as there are large number of technical points we can shared for it. it's not necessary in this post, maybe I can create another post for backend implemention.

So before project be deployed to real production environment, we used `spring-websocket` as the development server, because it has a better integration with `sockjs`. Yes, I choose the `sockjs-client` as the browser's library.

There are 2 reasons I choose the sockjs.

1. sockjs can be used as a polyfill for websocket that can solve browser compatibility issues.
2. sockjs can be integrated with different message protocols. (Here only use the Stomp protocol)

**The following table comes from sockjs's doc, we can see the rtc supported ways in different browsers (html served from http:// or https://)**

_Browser_       | _Websockets_     | _Streaming_ | _Polling_
----------------|------------------|-------------|-------------------
IE 6, 7         | no               | no          | jsonp-polling
IE 8, 9 (cookies=no) |    no       | xdr-streaming &dagger; | xdr-polling **&dagger;**
IE 8, 9 (cookies=yes)|    no       | iframe-htmlfile | iframe-xhr-polling
IE 10           | rfc6455          | xhr-streaming   | xhr-polling
Chrome 6-13     | hixie-76         | xhr-streaming   | xhr-polling
Chrome 14+      | hybi-10 / rfc6455| xhr-streaming   | xhr-polling
Firefox <10     | no &Dagger;      | xhr-streaming   | xhr-polling
Firefox 10+     | hybi-10 / rfc6455| xhr-streaming   | xhr-polling
Safari 5.x      | hixie-76         | xhr-streaming   | xhr-polling
Safari 6+       | rfc6455          | xhr-streaming   | xhr-polling
Opera 10.70+    | no **&Dagger;**      | iframe-eventsource | iframe-xhr-polling
Opera 12.10+    | rfc6455          | xhr-streaming | xhr-polling
Konqueror       | no               | no          | jsonp-polling

 * **&dagger;** : IE 8+ supports [XDomainRequest](https://blogs.msdn.microsoft.com/ieinternals/2010/05/13/xdomainrequest-restrictions-limitations-and-workarounds/), which is
    essentially a modified AJAX/XHR that can do requests across
    domains. But unfortunately it doesn't send any cookies, which
    makes it inappropriate for deployments when the load balancer uses
    JSESSIONID cookie to do sticky sessions.

 * **&Dagger;** : Firefox 4.0 and Opera 11.00 and shipped with disabled
     Websockets "hixie-76". They can still be enabled by manually
     changing a browser setting.

So, in theory, we can support all browsers by using `srping + spring-websocket + sockjs + stompjs`, In fact, we know our users will only use IE10+ browsers, looks like this consideration is needless, but if you are working on some projects which need this feature, you will like it~~

Then the client code looks like this:

```javascript
import { Stomp } from 'stompjs'
import SockJS from 'sockjs-client'

// 1. Create the socket
const sock = new SockJS('https://example.com')

// 2. Create the message protocol client
const stompClient = Stomp.over(sock)

// Create connection
stompClient.connect({}, frame => {
  // Subscribe the topic
  stompClient.subscribe('demo-topic', data => {
    // ... doing something with data
  })
})
```

### WebRTC Introducing