# An informal Implementor's Guide to Cable
[cable-docs-root]: https://github.com/cabal-club/cable/

Want to learn how to implement a peer to peer protocol from scratch in less
time than it takes to bake a loaf of sourdough bread? Or: How to write a
minimal partial implementation of [Cable][cable-docs-root] in a weekend.

[wire-crypto]: https://github.com/cabal-club/cable/blob/main/wire.md#4-cryptographic-parameters
[nacl]: https://en.wikipedia.org/wiki/NaCl_(software)
[libsodium]: https://doc.libsodium.org/
 
> To go on this journey you will need the following:
> 
> * a programming language - or some way to construct sequences of bytes
> * a set of [cryptographic routines][wire-crypto]. We recommend [libsodium][libsodium] and
>   whatever bindings or libsodium / [NaCl][nacl] libraries exist for your chosen programming language
> * an implementation of [LEB128](https://en.wikipedia.org/wiki/LEB128) - prevalent in most programming languages (we only need the unsigned routines)
> * a copy of the [Cable Wire Protocol][wire]

## Journey plan

For our minimal implementation we will implement 2 **request types**, 2
**response types**, and 2 **post types**. 

This means we're doing a partial (incomplete) implementation of Cable, but this subset is still
enough to build a chat client that retrieve posts from others, can author chat messages, and
even delete self-authored chat messages from communal chat history.

* **Message types**
    * **Requests**
        * Channel Time Range Request
        * Post Request
    * **Responses**
        * Hash Response
        * Post Response
* **Post types**
    * post/text
    * post/delete

_the responses, requests, and posts we will be dealing with in this guide_

If you want the full, authoritative explanation for everything that composes
Cable you are probably looking for the [Cable Protocol
documents][cable-docs-root] and the [Wire Protocol description][wire]. 

This guide comes in play rather as an affable and slightly too talkative (is
the guide drunk??) travel companion. It exists to help you along your journey
of implementing Cable, to guide your focus to points of interests that may
otherwise be vexing for the first-time traveler. A help document complementing
Cable's comprehensive and authoritative specifications. 

Well then, let's go!

## Messages & Posts

The lead stars in the play that is our protocol are **messages** and **posts**.

**Messages** (requests and responses) are how the protocol communication
happens and how peers negotiate information. They are transient and
impermanent: useful for the communication of data but after the sought data has
proliferated, a particular message has no more use.

**Posts** is our collective noun for the data and information being
communicated. A post may represent lines of chat in a channel or deletion
markers for a deleted post. Posts may also set data on a channel (such as a
channel description) or allow a user to define their name. For our minimal
implementation we only concern ourselves with chat text and deletions.

Posts are stored to disk, persisting between sessions, while messages are not.

### Posts: Asking and receiving

**Post Request** is all about asking to receive specific sets of posts. It has
as its primary payload an array of 32-byte BLAKE2b **hashes** whose associated
data is being requested i.e. in `hash = blake2b(post)`: we have the hash and
ask for the post. Often many hashes are being requested, with a hope to receive
their associated posts in response.

**Post Response** is what answers a Post Request and returns an array of posts—the data we requested by hash.

### Request ID: Connecting Responses to Requests

When there is a request and then one or more responses being answered as a
result of receiving that request, we can know that the response(s) and the
request are connected by the **messages** sharing a **request id** (`req_id`)
i.e. field `req_id` is set to the same value for the request and the
response(s).

### What the hash?!

The problem of knowing which hashes to request in the first place is solved by a
few different requests types which cause hashes to be sent in response. Each of
these "hash producing" requests are dedicated to querying remote peers for a
particular set of data, with fields on the request itself that specify the
query. 

**Channel Time Range Request** is what we'll limit ourselves to in our minimal
implementation - which requests hashes on a specific **channel**, and
furthermore tells the **responder** (the client receiving, processing, and
responding to a request) parameters such as the time frame (or, _time range_) a
**requester** is interested in, and whether they want to receive new hashes as
the responder is made aware of new posts.

**Hash Response** answers the hash producing requests (e.g. Channel Time Range
Request) and contains in its response an array of hashes. 

Cable is nifty in that you can compare a request's returned hashes with hashes
you already know of—whether from having recently requested them or because you
already store the posts which produce those hashes. This is how we can
deduplicate data and only request that which is new to us.

### Text und Delete

The two posts we concern ourselves with in this partial implementation are
`post/text`, describing a text post authored in a particular channel, and
`post/delete` which is a deletion of a previous post. A single post has a single
**author** and the author is represented by a [public keypair][wiki-public-key].
In Cable we use [EdDSA Ed25519][wiki-ed25519] keypairs. 

For establishing authenticity of a post—that we may verify a post's author is
indeed who they claim to be—each post has a **signature**. Think of this as an
identifiable flourish of the writer's hand using their favourite ink and
fountain pen—created not by hand but by means of modern mathematical operations
arising from the author using their keypair.  

This is another distinction between messages and posts: **messages are
unsigned** whereas **posts are signed**.

When we send a Channel Time Range Request, what we're asking for are the hashes
of all known posts of types `post/text` and `post/delete` for the requested
channel authored within the requested time range.

### Beware: Bytes

Cable's messages and post types are specified as sequences of bytes. We
structure these bytes according to what we call _fields_: subsequences of a
message or post that encodes a particular piece of data. A correct Cable post or
message must adhere to the order the fields are defined in for a particular post
or message type.

To discern the correct order in which to put bytes into for the protocol's
various message and post types, you will have to consult the [Cable Wire Protocol
document][wire].

As for how to go about implementation: a suggested path is to first make sure
you can construct the requests, responses, and posts you want to support.  Then,
after that is done, move on to thinking about how to answer requests and
constructing indexes. There are examples of correctly constructed binary
sequences available for all of Cable's defined message and post types in the
[`cable.js` documentation][cablejs-example]

If you pursue this path you will be able to **produce bytes** (_serialize_)
from the constructs you use to represent a post/request/response and
correspondingly enable you to read incoming bytes (_deserialize_) into your
constructs. 

The easiest implementation task to start with are the messages, as they lack
signatures and so you can punt on that bit while you acquaint yourself with the
other concepts.

[wire]: https://github.com/cabal-club/cable/blob/main/wire.md
[wire-ctr]: https://github.com/cabal-club/cable/blob/main/wire.md#6324-channel-time-range-request
[wire-msg-header]: https://github.com/cabal-club/cable/blob/main/wire.md#631-message-header

#### Channel Time Range Request

Let's look at the Channel Time Range Request and break it down. 

Looking at Wire Protocol description of [Channel Time Range Request][wire-ctr],
we see the fields specific to this request. 

But, before we start there, we need to actually implement the common fields
for all requests and before _that_, the common fields for all messages. 

Instead, let's start by looking at [the message header][wire-msg-header]. This
set of fields is what any request and any response will start with. The specific
bytes themselves will likely be different for each request and each response,
but the structure remains the same.

By combining the message header and the Channel Time Range Request fields we
see the full sequence of fields that comprise a Channel Time Range Request:

```
MESSAGE HEADER FIELDS
varint  msg_len
varint  msg_type
u8[4]   circuit_id
u8[4]   req_id

CHANNEL TIME RANGE REQUEST FIELDS
varint          channel_len
u8[channel_len] channel
varint          time_start
varint          time_end
varint          limit
```

[uleb128]: https://en.wikipedia.org/wiki/LEB128#Unsigned_LEB128
[wasm]: https://en.wikipedia.org/wiki/WebAssembly

#### Byte types: varints & `u8[N]`

Above we see various "types" mentioned: `varint`, `u8[4]` and `u8[channel_len]`
(or, more generally, `u8[N]` where `N` is a known length). 

Let's dig into those.

##### `u8[N]`

The `u8[N]` notation represents how many unsigned bytes that field takes up,
which we can see from looking at each part of the notation:

* `u` - unsigned
* `8` - 8 bits (1 byte)
* `[N]` - N of those bytes. For example, N=4 bytes for `u8[4] req_id`.

##### Varints

In Cable `varint` is defined to be the same as the [unsigned Little Endian Base
128 (unsigned LEB128)][uleb128] (LEB128) encoding. 

LEB128 lets us encode a variable length field - it could be 1 byte, it could be
3 bytes. Since LEB128 is a requirement for [WebAssembly][wasm], it is highly
likely you can find a library or routine for your programming language to
encode and decode unsigned LEB128 data (henceforth referred to as _varints_).

To transform a number into a varint you would **encode** to unsigned LEB128,
and to read an encoded number you would **decode** from unsigned LEB128 and
then go on your merry way using the decoded value.

It could look like something like this:

```javascript
/* given a module for handling leb128 called leb and we want to encode value 1024 */
/* and that leb.unsigned contains routines for encoding/decoding unsigned leb128 */
const n = 1024 	
// base 16: n is 0x400
const encoded_n = leb.unsigned.encode(n)
// base 16: encoded_n is 0x80 0x08
const decoded_n = leb.unsigned.decode(encoded_n)
// base 16: decoded_n is 0x400 once more
```

#### `msg_len` tricks

Consequently, a correctly constructed Channel Time Range Request will start with the bytes for
field `msg_len` and end with the bytes of field `limit`, and all the fields in between will be
found in the order you see above.  To construct the request you concatenate the bytes of each
field and, in so doing, construct a single sequence of bytes. 

That first field, `msg_len`, has a bit of a hidden trick: to write `msg_len` you have to know
the lengths of all the other fields that follow. Despite `msg_len` being the first field of any
message's byte sequence, it is the last field we write. 

What you can do to resolve this riddle is have one sequence of bytes, call it `buf`, which you
sequentially write bytes to for all other fields: `buf.write(bytes)`. 

Constructing the full & final byte sequence is accomplished by using the length of `buf` to write
`msg_len` and prepending it: 

```
msg_len = varint(buf.length)
finalBytes = concat([msg_len, buf])
```

#### Telling messages apart: `msg_type`

Each message is differentiated from other kinds of messages by field
`msg_type`. Each type of request and response has a static value describing its
type.

These are the values we care about for our partial implementation of 2 requests and 2 responses:

|`msg_type` value 	| Message name 							|
|-------------------|---------------------------|
|0 									| Hash Response 						|
|1 									| Post Response 						|
|2 									| Post Request 							|
|4 								 	| Channel Time Range Request|

Write it by first encoding the value as a varint and then writing the bytes. 

[wire-256-msg]: https://github.com/cabal-club/cable/blob/main/wire.md#631-message-header

In practice, however, all of Cable's currently defined message types have
values below 127 and so their varint representation would not differ from the
unencoded value. The wire protocol does however [reserve the first 256
values][wire-256-msg] for field `msg_type`, and varint encoding values in the
range `[128, 256]` would give byte sequences that differ from the unencoded
values.

#### Bytes of a Channel Time Range Request

Let's make the construction of a Channel Time Range Request concrete by
specifying its values in an example request.

In the below example request we're setting the following values:

* We're asking to receive post hashes for channel `default`...
* for posts authored between timestamps `0` and `100` (fictitious times, for
brevity). 
* We're asking for maximum limit of `20` hashes to be returned.
* The `req_id` was randomly chosen to be `0x95050429`.

The bytes below are expressed in base-16:

```
varint  msg_len             0x15 (= 21 base10)
varint  msg_type            0x04 (= 4 base10)
u8[4]   circuit_id          0x0000 0000
u8[4]   req_id              0x9505 0429

varint          channel_len 0x07 (= 7 base10; num UTF-8 codepoints for "default")
u8[channel_len] channel     0x64 65 66 61 75 6c 74 (= "default")
varint          time_start  0x00
varint          time_end    0x64 (= 100 base10)
varint          limit       0x14 (= 20 base10)
```

[cablejs-example]: https://github.com/cabal-club/cable.js/tree/nlnet-2024-t4-cd?tab=readme-ov-file#examples

Concatenating all the fields gives us the request bytes: 

```
0x15040000000095050429010764656661756c74006414
// or in a representation with space delineating the request's fields
0x15 0x04 0x00000000 0x95050429 0x01 0x07 0x64656661756c74 0x00 0x64 0x14
``` 

For complete examples of all other message and post types, see [`cable.js`][cablejs-example]. 

If we wanted to represent the request using JSON, it could look like as follows: 

(_note: fields in JSON are not arbitrarily orderable. Cable only specifies sequences of bytes and not JSON._)

```json
{
	"msgLen": 21,
	"msgType": 4,
	"circuitid": "00000000",
	"reqid": "95050429",
	"channelLen": 7,
	"channel": "default",
	"timeStart": 0,
	"timeEnd": 100,
	"limit": 20
}
```

### Reading Messages

On receiving a message, we determine what type of message it is by reading bytes until we have
assembled the `msg_type`. Now, this opens the question of how to read `u8[N]` and `varint`
encoded bytes.

For the static unsigned byte types, `u8[N]`, we simply read the specified `N` amount of bytes
and we're done! Varints, however, vary in size. We don't know ahead of time how many bytes to
read!

#### Reading Varints

First, a reminder: for Cable, we define our varints as being equivalent to
[unsigned LEB128][uleb128]. 

When reading a sequence of bytes that should encode a varint—e.g. field
`timeEnd`—we don't know how many bytes to read: it's variable! Here's one
heuristic that could be used for reading and gathering varint bytes: 

* Assume some maximal number of bytes that a legal varint can be represented by
(e.g. a max length of 10 bytes)
* Read until the number of bytes being considered exceeds the maximum size or
until—our preferred situation—we successfully decode a varint value.

Once a varint has been decoded we move onto processing the next field defined
for that message/post type.

#### Reading strings of text

Since Cable deals with chat, we will encounter text. We define all strings as
being UTF-8 encoded, with each string field being preceded by the number of
bytes it takes up. Let's look at field `text` of the `post/text` type:

```
varint				text_len
u8[text_len]	text
``` 

Reading text is then done by first reading and decoding a varint, `text_len`,
representing how many bytes the string is, and then immediately reading that
many bytes, `u8[text_len]`, and interpreting them as a UTF-8 encoded string,
giving us the contents of field `text`.

##### String validation

All strings have maximum and minimum allowable lengths expressed in terms of
Unicode codepoints. Different fields may have different limits, for example a
user's display name may be at most 32 codepoints while a channel name may be at
most 64 codepoints. Note: the number of codepoints in a string is not always
equivalent to the number of bytes for the same string.

Search through the [Wire Protocol][wire] for mentions of _codepoints_ and you
will easily find all of the min & max limits.

As for why string validation works this way, here's what the Wire Protocol
document has to say about using Unicode codepoints over the number of UTF-8
bytes:

> Codepoints are used instead of bytes to try to offset some of the social
> privilege embedded in the UTF-8 encoding. Latin-derived languages get a big
> advantage in UTF-8 when it comes to how many characters they can encode in a
> fixed number of bytes: one can encode 64 English letters in 64 bytes, but only
> 21 hiragana or sanskrit characters.

Unicode is a widely implemented standard. If, in a pinch, you're unable to find
or construct a routine for counting the number of codepoints in a UTF-8 encoded
string, you can use the number of UTF-8 bytes as a soft-ceiling: the number of
UTF-8 bytes of a string will always be equal to or less than the number of
codepoints represented. 

This tactic will be useful in the short-term and allow you to move forward with
your implementation, but you will run into issues while validating received
posts from implementations are able to count codepoints. The strings of
received posts will in some cases exceed the byte counting heuristic while
still fulfilling the codepoint requirements for the given field.

### Post Response

[wire-post-response]: https://github.com/cabal-club/cable/blob/main/wire.md#6332-post-response

[Post Response][wire-post-response] can play tricks on folk so we'll briefly
outline it as well. Post Response starts with the same message header as any
other message, which is then followed by a sequence that describes an array of
posts:

```
MESSAGE HEADER FIELDS
varint  msg_len
varint  msg_type
u8[4]   circuit_id
u8[4]   req_id

POST RESPONSE FIELDS
varint          post0_len
u8[post0_len]   post0
varint          post1_len
u8[post1_len]   post1
...
varint          postN-1_len
u8[postN-1_len] postN-1
varint          postN_len
u8[postN_len]   postN
```

The entire non-header payload is a sequence composed of (`post_len`,
`post_bytes`) pairs; a whole load of posts. 

First in the sequence is the byte length of the first post, immediately
followed by the full sequence of bytes describing that post. This pattern
continues for all the posts being responded with. 

The last entry in the sequence has to be a `post_len` with a value of `0` which
terminates the array of posts, ending the response.

[wire-hash-response]: https://github.com/cabal-club/cable/blob/main/wire.md#6331-hash-response
[wire-post-request]: https://github.com/cabal-club/cable/blob/main/wire.md#6322-post-request

### Hash Response and Post Request

Messages Hash Response and Post Request both operate on arrays of hashes and
thus happen to both end with fields `hash_count` and `hashes`.

Here's how to produce the bytes for those fields: 

* First write the bytes encoding the amount of hashes, `hash_count`, as a varint.
* Then, for as many hashes you have, write each 32 byte hash as it is.
* Yer done!

You should now be appropriately equipped with enough context to implement the
[Hash Response][wire-hash-response] and [Post Request][wire-post-request]
messages documented in the Wire Protocol without further guidance. 

Go forth and we'll see each other again to tackle posts, hashes, and learn about cryptography!

### Moving from Messages to Posts

The procedures described so far for constructing and reading message bytes
holds true for all of Cable's requests and responses. 

We now detour into the post types which, with its signatures and hashing of posts, necessarily
deals with cryptographic operations. The hope is that we exit the detour unscathed and can
continue onto describing strategies for indexing and querying stored posts. 

## Crypto party (not that kind (and no, not that kind, either))

[cablejs-crypto]: https://github.com/cabal-club/cable.js/blob/main/cryptography.js
[libsodium-hashing]: https://doc.libsodium.org/hashing/generic_hashing#usage
[libsodium-keypairs]: https://doc.libsodium.org/public-key_cryptography/public-key_signatures#key-pair-generation
[libsodium-sign]: https://doc.libsodium.org/public-key_cryptography/public-key_signatures#combined-mode
[libsodium-verify]: https://doc.libsodium.org/public-key_cryptography/public-key_signatures#key-pair-generation

For this section we'll assume you're using [libsodium][libsodium]. If you're not, then you
probably know what you are getting yourself into and you can use the following paragraphs as
guidance on aligning the specifics. 

You may be interested in looking at a reference implementation. The javascript implementation
is a useful source demonstrating how to perform the libsodium cryptographic operations we'll be
outlining: [cable.js/cryptography.js][cablejs-crypto]

We'll illuminate how to produce the hashes the protocol puts to good use, the general
routine involved in producing bytes for the post types, including how to generate and
insert the signature which attests to the authenticity and integrity of the post data, and
how to verify an incoming post's authenticity.

### Post Hashing

[wire-blake]: https://github.com/cabal-club/cable/blob/main/wire.md#41-blake2b
[wiki-blake]: https://en.wikipedia.org/wiki/BLAKE_(hash_function)#BLAKE2

Hashes have been mentioned hither and dither—now we're getting into how to
produce them. We use hashes as a fixed-size reference for posts. Hashes, with
their 32 bytes each, are smaller in size than the posts they are a reference
for, and so operating on them for the majority of requests and responses results
in sending less data. 

Hashing functions, in general, have the useful property of producing the same
output for the same input, so we can use hashes to deduplicate data as part of
protocol communication. On receiving an array of hashes in a Hash Response, we
can query the local store using those hashes to figure out which posts we
have—and don't need to request—and which ones we _do not_ have and therefore
want to request in a subsequent Post Request.

For example: 

Let's say we have a `post/text` in all of its glorious bytes (how to produce
those bytes will be dealt with in the next sections). To produce a hash of that
post, we'll take all of the post bytes and run them through the
[BLAKE2b][wiki-blake2b] hashing function. 

