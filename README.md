# Neighbour Awareness Networking JavaScript API

![NanJS logo](img/nanjs-logo.png)

## Summary

Neighbour Awareness Networking (also known as Wi-Fi Aware) enables mobile devices to discover and connect to each other without requiring any other connectivity or infrastructure between them.

Making this functionality available to websites would enable them to create fast and convenient connections between users who are physically close, opening up new ways of approaching the creation of Web solutions.

However, this can not be done lightly: a careless use of this technology would pose severe threats to privacy and security.

This document presents a draft proposal for a JavaScript API for Neighbour Awareness Networking that balances usefulness and user safety. It achieves this by changing the conceptual model from that of the underlaying network technology: instead of discovering and being discovered by every other node nearby, Web applications would register their interest in individual user sessions and will only be able to discover and conect to those.

The goal of this Web API is to make it easy to discover and connect to people who have allowed you to do so, and only to them.

This document also discusses an example protocol at the network level: this protocol is intentionally kept simple, as its purpose is to show the feasibility of implementing the proposed Web API.

## Use cases

+ transfer large files easily and quickly
+ stream high quality media directly between nearby devices
+ optimize Internet bandwidth usage
+ exchange data with nearby people while not connected to the Internet or to other existing infrastructure
+ detect that two users are physically close to each other
+ improved privacy and security, as the data can remain local
+ real-time collaboration with high bandwidth and low latency
+ local gaming
+ explore large files/datasets with nearby colleagues without needing to upload and manage them in a server, making in-place collaboration more fluid
+ share information between a user's mobile and PC without needing a shared network infrastructure
+ simpler and more secure communication with IoT devices

## Introduction to Neighbour Awareness Networking

Neighbour Awareness Networking ("NAN" from here on), also known as Wi-Fi Aware, is an official Wi-Fi specification that enables devices to discover and connect directly to each other without any other type of connectivity between them.

NAN works by forming clusters with neighboring devices, or by creating a new cluster if the device is the first one. This clustering behaviour happens at the OS level and applications have no control over it.

Applications using NAN may publish one or more discoverable services and/or may subscribe to one or more of those services. Devices can be both publishers and subscribers at the same time.

Service advertisements in NAN contain the service name (255 bytes) and may contain an optional field with extra information (255 bytes).

A subscriber will be notified when a matching publisher has been discovered, at which point it may exchange short messages or establish a fast bi-directional network connection with the other device.

Each discovery event or message received includes the MAC address of the originating peer. This MAC address will change periodically (every 30 minutes or so).

Short messages ("follow-up" in the specification) are 1-to-1 messages containing a payload of up to 255 bytes. Testing shows that they are exchanged at a rate of around 5-10 per second.

In order to establish a connection, both peers need to request it to their operating systems specifying the role of the peer (initiator or responder) and the MAC address of the other peer.

NAN connections use IPv6 link-local addresses, for example `fe80::5e:63ff:fefb:be0%aware_data0`.

NAN is available right now for some Android devices using Qualcomm chipsets (Pixel 2 and Pixel 3, with more coming in the near future). There are also implementations by companies like Intel, Broadcom and Marvell.

Android API: https://developer.android.com/guide/topics/connectivity/wifi-aware

## Privacy and security

A naïve approach to the Web NAN API would be to replicate the existing low-level API, providing functionality to:

+ advertise a service
+ receive a notification when a match has been found
+ sending small messages without creating a connection
+ establishing a connection
+ etc.

However, this naïve approach poses severe threats to privacy and security.

First, the user of the API would be notified of every nearby person who is using the same website, without asking their permission. This could lead to surveillance scenarios where websites detect and log everybody around the user.

Second, service announcements can be trivially spoofed. In the case of Android, there isn't any check or guarantee on the service name that you are announcing. In a hypothetical scenario, a website could use NAN to advertise as *"com.facebook"* and get matches for people using Facebook in the vicinity.

Third, the existing APIs do not require much in terms of permissions. For example, the Android API only requires permission to obtain the coarse location and to view and change the WiFi state, and applications may carry out NAN operations in the background. For websites, this poses a risk that a website might be doing things behind your back, even when you are not currently using it.

## Proposed solution

The challenge is to balance usefulness and user safety.

To do that, we must rethink how NAN is presented to the developer: instead of advertising and subscribing using services names, websites would discover and connect to specific user sessions.

Upon starting NAN, the user of the API would be assigned a unique `sessionId` object, the exact contents of which will depend on the implementation.

The user would receive the session IDs of other peers, and would use *those* to discover, validate and connect to them.

While this proposal relies on the exchange of these session IDs as a precondition to discovery and connection, it does not mandate a particular mechanism to do so. Two possible options could be:
 - via a secure exchange with the server
 - via a physical tap of the devices, using NFC

Another aspect that is not currently covered by this proposal is peer authentication.

### API Overview

The read-only property `Navigator.nan` returns an object of type `NanManager` that is the main contact point with the NAN subsystem.

The user of the API must being by invoking the `attach()` method which, if successful, will return the identifier for the current session.

```
navigator.nan.attach()
  .then((sessionId) => {
    // NAN started
    // sessionID is our identifier for this session
  },
  (reason) => {
    // error while starting NAN
  });
```

Invoking the `attach()` method may trigger a permission request dialog, display an ongoing notification, etc.

Once NAN is started and we have received the `sessionId` of the other user(s), we can subscribe to discovery events:

```
navigator.nan.ondiscovered = function(peer) {
  // called when a peer has been discovered
});

navigator.nan.subscribeToPeer(peerId);
```

A discovery event will return a `NanPeer` object encapsulating the information and state of a single peer.

The user can also get a list of the known peers by calling `NanManager.getPeers()`.

The `NanPeer` object can be used to request a connection:

```
navigator.nan.onconnected = function(peer) {
  // called when a connection has been established
});

navigator.nan.ondisconnected = function(peer) {
  // called when a connection has been interrupted
});

peer.requestConnection().then(
  (peer) => {
    // connection established
  },
  (reason) => {
    // failed to establish a connection
  }
```

Upon receiving a connection request, the other peer will be prompted to accept or reject it:

```
navigator.nan.onconnectionrequest = function(peer) {
  // call acceptConnectionRequest() here with isAccepted=true to
  // accept the connection, or with isAccepted=false to reject it

  navigator.nan.acceptConnectionRequest(peer, isAccepted)
});
```

After a connection has been established, `peer.baseUrl` will contain the base address of the peer, which may then be used to exchange data (the exact mechanism is outside the scope of this proposal).

### Draft API spec

`NanManager`

+ `start()`: attach to the NAN service; returns a Promise that, when succesfully resolved, provides the `peerId` for this user session
+ `stop()`: ends the session and detaches the user from the NAN service (after calling this method, `peerId` will no longer be valid)
+ `subscribeToPeer(peerId)`
+ `unsubscribeFromPeer(peerId)`
+ `getPeers()`
+ `onPeerStateChanged(peer)`
+ `onConnectionRequest(peer)`
+ `acceptConnectionRequest(peer, isAccepted)`

`PeerId`

+ `toBytes()`
+ `fromBytes()`

`NanPeer`

+ `state`: gone, discovered, connecting, connected
+ `onStateChanged()`
+ `unsuscribe()`

### Example 1

+ Web chat with a friend
+ Website detects that friend is nearby
+ Extra funcionality is provided: send files, share camera, stream media…

### Example 2 (with NFC)

+ *sender* and *receiver* open the website
+ *sender* selects the file to transfer
+ *sender* starts a NAN session
+ using NFC, *sender* transfers its `sessionId.public` and `sessionId.secret` to *receiver*
+ *receiver* starts a NAN session, discovers *sender*, and connects to it
+ *sender* transfers the file to *receiver*

[See this video of an Android app](https://darker.ink/static/media/uploads/08_awarebeam_1.mp4) built using a similar approach.

## Notes about network-level implementation

The purpose of this section is to show the feasibility of the proposed API by sketching a possible protocol at the network level. Note that this protocol is just an example and is intentionally kept simple.

### Session values

In this example implementation, each new NAN session generates a `sessionId`containing two internal fields:

+ `sessionId.public` will be used to identify the peer in NAN service announcements
+ `sessionId.secret` will be used to validate connection attempts

This `sessionId` object may be serialised and de-serialised in order to be exchanged between peers.

### Service publishing

User Agents using the NAN JavaScript API will act as both publishers and subscribers.

Only one active session will be advertised at any time. NAN publishing and discovery is paused when the website is in the background.

Service announcements use a service name like e.g. `"org.w3c.example.nan.id"`, and place in the extra info field a hash of their `sessionId.public` and `sessionId.secret`.

When the current session is paused or finished, the extra info field is cleared so other peers are notified that the session is no longer active; if no other sessions become active, NAN publishing stops altogether shortly afterwards.

(Another option for the announcements would be to use the `sessionId.public` as the service name, which would have the benefit that only peers that knew it beforehand would be able to discover the node.)

### Discovery

Each peer subscribes to the same service name, e.g. `"org.w3c.example.nan.id"`, and gets discovery events for each nearby peer advertising the same.

Every time that the User Agent receives a new discovery event or the extra info field of a discovered peer changes, it will get compare the received discovery information with those that had been previously registered.

Only if the values match, the User Agent will notify the user that a peer has been found.

Publishing and discovery stop when a website is not in the foreground.

### Connection

The implementation must consider two scenarios:

+ *symmetric*: both peers have each other's `sessionId`
+ *asymmetric*: following a one-way exchange of `sessionId` from one peer to another, e.g. via NFC

Connection requests are only possible between peers that have discovered each other. They follow this sequence:

+ *initiator* sends a `REQUEST` message that includes the `sessionId.secret` of both peers
+ *responder* validates the received secrets
+ if the secrets are invalid, the application rejects the connection, etc., *responder* sends `REQUEST_REJECTED`
+ if the connection is not possible (e.g. internal error, too many active connections, etc.), *responder* sends `REQUEST_DROPPED`
+ if the secrets are valid and the connection is possible, *responder* sends `REQUEST_ACCEPTED`
+ if the request has been accepted, each peer proceeds to create its end of the network
+ when the connection has been established, each peer sends a `HELLO` message containing its IP address
+ when a peer closes the connection, it sends a `BYE` message

### After the connection is established

The standard option for P2P communication once the connection is established would be WebRTC.

However, NAN uses scoped IPv6 addresses that are not currently supported by WebRTC. More work will be needed on this area. Relevant bugs and discussion:

+ https://bugs.chromium.org/p/webrtc/issues/detail?id=9978
+ https://bugzilla.mozilla.org/show_bug.cgi?id=1445771
+ https://groups.google.com/forum/#!topic/discuss-webrtc/FlKQafa1Kfo


