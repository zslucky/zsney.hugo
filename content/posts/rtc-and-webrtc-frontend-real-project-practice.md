---
title: "RTC and WebRTC Frontend Real Project Practice"
date: 2017-08-02T07:46:13+08:00
draft: false
categories:
  - frontend
tags:
  - rtc
  - webrtc
---

This is a real project practice that I did by using `RTC` and `WebRTC`, I will Introduce the technical points and some important points that I met in development, hope this can help you to know RTC a bit more deep in frontend.

<!--more-->

## Content

- [About Project](#about-project)
- [Technical Summary](#technical-summary)
- [Technical Introducing](#technical-introducing)
  - [RTC Introducing](#rtc-introducing)
  - [WebRTC Introducing](#webrtc-introducing)
- [Full Architecture In My Mind](#full-architecture-in-my-mind)

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
IE 8, 9 (cookies=no) |    no       | xdr-streaming **&dagger;** | xdr-polling **&dagger;**
IE 8, 9 (cookies=yes)|    no       | iframe-htmlfile | iframe-xhr-polling
IE 10           | rfc6455          | xhr-streaming   | xhr-polling
Chrome 6-13     | hixie-76         | xhr-streaming   | xhr-polling
Chrome 14+      | hybi-10 / rfc6455| xhr-streaming   | xhr-polling
Firefox <10     | no **&Dagger;**      | xhr-streaming   | xhr-polling
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

That's it, quite easy right?

### WebRTC Introducing

For WebRTC, it's a quite new API in modern browsers, if you have some project that you kown the user will only use newest browsers then you can try this feature.

For WebRTC, there are many technical points in it, the most important one I think should be Network, as WebRTC based on P2P connection, so how to let 2 peers to know and connect each other is the precondition.

So let's see the P2P connection flow first, then explain for every components.

![P2P](/frontend/p2p.png)

I will use a example flow to explain the work flow for WebRTC:

1. Make sure the both peer are online, we can know this through signal server(which used to register peer info and transfer peer info).
2. Now left peer want to communicate with Right peer, so he should prepare the peer first and add local media stream to it.
3. The left peer will prepare the ICE server in local.
3. The left peer will send request to STUN and TURN server to ask its peer info in public network.
4. After peer info ready, left peer will send the info to signal server, then signal server will forward this info to right peer.
5. When right peer received the left peer's info, he know that left peer want to communicate to him.
6. The right peer will prepare the peer(maybe this action can be done more early) and add local media stream to it.
7. The right peer handled left peer's info as its remote connection info.
8. The right peer will prepare the ICE server in local.
9. The right peer will send request to STUN and TURN server to ask its peer info in public network.
10. After peer info ready, right peer will send the info to signal server, then signal server will forward this info back to left peer.
11. The left peer received the right peer's info, handled the right peer's info as its remote connection info.
12. During this time, ICE server will check valid resources and build connection between 2 peers.

In this flow, there are 4 important things:

 * **`Signal server`** : The server used for peers to exchane signals(which is the peer info here)
 * **`ICE server`** : The server which implemented in browser, used to communicate another ICE server in other browser.
 * **`STUN`** : The server can detect your device's IP address and port.
 * **`TURN`** : This like STUN, but it's a relay server that can relay on 2 devices which behind of the complex network.

 > Here In my poject I used rtc server as the signal server, used some public STUN servers.

In real world, I still met a big challenge with complex network, before release the project, we tested many times in local network, everythings OK, but after deployed out to public network, everyhing are broken, what happened? I'm sure this issue due to the network, but how to determine the issue?

Here is my investigation step(not only for network issue, but also for others):

1. **Check the device status**: This step can be down by using a tool from google, [WebRTC troubleshooter](https://test.webrtc.org/), this tool can help you to know a overview for your device's status.
2. **Check media in local**: This step should input some code to test your local media's status by using Media API and `chrome://media-internals/` devtool.
3. **Check ICE with STUN and TURN**: This step used to verify whether STUN and TURN are working correctly, and whether ICE can gather the STUN and TURN info, this step can be down by using a another tool from google, [Trickle ICE](https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/).
4. **WebRTC devtool**: If all above things are correct, your project may worked. Actually, all browsers which support WebRTC also provide devtool for WebRTC, like `chrome://webrtc-internals` in Chrome, `opera://webrtc-internals` in Opera, `about:webrtc` in Firefox

Finally, I found the my project should use TURN server as a fallback, because STUN server can only help to find same hosted or in public network machines, if the machines which behind the firewall or NAT, it may failed, then I tried to find is there any public TURN server, but unfortunately, looks only a few. So I setup my own TURN server.(like COTURN etc... there have some exsit images in docker hub).

> We must try STUN first, because if 2 peers can connect directly by using UDP or TCP, it should be efficient. TRUN server is a relay server which between the 2 peers, it should be expensive.

Then I solved my issues, everyone can be reached in everywhere.

## Full Architecture In My Mind



Thank you for you reading!