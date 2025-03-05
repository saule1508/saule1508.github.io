---
layout: post
title: Netbird troubleshooting
published: true
---

How to troubleshoot P2P connections with Netbird by testing webrtc connections with golang
<!--more-->

## Intro

Netbird is a very cool project that makes it easy to set-up a private network. Netbird is open source friendly and they provide a set-up script that lets you set-up netbird on a self-hosted infrastructure very quickly, so that you can play with it. Later you can decide to use the cloud version or continue using self-hosted but with support and enterprise features. Or simply continue with the free self-hosted solution.

At the core of a Netbird VPN network is Wireguard: peers are connected to each other via a wireguard tunnel. To understand wireguard quicky, take the tour ([wireguard](https://www.wireguard.com/#conceptual-overview)). Another key technology that makes peer to peer connections possible is Webrtc. With Webrtc, two peers (each one being potentially behind nat devices and firewall) will manage to establish a direct connection, either directly or - in case it is impossible because of the presence of unfriendly NAT devices -  via a relay mechanism (using an external server, called a TURN server).

When a peer join the Netbird VPN, the Netbird client uses webrtc to connect it to the other peers. It will then show you other peers and how they are connected : via a direct P2P connection of via a relyed connection. Sometimes you want to make sure a direct connection is possible, to avoid the overhead of the relay.

Im my infra I was experiencing  case where direct P2P connections were not possible, so that all connections were relayed. To troubleshoot the issue, and before asking help to the internal network experts, I decided to study the technology, in particular webrtc because it is at the heart of the connection issue. This turned out to be a super learning opportunity, because webrtc involves STUN/TURN (traversal utilities for NAT) which in turns requires a good understanding of NAT. Netbird being written in Go, they use a popular library called PION. PION is an implementation in Go of the webrtc protocol (which is originally a javascript in the browser framework). 

After listening to a lot of videos on youtube, experimenting with webrtc in javascript, looking at the examples in the PION github project, I decided to write my own program in Golang. This was easier than I expected, it gave me a lot of insight on webrtc and let me acquire a few golang patterns in the process.

## How to learn and play with webrtc

With webrtc two peers that want to communicate with each other will use a out-of-band communication mechanism (i.e. not part of the webrtc protocol) to exchange connection candidates. Candidates are basically pairs of ip address and udp port. This out- of-band mechanism is a signaling server. 

So the first thing is to create a signaling server. In golang, to have a basic signaling server, is not very difficult. There are plenty of resources on internet to show you. The more sensible choice is to implement the signal server with websocket but to keep it simple I did it with plain http (with the help of chatgpt !). Even though the code is not long nor involved, there are valuable techniques in golang that I learned. Especially how we can implement a  long polling http request, using a channel to enqueue/dequeue work. A peer will send a get request to the signaling web server, and the signaling server will "block" the request, pulling messages from a channel and sending them to the client as they arrive, without stopping the request. This is very golang idiomatic.

Note: I will certainly re-implement the signaling part using websocket, as it is probably trivial to do, but for now this is largely sufficient for my setup.


There are a lot of good resources on webrtc, here are my some of my favorites

* [webrtc PION](https://github.com/pion/webrtc): this is the library used by netbird, look at the samples. 
* [WebRTC Crash Course](https://www.youtube.com/watch?v=FExZvpVvYxA&amp;t=1282s) (very long but complete explanation of NAT). 
* [WebRTC AP on MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API) 

For testing and learning webrtc I developped two small golang programs, one for the signaling part and one webrtc program (using Go/Pion library) to run on each peer. The signaling server must be started before, it is a basic http server that must be reachable by the two peers (so on public internet in most cases).

[webrtc lab](https://github.com/saule1508/webrtc-lab)

### Signaling server

Once you have cloned the project, you can copy the signaling server code where you want to start it, or make a binary and copy it over

```
cd signal
make build
./bin/signaling-server -port 8080
```
or you can just

```bash
go run signaling-server
```

### webrtc client : answerer

For the webrtc, there is only one program, that can be used on both peers where you want to test connectivity. The program must be started first with the role answerer

```
cd client
go mod install
go run webrtc_client.go -signaling-addr http://127.0.0.1:8080 -role answerer -with-turn=false
```
the go mod install is needed to download the Go/Pion library. You might need to edit the go.mod file depending on the golang version on your system.

Note that I use the with-turn flag to false, in case you don't have turn servers or in case you are only interested in direct P2P connections. Otherwise webrtc will automatically fall back to turn to establish the connection.

On the signaling side, you should see the answerer registrated

```text
time=2025-03-01T10:38:33.091+11:00 level=INFO source=/home/pierre/webrtc-lab/signal/signaling_server.go:34 msg="Starting signaling server" port=8080
time=2025-03-01T10:44:26.699+11:00 level=INFO source=/home/pierre/webrtc-lab/signal/signaling_server.go:66 msg="receive registered" id=answer-peer-1
```

As soon as the webrtc client starts it gathers connection candidates and log them on the console, this is valuable information to understand the process. Also take care that as soon a candidate is found, it is send (via signaling) to the other peer, even before a SDP offer is made. This is called trickle ice. It means a peer can receive candidates before a SDP has been exchanged, it needs to keep them in a temporary structure so that it can add them to the SDP after.

Note: when you use the trickle ICE test ([trickle ICE test](https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice), this is exactly the same info you see as the candidate gathering logged by this program. It just shows (via the help of STUN servers) what ICE connection candidates are discovered. The advantage of doing it in Go is to better understand and also because on servers you usually don't have a browser.
 
 ### webrtc client : offerer

On the second peer you start the same program but with a different role. The offerer is the one sending an offer (a SDP) to the other peer, and in return it receives an answer. 

 ```
 cd client
 go run webrtc_client.go -signaling-addr http://127.0.0.1:8080 -role offerer -with-turn=false
 ```

 On the signaling side, you should see the offerer registrated, and then on both client you can see the local and remote ICE candiate and the offer / answer exchange and finally the datachannel created and used

## Take away

By implementing and running wertc on the server you can troubleshoot the ICE candidate exchange and understand how the exchange of information over signal is performed.

In my infra, I could establish a P2P connection without the help of a turn server, so I knew the issue was not related to webrct. At the end it turned out to be a routing issue in the linux kernel, which enabled me to learn about policy based routing in linux. But this is another story (check here for rule based routing with wireguard [wireguard routing](https://www.wireguard.com/netns/))