Cable's Wire Protocol document provides [BLAKE2b's cryptographic
parameters][wire-blake]; the provided parameters are intended to be identical to
the use of [`sodium.crypto_generichash`][libsodium-hashing].

In a line, this is what we do:

```
hash = blake2b(postBytes)
```

Which looks like this in the javascript implementation:  

```javscript
/* using libsodium in cable.js/cryptography.js */
// allocate the correct size of buffer to store the hash using `b4a` (nodejs Buffer/Uint8Array library)
const hash = b4a.alloc(sodium.crypto_generichash_BYTES)
// run the hashing function on `postBytes` and put the result in `hash`
sodium.crypto_generichash(hash, postBytes)
// `hash` now contains blake2b(postBytes)
```

We'll make it concrete. Let's take the post bytes we'll end up with in a few
sections (spoilers!) and hash it:

```
// postBytes contains the following bytes (base-16 encoded below) 
// 0x25b272a71555322d40efe449a7f99af8fd364b92d350f1664481b2da340a02d06725733046b35fa3a7e8dc0099a2b3dff10d3fd8b0f6da70d094352e3f5d27a8bc3f5586cf0bf71befc22536c3c50ec7b1d64398d43c3f4cde778e579e88af05015049d089a650aa896cb25ec35258653be4df196b4a5e5b6db7ed024aaa89e1b300500764656661756c740d68e282ac6c6c6f20776f726c64

const hash = b4a.alloc(sodium.crypto_generichash_BYTES)
sodium.crypto_generichash(hash, postBytes)
// `hash` now contains blake2b(postBytes) which for the bytes above turns out to be:
// 0x1971c3829f1df088fc2b0a1172174ada80c14650b679587a305dca7b1c396a39

// Look at that difference in size!
```


