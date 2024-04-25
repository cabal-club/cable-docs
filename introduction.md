# Introduction to Cable for the interested person
We will try to answer one question:

_Why on earth would you be interested in this?_ 

The answer is spread across three focal points and in how they converge and interplay.

### Protocol focus

What Cable, as a protocol and by extensions the chat clients built for it, is and pays attention to:

* Cable is focused on peer-to-peer private group chat across topic-based channels held within the group
* Usable when the internet is not
* Chat groups are shared privately via friends, not a global index
* Delete posts you regret making from your and other devices
* Retrieve posts written when you were off doing something else
* Only store as much data as you ask for
* No append-only log, no eternal growth requirement
* Technically able to use the same identity on multiple devices
* You and your friends are "the cloud": posts within a group propagate and are stored across the group's members

### Technical focus

These following paragraphs try to explain the technical aspects we have focused on when
designing and thinking about Cable as a protocol.

**Simple architecture** 

The protocol should be constructed in such a way that it is feasible for
under-resourced groups (e.g. students, volunteers) to implement using any
programming language. Even partial implementations should be useful and able to
communicate with more complete implementations.

**Long term sustainability** 

The design priorities focus on group chat while ensuring ease of implementation
and long-term maintenance. We also want the protocol to be and remain usable
even on lower-end computing devices.

**Availability should compound not exclude**

Contributing to a chat's availability should be possible for any member, whether
on their device or by a long-running dedicated session executing on their own
hardware. All efforts should combine to provide greater availability.

**Peer-to-peer-centric** 

Group members storing data and acting as peers are the network topology's central concept, not
servers communicating with servers.

**Request-response instead of grow-only logs**

Cable does not contain a "grow-only log" (aka append-only log) data structure
as a means to accomplish data synchronization. Instead it has a request and
response structure that allows simple data querying between connected peers as
a mechanism to discover, retrieve, and share posts that happened while offline. 

This has been an important consideration as one of the design goals was to
enable post deletion in the protocol and its clients. Finally, there there is
nothing preventing the use of the same Cable identity (identified by a keypair)
on multiple devices.


### Social focus

The following facets are how we have considered the interplay and impact of
protocol on users and vice versa, as well as those considerations we as
developers take in relation to the project.

**Empower users**

Any member can create a new channel within the group; take involved discussions
into the side rooms and let those interested to continue discussing join you.

We provide clear moderation tools and make them available to all users. Nobody
should be shut out from hiding users they find onerous, hiding individual posts
from view, or from what deciding what they store on their device.

**Step away from social centralization** 

We do not want communities to fall over from burned out server administrators,
nor due to the whims of incidental group founders. We do not expect that
because someone can run digital infrastructure they are also gifted in conflict
mediation.

Chats sync across and between users, and the moderation system is subjective and
enables assigning and unassigning admins and moderators to those users who have
garnered trust in dealing with thorny social problems.

**Step away from financial centralization**

As a project, we're volunteer-focused and volunteer-run. We aim to provide a
protocol that can outlast our own active involvement.

**Step away from ecosystem centralization** 

We do not want to tie ourselves to a single upstream technology or programming
language ecosystem—different ecosystems provide different benefits and
drawbacks, and they all have enthusiasts. 

Cable's existence should not hinge on the decisions of opaque upstream choices
and it should be possible—and feasible—to build and run whether you favor
gophers, crabs, rabbits, or something completely different.

## Conceptual Protocol Description

> **Cable in a nutshell**
> 
> The Cable protocol sees members of a private group chat collaborate in a
> decentralized and direct way to share chat history through the sending of
> **requests** for **posts**, which are answered by **responses** using data stored
> locally and matching a received request.

Cable is a peer-to-peer protocol for private group chats where users
collaborate with each other to exchange data. 

The boundaries of a private group chat (a cabal) are delineated and identified
by a secret joining key (the cabal key). 

A joining key is randomly chosen on creating a new group chat. To join the
chat, prospective members have to possess the joining key—whether receiving it
by post card, verbally, online through another program, or other means. All
communication and posts stay within the group's boundaries. Members do not need
to be online to post to the chat, they'll simply sync up whenever they make a
connection to another member.

The design of Cable makes it a binary pull-based protocol:

* **Binary**: the protocol sends binary data, defined as bytes of varying sizes,
  representing protocol-specified fields of data.
* **Pull-based**: data is only sent when others have asked us for it, rather
  than always pushing it out. Although new data may be sent immediately if it
  matches a request we've received.

### Cable Anatomy: Wire, Handshake, Moderation

