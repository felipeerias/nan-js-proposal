# Neighbour Awareness Networking JavaScript API

## Motivation

Neighbour Awareness Networking ("NAN" from here on), also known as Wi-Fi Aware, is a Wi-Fi standard that enables devices to discover and connect directly to each other without any other type of connectivity between them.

NAN works by forming clusters with neighboring devices, or by creating a new cluster if the device is the first one. This clustering behaviour happens at the OS level and applications have no control over it.

Applications using NAN may publish one or more discoverable services and/or may subscribe to one or more of those services. Devices can be both publishers and subscribers at the same time.

A subscriber will be notified when a matching publisher has been discovered, at which point it may exchange short messages or establish a fast bi-directional network connection with the other device.

Neighbor Awareness Networking is available right now for some Android devices using Qualcomm chipsets (Pixel 2 and Pixel 3, with more coming in the near future). There are also implementations by companies like Intel, Broadcom and Marvell.

Android API: https://developer.android.com/guide/topics/connectivity/wifi-aware

## Use cases

+ transfer large files easily and quickly
+ stream high quality media directly between nearby devices
+ optimize Internet bandwidth usage
+ exchange data with nearby people while not connected to the Internet or to other existing infrastructure
+ detect that two users are physically close to each other
+ improved privacy and security, as the data can remain local
+ real-time collaboration with high bandwidth and low latency
+ simpler and more secure communication with IoT devices

## Privacy and security

A naïve approach to the Web NAN API would be to replicate the existing low-level API, providing functionality to:

+ advertise a service
+ receive a notification when a match has been found
+ sending small messages without creating a connection
+ establishing a connection
+ etc.

However, this naïve approach poses severe threats to privacy and security.

First, the website would be notified of every nearby person who is using the same website, without asking their permission. This could lead to surveillance scenarios where websites detect and log everybody that is around the user.

Second, service announcements can be trivially spoofed. In the case of Android, there isn't any check or guarantee on the service name that you are announcing. In a hypothetical scenario, a website could use NAN to advertise as *"com.facebook"* and get matches for people using Facebook in the vicinity.

Third, the existing APIs do not require much in terms of permissions. For example, the Android API only requires permission to obtain the coarse location and to view and change the WiFi state, and applications may carry out NAN operations in the background. For websites, this poses a risk that a website might be doing things behind your back, even when you are not currently using it.

## Proposed solution

The challenge is to balance usefulness and user safety.

To do that, we must rethink how NAN is presented to the developer: instead of advertising and subscribing using services names, websites would discover and connect to specific user sessions.

Upon starting NAN, the website would be assigned a unique session ID. It would receive the session IDs of other users, and would use *those* to discover and connect to other peers.

While this proposal relies on the exchange of user session IDs as a precondition to discovery and connection, it does not mandate a particular mechanism to do so. Two possible options could be:
 - via a secure exchange with the server
 - via a physical tap of the devices, using NFC

## API Overview

The read-only property `Navigator.nan` returns an object of type `NanManager` that is the main contact point with the NAN subsystem.

The caller must being by invoking the `attach()` method which, if successful, will return the identifier for the current session.

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

Invoking the `attach()` method may trigger a permission request to the user, display an ongoing notification, etc.

Once NAN is started and we have received the session ID of the other user(s), we can subscribe to discovery events:

```
navigator.nan.ondiscovered = function(peer) {
    // called when a peer has been discovered
});

navigator.nan.addPeerSessionId(peerSessionId);
```

A discovery event will return a `Peer` object encapsulating the information about a single peer. We use this object to request a connection:

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

After a connection has been established, `peer.baseUrl` will contain the base address of the peer (typically IPv6) which may then be used to exchange data.

### API Example 1

### API Example 2 (with NFC)

## Notes about network-level implementation

The purpose of this section is to show the feasibility of the proposed API by sketching a possible implementation.

### Basics of NAN networking

NAN devices discover each other and create clusters, which are then used to exchange service announcements and messages. Applications don't have any control over this clustering behaviour and are only notified when specific service publishers are found.

Service advertisements in NAN contain the service name (255 bytes) and may contain an optional field with extra information (255 bytes).

If additional security is needed, the specification recommends distributing a secret value among the nodes and using it to replace the service name.

Short messages ("follow-up" in the specification) are 1-to-1 messages containing a payload of up to 255 bytes. Testing shows that they are exchanged at a rate of around 5-10 per second.

Each discovery event or message received includes the MAC address of the originating peer. This MAC address will change periodically (every 30 minutes or so).

In order to establish a connection, both peers need to request it to their operating systems specifying the role of the peer (initiator or responder) and the MAC address of the other peer.

NAN connections use IPv6 link local addresses, for example `fe80::5e:63ff:fefb:be0%aware_data0`.

### Session values

Each new NAN session begins by generating two random values: a `session ID` and a `session secret`.

+ the `session ID` will be used to identify the peer in service announcements
+ the `session secret` will be used to validate connection attempts

### Service publishing

User agents using the NAN JavaScript API will act as both publishers and subscribers.

Only one active session will be advertised at any time. NAN publishing and discovery is paused when the website is in the background.

Service announcements use a fixed service name, e.g. `"org.w3c.example.nan.id"`, and place in the extra info field a hash of their `session ID` and `session secret`.

When the current session is paused or finished, the extra info field is cleared so other peers are notified that the session is no longer active; if no other sessions become active, NAN publishing stops altogether shortly afterwards.

(Another option for the announcements would be to use the `session ID` as the service name, which would have the benefit that only peers that knew it beforehand would be able to discover the node.)

### Discovery

Each peer subscribes to the same service name, e.g. `"org.w3c.example.nan.id"`, and gets discovery events for each nearby peer advertising the same.

Every time that a peer receives a new discovery event or the extra info field of a discovered peer changes, the User Agent will check the hash of its `session ID` and `session secret` with those that had been previously registered.

Only if the hashes match, the User Agent will notify the calling application that a peer has been found.

Publishing and discovery stop when a website is not in the foreground. When the website becomes active again, it will receive the notifications for the discovery events that happened while it was paused, if they are still valid.

### Connection

The implementation must cover two scenarios:

+ *symmetric*: both peers have each other's `session ID` and `session secret`
+ *asymmetric*: following a one-way exchange of `session ID` and `session secret` (e.g. via NFC)

Connection requests are only possible between peers that have discovered each other. They follow this sequence:

+ *initiator* sends a `REQUEST` message that includes the `session secret` of both peers
+ *responder* validates the received secrets
+ if the secrets are invalid, the application rejects the connection, etc., *responder* sends `REQUEST_REJECTED`
+ if the connection is not possible (e.g. internal error, too many active connections, etc.), *responder* sends `REQUEST_DROPPED`
+ if the secrets are valid and the connection is possible, *responder* sends `REQUEST_ACCEPTED`
+ if the request has been accepted, each peer proceeds to create its end of the network
+ when the connection has been established, each peer sends a `HELLO` message containing its IP address
+ when a peer closes the connection, it sends a `BYE` message

### After the connection is established

The standard option for P2P communication once the connection is established would be WebRTC.

However, NAN uses scoped IPv6 addresses that are not currently supported by WebRTC. Relevant bugs and discussion:

+ https://bugs.chromium.org/p/webrtc/issues/detail?id=9978
+ https://bugzilla.mozilla.org/show_bug.cgi?id=1445771
+ https://groups.google.com/forum/#!topic/discuss-webrtc/FlKQafa1Kfo