Now that's what we call hashing! Literally—that's it!!

### Post Production

Like messages, posts have many different fields which are described by byte
sequences of variable length varints or fixed sequence `u8[N]`. Unlike
messages, posts have an explicitly mentioned author—the creator of the post—and
a signature.

[wire-posttext]: https://github.com/cabal-club/cable/blob/main/wire.md#622-posttext
[wire-postheader]: https://github.com/cabal-club/cable/blob/main/wire.md#621-header
[wire-channel-name]: https://github.com/cabal-club/cable/blob/main/wire.md#541-names
[wire-links]: https://github.com/cabal-club/cable/blob/daf50e529fa48764aaafed84e0480993af814529/wire.md#512-links

#### `post/text`
Let's look at the fields describing the [`post/text` post type][wire-posttext].
All posts share a [post header][wire-postheader] which describes fields common
to all post types, so we'll prefix that to our outline:

```
POST HEADER FIELDS
u8[32]              public_key
u8[64]              signature
varint              num_links
u8[32 * num_links]  links
varint              post_type
varint              timestamp   

POST/TEXT FIELDS
varint          channel_len
u8[channel_len] channel
varint          text_len
u8[text_len]    text
```

The majority of these fields can be filled in the same manner as when we were producing
bytes for the requests and responses (see section Channel Time Range Request).