[intro-cable-root]: https://github.com/cabal-club/cable/
[intro-cable-wire]: https://github.com/cabal-club/cable/blob/main/wire.md
[intro-cable-handshake]: https://github.com/cabal-club/cable/blob/main/handshake.md
[intro-cable-moderation]: https://github.com/cabal-club/cable/blob/main/moderation.md

Cable's functionality is divided across three [protocol descriptions][intro-cable-root]. 
The [**Cable Wire Protocol**][intro-cable-wire] describes the data model and how to exchange data. 
We treat how to establish a secure channel between two peers in the [**Cable
Handshake**][intro-cable-handshake] description.
Finally, the [**Cable Moderation Protocol**][intro-cable-moderation] layers a
system for content moderation on top of the Cable Wire Protocol.

When we refer to Cable, we refer to these three protocols acting in unison but
in particular to the data exchange described by the Cable Wire Protocol, which
is the bedrock the other two descriptions exist to enable.

### Three Core Concepts

Cable has three core concepts: **Requests**, **Responses**, and **Posts**.

**Peers**, users running a client implementing Cable and connected to other
users for a particular group chat, send requests for either posts or hashes of
posts.

**Posts** is our collective noun for the data and information being
communicated. A post may represent lines of chat in a channel or deletion
markers for a deleted post. Posts may also set data on a channel (such as a
channel description) or allow a user to define their name. 

**Hashes** are fixed-sized references of posts, derived by running a post
through a hashing function (BLAKE2b). Requesting hashes lets peers deduplicate
what posts might be sent in a response based on what they've already received
from other peers and what posts they already have stored locally. This makes
the protocol relatively efficient in terms of using less data.

Posts are signed by their author, allowing us to confirm that post contents
have been unchanged and that the claimed author is correct.

When it comes to **requests**, there are different types of requests depending
on what a peer wants to know.  

Two requests ask for explicitly specified pieces of data (Hash Request, Post
Request) while the other requests ask for data matching a set of parameters.
Peers may ask others for what channels others know, or for all posts made in a
certain channel in a certain timespan, _"Give me all the messages made in the
past two weeks in channel baking."_ Certain requests may be long-lived, those
requests ask responders to send new responses as they are made aware of new
posts matching the request parameters. 

Receiving a request from a peer for which you have the associated data results
in replying with at least one **response**.  The response contains either hashes
or posts.

One way we can think of the flow of posts and hashes across peers is as a kind
of request-response cycle, which is initiated by a "hash producing" request.

#### Request-Response cycle

The request-response cycle consists of the following phases:

* Requesting hashes
* Responding with hashes
* Requesting posts
* Responding with posts
* Storing and indexing posts

```
time    Requester                                   Responder

   1    →→→ [req_id α] Channel Time Range Request     
   2                                                <receives request α>
   3                                                ←←← [req_id α] Hash Response 
   4    <receives response α>                        
   5    →→→ [req_id β] Post Request                 
   6                                                <receives request β>
   7                                                ←←← [req_id#β] Post Response 
   8    <receives response β>                       
```
*request-response cycle initiated by requester sending a channel time range request. each request and response pair is identified by the value set on field req_id*

**Requesting hashes**

A peer asks to receive hashes for some purpose by making a "hash producing"
request. For example, the **Channel Time Range Request** is used to ask for
hashes related to a channel limited to a particular time range. 

**Responding with hashes**

Having received a "hash producing" request, and knowing of hashes answering it,
responders send a **Hash Response** with hashes matching the request and within
the parameters it defines. 

**Requesting posts**

The requester receives the Hash Response and uses its hashes to query their
local store and determine which hashes correspond to posts they don't have. The
requester then fires off a **Post Request** to request those posts. 

**Responding with posts**

Responders receive the Post Request and reply with those posts they currently
have stored which correspond to one of the hashes in the request. As their reply
they send a **Post Response** back to the requester.

**Storing and indexing post**

The **Post Response** reaches the requester. They check that the posts correspond to hashes
they had requested, finally storing and indexing the posts. The cycle has been completed.

## Exploring more

When it comes to learning about Cable, there exists more than what we have written in this introduction. 

For example, you may be interested to [read the protocol specifications
directly][intro-cable-root]. 

If not that, then perhaps you are the kind of person who learns by doing
things, in which case you may want to [check out the slightly zany Cable implementation
guide][MISSING_LINK].

In any case, we hope you have had an interesting time reading. If you want to
stay up to date we try to send project updates once or twice a year on our Open
Collective page:

[opencollective.com/cabal-club](https://opencollective.com/cabal-club)

Thanks for stopping by!