The fields specific to `post/text` are straightforward: a chat message has text
and it is posted to a particular channel. Both types of data are variable length
with documented max sizes [[1]][wire-posttext], [[2]][wire-channel-name].

Field `timestamp` is the time the post was authored as represented by the
number of milliseconds since the UNIX epoch, and the `post_type` tells us what
type of post it is. The post type is defined as `0` for `post/text`.

Fields `links`, `public_key`, and `signature`; they require a bit more explaining.

#### Links to the Past

`links` is best explained by reading the Wire Protocol's [Links section][wire-links]. 

In brief, each post has a field `links` which is an array of hashes. The field constitutes a
_causal proof_, demonstrating that the current post—the post whose field `links` we are
analyzing—has a _happened-after_ relation to the posts referenced by their hashes in `links`.

We can use this relation to partially order chat messages in a way resistant to clock-skew.
Clock-skew occurs when two or more computers have their clocks set in a way that is not
compatible, where e.g. one computer's clock is running ahead of the other. This causes
incompatible expectations when setting field `timestamp`: a post that causally happened before
another post may, due to clock-skew, have a `timestamp` claiming it happened after.

Field `links` is only relevant to set for post types `{ 'post/text', 'post/topic', 'post/join', 'post/leave' }`.

Again: you are best off by having [read the complete explanation][wire-links].
We will return to `links` when writing about indexes in future sections.

As regards producing the bytes to fill field `links`: 

* First write the bytes encoding the amount of hashes being linked, `num_links`, as a varint.
* Then, for as many hashes you are linking to, write each 32 byte hash.

#### Keypairs

[wiki-public-key]: https://en.wikipedia.org/wiki/Public-key_cryptography
[wiki-ed25519]: https://en.wikipedia.org/wiki/EdDSA#Ed25519

In order to do anything with fields `public_key` and `signature` we first need a keypair. 

As mentioned earlier in the guide Cable uses [EdDSA Ed25519][wiki-ed25519] keypairs, a
digital signature scheme used in [public-key cryptography][wiki-public-key]. It is called
a keypair because we have two interrelated keys: one **public key** and one **secret
key**. It is a "signature scheme" because it lets one party create signatures on data that
other parties can independently verify without needing to confidentially transmit a piece
of information.

The public key is what is referenced by field `public_key` and what other peers use to
verify field `signature`.  The secret key on the other hand is used to create field
`signature`. Per its name, the secret key should be kept secret as its possessor can
create and sign any posts for the associated `public_key`, claiming to be them.

The `public_key` can be regarded as a persistent id with which to identify a particular
peer.  Using this type of system gives us an easy way of creating identities without
requiring sensitive credentials or central coordination, as each peer will randomly
generate a unique set of keys with an extremely low chance of generating the same set of
keys.

#### Generating a keypair

In libsodium, an EdDSA Ed25519 keypair can be generated by calling `sodium.crypto_sign_keypair`
and providing two allocated slices of memory: one to store the public key, the other to
store the secret key.

A few lines accomplishing this (rewritten for the guide's context) from the javascript
implementation: 

```javascript
/* allocate buffers using `b4a` (nodejs Buffer/Uint8Array library) */
// allocate a correctly sized buffer for the public key
const publicKey = b4a.alloc(sodium.crypto_sign_PUBLICKEYBYTES)
// allocate a correctly sized buffer for the public key
const secretKey = b4a.alloc(sodium.crypto_sign_SECRETKEYBYTES)

// generate a new keypair and fill the contents of publicKey and secretKey with */
// their respective keys
sodium.crypto_sign_keypair(publicKey, secretKey)
```

#### Signatures, Please!

[libsodium-sign]: https://doc.libsodium.org/public-key_cryptography/public-key_signatures#combined-mode

Now that we can generate a keypair, let's get into the signature business. By calling
`sodium.crypto_sign` we sign all the bytes coming after `signature` and simultaneously
place the finished signature into the right position of the post data. Performing
signatures in this manner is called operating in combined mode.

More details can be found in the relevant section from libsodium's documentation on
[creating and verifying signatures][libsodium-sign].

That means that, yet again, while `signature` is one of the first fields of any` post, it is
necessarily the last field we fill (we need bytes to sign!).

Creating a signature and verifying a signature are quite similar so we will treat them at
the same time, but here are the main differences:

* During signature creation, field `signature` is initially allocated memory but kept
  empty, and only contains the signature after calling `crypto_sign`
* Verifying a `signature` operates on the _public key_, while signing data uses the _secret key_

Creating a signature in javascript:

```javascript
  // `sigAndPayload` contains all bytes after field `public_key`, including the 64 bytes used
  // by field `signature`. note: at this point, these 64 bytes are unallocated and do not
  // contain a signature. this reference is what is used to write the signature directly into the
  // post data
  const sigAndPayload = postBytes.subarray(sodium.crypto_sign_PUBLICKEYBYTES)

  /* 
  * postBytes contains all the bytes of a particular post, including 64 unallocated bytes to which field `signature` will be written.
  * secretKey is the secret key portion of the keypair. 
  */
  // contains all bytes after field signature
  const payload = postBytes.subarray(sodium.crypto_sign_PUBLICKEYBYTES + sodium.crypto_sign_BYTES)
  sodium.crypto_sign(sigAndPayload, payload, secretKey)
  // postBytes now contains the signature
```

Verifying a signature in javascript:

```javascript
  // `sigAndPayload` contains all bytes after field `public_key`, including the 64 bytes used
  // by field `signature`. 
  const sigAndPayload = postBytes.subarray(sodium.crypto_sign_PUBLICKEYBYTES)
  /* 
  * postBytes contains all the bytes of a particular post, including 64 bytes of `signature` to be verified.
  * publicKey is the public key portion of the keypair.
  */
  const payload = postBytes.subarray(sodium.crypto_sign_PUBLICKEYBYTES + sodium.crypto_sign_BYTES)
  // `boolSignatureCorrect` is true if the signature could be verified, false otherwise
  const boolSignatureCorrect = sodium.crypto_sign_open(payload, sigAndPayload, publicKey)
```

### Cobbling together a line of chat: bytes of a `post/text`

Just like the Channel Time Range Request example that provided the bytes to
solidify our understanding, let's do the same thing but for `post/text`!

We'll need a keypair:

```
public_key bytes
0x25b272a71555322d40efe449a7f99af8fd364b92d350f1664481b2da340a02d0

secret_key bytes
0xf12a0b72a720f9ce6898a1f4c685bee4cc838102143db98f467c5512a726e69225b272a71555322d40efe449a7f99af8fd364b92d350f1664481b2da340a02d0
```

The post will have the following contents:

* The contents of the chat message consists of the UTF-8 encoded string `h€llo
world`. This tests our UTF-8 counting abilities due to the use of the
symbol `€` which is 3 UTF-8 bytes compared to the 1 byte of using a regular
'e'. 
* We're posting to channel `default`. 
* The timestamp is set to `80` (fictious but short). 
* We have one link that we're setting, a post hashing to `0x5049d089a650aa896cb25ec35258653be4df196b4a5e5b6db7ed024aaa89e1b3`.
* Field `signature` is derived in the manner described in section _Signatures, Please!_.

As previously, all of these values (including the keypair) come from [cable.js's complete examples][cablejs-example].

Here's what the bytes look like:

```
u8[32]              public_key	0x25b272a71555322d40efe449a7f99af8fd364b92d350f1664481b2da340a02d0
u8[64]              signature	0x6725733046b35fa3a7e8dc0099a2b3dff10d3fd8b0f6da70d094352e3f5d27a8bc3f5586cf0bf71befc22536c3c50ec7b1d64398d43c3f4cde778e579e88af05
varint              num_links	0x01
u8[32 * num_links]  links	0x5049d089a650aa896cb25ec35258653be4df196b4a5e5b6db7ed024aaa89e1b3
varint              post_type	0x00 (= 0 base10)
varint              timestamp   0x50 (= 80 base10, unencoded)

varint          channel_len	0x07
u8[channel_len] channel		0x64 65 66 61 75 6c 74 	(= "default")
varint          text_len	0x0d (= 13 base10)
u8[text_len]    text		0x68 e2 82 ac 6c 6c 6f 20 77 6f 72 6c 64 (= "h€llo world")
```

Concatenating the fields of this `post/text` gives us the following sequence of bytes:

```
0x25b272a71555322d40efe449a7f99af8fd364b92d350f1664481b2da340a02d06725733046b35fa3a7e8dc0099a2b3dff10d3fd8b0f6da70d094352e3f5d27a8bc3f5586cf0bf71befc22536c3c50ec7b1d64398d43c3f4cde778e579e88af05015049d089a650aa896cb25ec35258653be4df196b4a5e5b6db7ed024aaa89e1b300500764656661756c740d68e282ac6c6c6f20776f726c64
```

This post in turn is identified by (or _hashes to_) the hash: 

```
0x1971c3829f1df088fc2b0a1172174ada80c14650b679587a305dca7b1c396a39
```

### Post deletion: bytes of `post/delete`

[wire-cable-moderation]: https://github.com/cabal-club/cable/blob/main/moderation.md

When an author wants to remove a post they previously authored, they can do so
by authoring a post of type `post/delete` referencing the posts to be
deleted by their hashes. Creating a `post/delete` is to a great degree the same
as with the previous `post/text` example, but there are a few differences:

* there is no `channel` field 
* `post_type` is set to 1
* the only other non-post header fields we set are `num_deletions` and `hashes`

Field `deletions` contains the concatenation of the bytes of the hashes of the
posts being deleted (whew! a roller-coaster of a sentence fragment that was)
with `num_deletions` containing the number of hashes. This is entirely similar
to how we've previously acted on fields `links` or `hashes` for example—all
three fields operate on hashes, after all!

#### Delete me not?

The only posts eligible to be deleted for this post type are those authored by
the same author as of the `post/delete` that is, you may only delete posts you
have made, not those made by others. But read on!

For acting on posts authored by other users, the [Cable Moderation Protocol
document][wire-cable-moderation] outlines mechanisms for hiding and dropping
(removing from the post from local storage and not downloading it again) posts
authored by other users. 

One reason for the split is that `post/delete` is unconditionally acted on by
all who receive a validated `post/delete`, while the moderation post types are
applied depending on whether the author of the post is regarded as a moderation
authority.

### Next steps

We've touched on many facets and essentials for implementing Cable. 

In future sections we will outline how to construct indexes which let us
reference the data we have stored and facilitate post retrieval with an eye
towards answering requests as easy as possible. We'll outline how to set the
`links` field appropriately and, deal with some of the nuance embedded in the
request and response.

## Indexing

_»Indexing»_. Creating an index which can be referenced to answer queries. A way
to find information. That's what this section is about.

In Cable, we create indexes to be able to answer the protocol's requests and so
that we can find and present information in a chat client based on the data we
are storing. 

An index can be represented in whatever way you want. As long as your
implementation correctly responds to requests then everything is great as far
as other peers are considered. Indexes aren't mentioned or specified by the
protocol description, but arise all the same as a result of creating an
implementation satisfying the protocol behaviour.

When constructing your indexes and storing information, you have many options
available to you. You could choose a traditional database, maybe a useful smol
one like sqlite3.  Or perhaps you're experimenting with flat files, storing
data in a `.git/` inspired directory tree, using other flat files as indexes to
navigate the directories. Maybe you're doing something completely different.  

Regardless of your approach, there are common problems that will need to be
solved for answering requests. These commonalities are what we will treat now.

### An Index for Channel & Time 

When forming a response to a received a Channel Time Range Request, what is the
requester asking about? Well, they want to know about _hashes_ that were
authored in the given _channel_ and within a _time frame_ delineated by the
timestamps in fields `time_start` and `time_end`. Let's create an index to
solve this problem, and call it by the name _channel-time index_.

In other words, the channel-time index associates a particular **hash** with
the **channel** it was posted to and the **timestamp** for when it was authored. 

We query the index by providing:

* a channel name, and 
* an interval representing timestamps

We receive back a list of hashes, or, if there were no matches, an empty list.

A sketch of this index may take on the following structure:

```
<channel><ts_authored>: <hash>
```

* `<channel>` being the channel name, 
* `<ts_authored>` the timestamp, and
* `<hash>` the associated hash. 

`<channel><ts_authored>` forms a particular key
accessing that hash. 

We can prevent key collisions by, for instance, adding on the additional data
the end of the key, such as the `public_key` of the post author or making sure
the timestamp component is monotonic (only increasing), or using a unique
counter.

### Data Index

When answering a post request, we need to respond with the associated posts.
That means we just need to query our local storage for the posts identified by
the hashes. This index should ideally be the first one we construct, because it
lets us store and reference the data we store. Let's call this the _data index_

In a sketch, the data index could look as:

```
<hash>: <data>
```

and that should be it for answering post requests!

### Indexing posts

For all received posts, we validate that the post signature is correct. If the
signature doesn't validate correctly, we should discard the post. 

Each valid post should then be indexed: once in the data index, so we can find
this post by hash, and then if it was a `post/text` in the channel-time index.
We'll deal with `post/delete` in the next section.

When we receive a `post/delete`, we query all the posts referenced by the
hashes of field `deletions` and check that field `public_key` is the same for
each of those posts as that of the `post/delete` itself. If it checks out, then
remove all the posts from storage (i.e. the data index) and keep track of their
post hashes such that we do not store them again. This necessitates also
removing their post hashes from the channel-time index, and inserting the hash
of the `post/delete` at least once for every channel represented by the posts
that were deleted.

#### Creating a Delete Index

The post type `post/delete` contains at least two different hashes:

* the hash of the `post/delete` itself
* and the list of hashes being deleted, represented in field `hashes`.

In the section above, we described how to index the hash of the `post/delete`
itself (its hash goes into the channel-time index and is indexed once for each
channel represented by the posts being deleted.)

Now we detail how to index the deleted hashes, so that we can prevent
downloading them again or responding with them to received requests.

It is quite simple: we just store the hashes in a special index. When we
receive a post, we heck whether its hash is in this index and if it is, we
discard it!

The delete index could be as simple as:

```
<posthash>: 1
``` 

That is, we just record the post hash. The value `1` is of no great importance.

### Indexing links

We briefly detailed the `links` field earlier in the guide, now we'll treat
how to set them for posts.

The gist of what needs to be done is to track which posts link to which posts,
through the values set in each of their field `links`, on a per-channel basis. 

The newest post being authored sets its links to those posts who, through
querying the index, are discovered to not have anything _linking to them_.
These unlinked posts are called the links "head(s)" of that channel. 

In other words: each channel has its own set of posts regarded to be the
"head", or the latest known post(s). 

Each time we receive a post, we may be receiving a new entry to the set of
heads, if the incoming post is found not to be linked to by anything else.  We
also create a new head every time we post, since the new post by definition is
not linked to by anything else—it didn't exist a few seconds ago, how could it
be linked?!

The only posts that are relevant to link, and for which field `links` should
be set, are posts of type:

* `post/text`
* `post/topic`
* `post/join`
* `post/leave`

#### Creating the links index

The outlined behaviour can likely be satisfied in many different ways, so
here's one.

We want to track three different things.

* The *outgoing links* from a particular post hash
* The *incoming links* for a particular post hash
* The list of current heads for a given channel

**Outgoing links**: This is simply indexing the contents of each post's field
`links` so that it is queryable without having to retrieve the entire post and
then access the field.

**Incoming links**: If post A is the only post that links to post B, we say
that post B has a single incoming link (or, reverse link), namely the link
from A to B.

To determine if a received post is linked or not, we take its post hash and
query the links index's incoming links for the hash. If we do not find any
incoming links for that post hash, it means the post is a head. We then insert
its post hash as an additional entry to the current heads of that channel.

The links index could be structured something like this, with three different
types of keys being used:

```
OUTGOING LINKS

<posthash>: <linked_posthash>

INCOMING LINKS

<linked_posthash>: <posthash>

HEADS

<channel>: [list of post hashes]
```

* `<posthash>` is the post we are actively indexing
* `<linked_posthash>` is one of the `links` entries of the post being indexed
* `<channel>` is the channel field of the post being indexed

From the above index we can see that, each time an outgoing link is being
recorded that we also create an incoming link for the post hash being linked,
recording the relation in the opposite direction.

#### Setting a post's field `links` 

The list of current heads for a given channel grows (as we receive posts which
are not linked by anything else) until we make a post. 

At the point of making a post, we get the full list of heads of the channel
being posted to. The heads are then set on the post as the values for field
`links`. 

We finally empty the current list of heads, and set its only entry to the
hash of the post we just made.

## Parting farewells & open questions

*Takes a deep breath* That was a lot! But perhaps not so much, considering we
now have a communications infrastructure that can be used to keep in touch with
friends and fams. There's just a little bit more before you will be left alone
to craft & send chat messages with gusto. 

We'll outline very briefly the procedure for rendering chat messages using the
indexes, and touch on some open questions.

### Ring Ring, "Hello, who's there?" It's me REQUEST 

Let's get into the overall behaviour of making requests and how to respond to them
using our indexes.

#### Requesting a Channel Time Range

How do do we know which channels to request for, when we don't have Channel List
Request / Channel List Response implemented?

Well, one thing to know is that Cable's chat clients like to start with a
channel named `default` as a known starter channel.  Another thing you could do
is that your implementation could always hardcode its own starting set of
channels. 

A third little tricky strategy is to wait until receiving a Channel Time Range
Request. It, after all is a request we can comprehend, and it contains a
`channel` field! We can use the channel it is requesting and request
information ourselves: that is, after all, a channel some peer cares about! 

#### Responding to a Channel Time Range Request

When we receive a Channel Time Range Request, we query the channel-time index
using the fields of the request to see if we have any hashes matching its
parameters. If we do have hashes to respond with, we limit the array of hashes
to at most `limit`, if the request's field `limit` is non-zero. 

Finally, we craft the response by putting all those hashes in a Hash Response,
taking care to set the `req_id` field to the same value as in the Channel Time
Range Request we are answering. Then we send it on its way!

If `time_end` is non-zero, and we have responded with all the hashes we have
that match the request, there is one thing left to do. We send a last,
concluding, Hash Response. This concluding response has the `req_id` set to the
same as the request, just like before. 

What's different is that we leave field `hashes` empty, signaling _"That's all
I got for ya buddy!"_. This lets the requester know they have received all they
will be getting for that particular request.

#### Requesting Posts & Responding in turn

When we have received a list of hashes from having previously dispatched a
hash-producing request and received answers to it, we want to start asking for
the associated posts.

Using the data index, we make sift the list of hashes we've received to only
contain those hashes we don't know about. Then we dispatch a Post Request for
those hashes!

Answering a Post Request consists of checking the data index for which hashes
one knows about, and then creating a Post Response containing those posts and
dispatching it!

Like previously, the Post Request and its subsequent Post Response must have
the same value set on field `req_id`.

### Chat clients!

[wire-causal]: https://github.com/cabal-club/cable/blob/main/wire.md#513-causal-sorting

This is roughly what you need to do to get posts from your data storage and
displayed like chat messages!

* Get the list of known channels by querying channel-time index for the unique set of channels
* For the currently focused channel, get the latest (by timestamp) N post
  hashes
* Get the associated posts by querying the data index
* Deserialize retrieved posts into a construct you can operate on in the chat client code
* Render the posts in your client using the values found in fields `text`,
  `timestamp`, `links` and `public_key`
* Sort posts using both timestamps and links for optimal chat consistency
* You have a chat, Harry!

For the procedure on how to render posts sorted in a clock-skew resistant
manner using timestamps and links, see the Wire Protocol section [5.1.3 Causal
Sorting][wire-causal]. You will probably need to query the links index and its
incoming / outgoing links to accomplish this feat!

### Open questions and unfinished threads

We haven't detailed how to position ourselves, with indexes and what not, to
answer requests that are to be kept 'alive', which entails answering them over
time as new posts come in, such as when Channel Time Range Request sets field
`time_end` to 0.

[cable-core-live]: https://github.com/cabal-club/cable-core.js/blob/nlnet-2024-t4-cd/live-queries.js

It is not impossible to figure out, far from it! The gist: we need to keep
track of which requests are alive and on which channels. When new posts are
stored we check their channel field (if it is a post type which has it!)
against the channels where we are currently tracking live requests. 

You may find it elucidating to look at the javascript implementation's
generalized solution: [`cable-core/live-queries.js`][cable-core-live].

_"How do I find other peers / what is the solution for networking?"_

Unresolved as of writing! This is our next big research project: to determine
which solution to choose for networking peers across all potential Cable
implementations and programming language ecosystems. 

In the meantime, use a simple tcp server and sockets for local testing. Cable
can, after all, be used over any network connection.

### Next implementation steps

What this humble guide has attempted to show is how to implement a partial implementation of Cable.

For a still partial implementation, but less so, implement all of the above but also:

* **Message types**
    * **Requests**
      * Channel State Request
* **Post types**
    * post/info
    * post/join
    * post/leave
    * post/topic

For complete functionality—supporting the entire Wire Protocol—implement all of the above but also:

* **Message types**
    * **Requests**
      * Channel List Request
      * Cancel Request
    * **Response**
      * Channel List Response

For the whole shebang, implement all of the above and the [Moderation
Protocol][wire-cable-moderation]'s types and posts:

* **Message types**
    * **Requests**
      * Moderation State Request
* **Post types**
    * post/role
    * post/moderation
        * Tip: start by supporting the actions hide user & unhide user
    * post/block
    * post/unblock

[cable-handshake]: https://github.com/cabal-club/cable/blob/main/handshake.md

Additionally, a complete implementation should implement and be able to perform
the [Cable Handshake][cable-handshake].

## The End

Thank you for reading, and I hope you will continue on your journey. Let us
know about any implementations you create!

Stay up to date with our ongoings! We try to send project updates once or twice
a year on our Open Collective page:

[opencollective.com/cabal-club](https://opencollective.com/cabal-club)

Bye!
