# Cable Protocol Comparison

Comparing a project against others is difficult. 

There are selection and favoritism biases, reams of documentation to process
while contending with a lack of foundational knowledge for a given project, and
time pressures.

More importantly, all projects have their own specific nuances and reasons of
existence. The act of comparing necessarily erases these essential qualities.
At the same time, a comparison can be a useful tool to help new people orient
themselves using past experiences. 

The possibility space of comparisons we could make is... quite large. We have
elected to limit the comparison to 1) established protocols, 2) in use today,
and 3) focusing on the same domain: chat. We have also chosen to focus on
projects with openly available implementations and code.

We try to compare Cable to other projects using a set of
criteria that are of foundational importance to our project, while at the same
time we hope the criteria themselves are, mostly, intelligible without a
specialized background.

Make note, however, that all of the mentioned projects are good, useful, and
valuable in their own rights. As are projects we haven't listed in this
comparison for the previously mentioned reasons or due to our lack of knowledge
about them.

_Any misrepresentations of other protocols are good-natured mistakes and we
want to address and fix any such issues, please reach out if you find any!_

> This comparison was part of a grant milestone focused on documentation. The
> grant itself was part of [NGI Assure](https://nlnet.nl/assure), a fund established by
[NLnet](https://nlnet.nl).

One last note before you go on reading this comparison: 

You may benefit from reading about the background and considerations driving
the choices made behind Cable's design as a protocol: [Introduction to Cable
for the interested person](/introduction.md). 

Because the design choices strongly influence this comparison.

## Comparison at a glance

The meaning of each column is described in detail in the next section, which
itself is followed by a section performing a one-by-one comparison treating
each project according to the different criteria.

**Note**: In this table's first couple of rows we compare Cable against our old
approach, called Cabal-HC in this comparison. HC is short for hypercore—the
technology stack we built on when starting out with that implementation back in
2018.

| Name                         | Binary vs Text | Peer-to-peer or server                                    | Clear specifications | Implementability | Single core implementation | Organizational form             | Has delete                                           | Group vs individual                                        |
|------------------------------|----------------|-----------------------------------------------------------|----------------------|------------------|----------------------------|---------------------------------|------------------------------------------------------|------------------------------------------------------------|
| Cable                        | Binary         | Peer-to-peer                                              | Yes                  | High             | No                         | Volunteer-driven project        | Yes                                                  | Group                                                      |
| Cabal-HC (our old protocol!) | Text (JSON)    | Peer-to-peer                                              | No                   | Low              | Yes (Nodejs)               | Volunteer-driven project        | No                                                   | Group                                                      |
| XMPP                         | Text           | Server mainly (p2p via extension)                         | Yes                  | High             | No                         | Ecosystem around IETF standard  | No (yes via extension)                               | Individual (group via extension)                           |
| IRC                          | Text           | Server                                                    | Yes                  | High             | No                         | ecosystem around IETF standard  | No                                                   | Group (per-channel only)                                   |
| Matrix                       | Text (JSON)    | Server                                                    | Yes                  | Low              | No                         | Commercial with venture funding | Yes (no, for some data) \[1\]                        | Group (per-channel mainly)                                 |
| SimplexChat                  | Text (JSON)    | Server                                                    | Yes                  | Medium           | Yes (Haskell)              | Commercial with venture funding | Yes                                                  | Group & individual                                         |
| DeltaChat                    | Text           | Server                                                    | Yes                  | Medium           | Yes (Rust)                 | Grant-driven volunteer project  | Yes & No\*                                           | Group & individual                                         |
| Briar                        | Binary         | Peer-to-peer                                              | Yes                  | Medium           | Yes (Java)                 | Company with grants             | Yes                                                  | Individual-centric with group invites (contact-book based) |
| cwtch                        | Binary         | Server (peer-to-peer 1-on-1 connections possible via Tor) | No                   | Low              | Yes (Golang)               | Non-profit, Canada              | No? (but defaults to not store conversation history) | Group & individual (contact-book based)                    |
| Signal                       | Binary         | Server                                                    | Yes                  | Low              | Yes (Rust)                 | Non-profit, USA                 | Yes                                                  | Group (per-channel only)                                   |

### Summary

In Cable, we've tried to combine the best parts of how the older protocols
worked (high implementability) with the insights and usefulness of newer
projects: group chat paradigms have changed since the 1980s, peer-to-peer is a
strong architectural choice if possible to support, and delete is an important
feature for many reasons.

This is represented in no better place than in the most relevant comparison:
between Cable and our previous implementation of the same idea, Cabal-HC.

## Comparison criteria

* **Text or binary communication**
* **Peer-to-peer or server-based topology**
* **Clear specifications**
* **Implementability**
* **Single core implementation**
* **Organizational form**
* **Delete**
* **Group-centric or individual-centric**

Each criterium has a set of guiding questions, which frame it as one or more questions, and a
motivation describing its inclusion and perceived importance.  In the next few paragraphs we excavate
the meaning of each before going into our comparison proper. 

**Text or binary communication**

[json-canon]: https://www.rfc-editor.org/rfc/rfc8785

Guiding question:

_"Does the protocol use text-encoded messages for communication or does it encode its messages as binary payloads?"_

Why we care about the distinction: 

Passing binary data is more efficient and uses less data than if the exchange
uses a text representation. In certain cases, binary is also easier to
implement (the order of bytes is necessarily both ordered and specified) than
text-based communication. Finally, JSON-specific formats can be difficult to
implement across programming language boundaries depending on whether it is
[canonicalized][json-canon].

**Peer-to-peer or server-based topology**

Guiding question:

_"Is the network topology's central concept device-centric (peer to peer) or server-centric?"_

Why we care about the distinction: 

This criteria is about whether you can feasibly run a full node on any device
(peer-to-peer) or if you need other project-specific infrastructure to exist
and run (server-based) in order to communicate. 

As for why we think that matters: peer-to-peer projects can, and often are,
facilitated by companion server infrastructure or relays, but, at their core
they can function without them (such as in scenarios with serious network
infrastructure degradation or failure). The opposite does not hold true for
server-based projects as they have essential roles, importance, and/or
functionality assigned to their server component.  Moreover, additional
infrastructural and architectural requirements impact project implementability.

Ideally, peer-to-peer projects lowers the barrier for a project to continue
existing as any user can start using the project and provide availability
through having the program open without need of additional expertise or
specialization. BitTorrent is a good example of what the paradigm can
accomplish in terms of project lifespan, resilience, and relevance.


**Clear specification**:

Guiding questions:

* _"Is the protocol behaviour fully described by a set of specifications?"_
* _"Are the specifications all that is required to create an interoperable implementation?"_

Why we care about the distinction:

The existence and availability of clear specifications affects both the
Implementability and Single core implementation criteria. It makes it easier to
grow the ecosystem around the protocol and the pool of stakeholders, interested
parties, and developers. 

Having clear specifications gives a protocol additional life in the event that
its originators move on to other things, making interested parties and
stakeholders legitimate candidates as new project stewards, and enables
rediscovery of the project should it should fall into temporary obscurity.

**Implementability**:

Guiding questions:

* _"Can the protocol behaviour be fully implemented by an independent group reading the available documentation?"_
* _"Is it feasible for small, under-resourced groups to implement the protocol
  in such a way as to achieve full interoperability with the reference
  implementations?"_
* _"Is the protocol behaviour strongly tied to a particular ecosystem or is it ecosystem independent?"_
* _"Is there a divide between what is described by the protocol documentation
  and what implementations require in practice for interoperability?"_

Why we care about the distinction:

Implementability is affected by having a clear and complete protocol
documentation, care taken during the design phase to avoid encoding incidental
platform details and eccentricities, as well as making tradeoffs that constrain
accidental complexity while still allowing for essential complexity.

* Accidental complexity: Depending on the sort order of the v8 javascript
  engine for JSON keys for string serialization. 
* Essential complexity: Choosing UTF-8 as the explicit string encoding to allow
  many different natural languages to be used and represented.

Implementability has an effect on project lifespan by overall impacting its
resilience. It is more difficult to onboard potential developers to a project
with low implementability as the protocol knowledge may be contained within
individual developers rather than externalized in documentation. The tradeoffs
that affect implementability may similarly affect comprehending the system's
parts and how they combine into a whole.

Projects with high implementability have the potential to be implemented using
techniques and languages with high power efficiency, giving new purpose to old
devices. 

**Single core implementation**

Guiding question:

_"Is the protocol mainly implemented by a single core implementation?"_

Why we care about the distinction:

This criteria acts as a measure of protocol implementability. By *»core
implementation»* is meant an implementation that can functionally act as a
stand-in replacement for the reference implementation. 

Low complexity protocols tend to have many implementations that can
interoperate, ensuring the project's survival into the future by spreading
development across many stakeholders.

**Organizational form**

Guiding questions:

_"Does the project require dedicated and continual funding to exist? What kind?
Does the organizational form impose a hierarchy imposed between developers and
users?"_

Why we care about the distinction: 

All things considered equal, a volunteer project is paradoxically less
sensitive to project economical shocks than a commercially run project.
_»Paradoxically»_ because volunteer projects can often be identified by their
lack of funds.

If a commercial company fails to raise funds, that can often be the end of that
project due to bankruptcy proceedings. 

People engaged in volunteer projects derive value from other avenues such as
satisfaction, curiosity, experience, comradery.  Receiving donations can be a
boost and a welcome bonus—especially for dealing with useful/necessary but
onerous tasks—but is often not an existential concern as financial concerns
were not a core consideration when getting involved.

Volunteer projects also function as an organic environment for interested
developers to test out a project and potentially become more involved and
contributing to the project on some level. This can be contrasted with the
budget- and rule-based hiring systems of companies, and the hierarchy and
conflicts which arise over time between company management, paid staff, and
eager volunteer contributors.

**Delete**

Guiding questions:

_"Can you delete your own posts? Can you prevent storing posts you don't want to store?"_

Why we care about the distinction: 

We think it's important for users to have the ability to, as far as possible,
remove content they themselves authored and which they no longer want to be
shared. Information once shared is very difficult (impossible?) to verifiably
retract, but steps in this direction towards "damage mitigation" are still
useful, in particular for scenarios of accidental information sharing. In a
phrase, arbitrary message revocation is what we are looking to determine if
the compared protocols support.

This criteria falls in line with, and supports, the "right to be forgotten".

**Group-centric or individual-centric**

Guiding questions:

_"How is data access structured in terms of the protocol? Is it
group centric like messages in a group chat, or individual-centric like in the
1-on-1 instant messaging paradigm?"_

Why we care about the distinction:

All other factors being the same, a project that services group chat but which
was originally designed for a 1-on-1 paradigm will have a higher degree of
friction during implementation and use compared to projects designed for group
chat from the ground-up. Additionally, any project that supports a
group-centric approach can also be used as a 2 (or 1!) person chat.

## Comparisons one-by-one

[cwtch]: https://web.archive.org/web/20240331154636/cwtch.im
[delta.chat]: https://web.archive.org/web/20240407200944/delta.chat/en
[briar]: https://web.archive.org/web/20240421231946/https://briarproject.org/
[simplex]: https://web.archive.org/web/20240404142515/https://simplex.chat/
[matrix]: https://web.archive.org/web/20240423070017/https://matrix.org/
[irc]: https://en.wikipedia.org/wiki/IRC
[xmpp]: https://en.wikipedia.org/wiki/XMPP
[cable-cable]: https://github.com/cabal-club/cable/
[cabal]: https://github.com/cabal-club/cabal-core/tree/6fe78849aaf7122250f53ed5df2025fe49f1fc11
[signal]: https://web.archive.org/web/20240423204740/https://signal.org/

[Cable][cable-cable]

* **Text or binary communication**: Binary.
* **Peer-to-peer or server-based**: Peer-to-peer.
* **Clear specifications**: Yes. Clear specifications exist and they describe the entire protocol behaviour.
* **Implementability**: High. This has been one of the core considerations driving the protocol's design.
* **Single core implementation**: There are multiple implementations. As of
writing, in addition to the nodejs implementation there exists an
implementation in Rust, as well as partial implementations in Golang, Erlang,
and Elixir.
* **Organizational form**: Volunteer-based. From time to time, the project applies for and receives grants, as well as individual donations from fans of the project. At the end of the day the project's development does not revolve around income.
* **Delete**: yes.
* **Group-centric or individual-centric**: Group-centric.

[Cabal-HC][cabal]

* **Text or binary communication**: Text-based using JSON.
* **Peer-to-peer or server-based**: Peer-to-peer. Relies on a set of hyperswarm boostrap servers and supplemented by Cabal's community-run DHT nodes. Discovery also works over LAN via multicast DNS.
* **Clear specifications**: No. No protocol description or specification exists.
* **Implementability**: Low. We based Cabal ontop of an existing data structure strongly tied to the nodejs ecosystem, called Hypercore. We layered our chat project / protocol, and data model using and on top of the technologies supporting hypercore.
* **Single core implementation**: There is only a single core implementation in nodejs and no documented protocol specification.
* **Organizational form**: (Note: same as Cable) Volunteer-based. From time to time, the project applies for and receives grants, as well as individual donations from fans of the project. At the end of the day the project's development does not revolve around income.
* **Delete**: no.
* **Group-centric or individual-centric**: Group-centric.

[XMPP][xmpp] (_Extensible Messaging and Presence Protocol_)

* **Text or binary communication**: Text-based protocol.
* **Peer-to-peer or server-based**: Server-based. Peer-to-peer support exists via a protocol extension (jingle, XEP-0166).
* **Clear specifications**: Yes. XMPP is thoroughly documented across many specifications and protocol extensions. 
* **Implementability**: High. There is however a great variability in coverage of the different extensions across clients and servers.
* **Single core implementation**: No. Being an IETF standard with a historical precedent, many interoperable implementations and clients exist.
* **Organizational form**: Open source ecosystem around a set of IETF standards.
* **Delete**: has delete via protocol extension XEP-0313.
* **Group-centric or individual-centric**: Individual-centric. Group chat exists via protocol extension XEP-0045.

[IRC][irc] (_Internet Relay Chat_)

* **Text or binary communication**: Text-based protocol.
* **Peer-to-peer or server-based**: Server-based.
* **Clear specifications**: Yes. IRC is documented in a set of RFCs. 
* **Implementability**: High. Some IRC networks have unique quirks and behaviour are not covered by specifications, but the baseline that constitutes IRC can be implemented by small groups in most of all common programming languages.
* **Single core implementation**: No. Being an IETF standard, and more
  importantly due to its high implementability, there are a great many
  interoperating implementations in existence, with new ones being authored to
  this day.
* **Organizational form**: Open source ecosystem around a set of IETF standards.
* **Delete**: No.
* **Group-centric or individual-centric**: Group-centric, only on a per-channel basis.

[Matrix][matrix]

* **Text or binary communication**: Text using JSON-format.
* **Peer-to-peer or server-based**: Server-based. There have been efforts to
  put the server-based architecture into a peer-to-peer packaging but it does
  not seem to be the main offering as of writing.
* **Clear specifications**: Yes. The Matrix protocol is thoroughly documented.
* **Implementability**: Low. This is despite the Matrix thorough protocol
  specifications and different implementations in existence. The reason is
  that the total protocol description exhibits great complicacy across a
  significantly large surface area. The length specification documents describe
  implementation tasks spread across multiple, interdependent layers: the
  Client-Server, Server-Server, Application, Identity Service, and Push gateway
  APIs, respectively. [2] 
* **Single core implementation**: There exists one main implementation, or
  "homeserver", that fully satisfies the Matrix specifications—it is called
  Synapse. There is a second generation implementation being built by the same
  company as drives development, called Dendrite. There is also a third
  implementation, Conduit, that has been considerably funded and is championed
  by New Vector as an alternative community implementation. There also exist a
  few partial implementations by developers not affiliated with the company.
* **Organizational form**: Dual entity: commercial company (New Vector) with
  venture capital funding driving development coupled with a UK 'Community
  Interest Company' (The Matrix.org Foundation).
* **Delete**: Yes, with caveats: it seems that, due to how Matrix resolves state, some data cannot be deleted. [2]
* **Group-centric or individual-centric**: Group-centric per-channel mainly.
  There does exist an additional group context which seems to have been added
  after the protocol's design phase, which allows grouping of channels after
  they have been created.

[SimpleX Chat][simplex]

* **Text or binary communication**: Their baseline protocol, SMP, is text-based. [6]
* **Peer-to-peer or server-based**: Server-based.
* **Clear specifications**: Yes. The protocol behaviour is outlined and described in a set of documents.
* **Implementability**: Medium. While the project documents its protocol across
  a set of documents and concentrates on the SMP server as its core primitive,
  the SMP server protocol appears to be if not tied to Haskell, then at least
  Haskell-focused. [6]
* **Single core implementation**: There is a single implementation in Haskell.
* **Organizational form**: Commercial company with venture capital funding. [8]
* **Delete**: Yes.
* **Group-centric or individual-centric**: Group-centric.

[Delta Chat][delta.chat]

* **Text or binary communication**: Text.
* **Peer-to-peer or server-based**: Server-based using email infrastructure and email addresses as identities.
* **Clear specifications**: Yes. Delta Chat is based on email-based server
  infrastructure, which itself is exceedingly well documented. The project is
  also based on a set of standards which are listed a document by the project.
* **Implementability**: Medium.  Though based on email and linking to the
  standards in use, it is not clear if the referenced standards comprise the
  complete set of specifications required for interoperability. [1] The
  protocol specification [9] positions itself as a "rough" description of how
  it implements chat on top of email, introducing uncertainty as to how an
  independent group may implement the protocol in an interoperable and
  independent manner. 
* **Single core implementation**: There is a single implementation in Rust.
* **Organizational form**: Independent project? Non-profit? Volunteer? It is hard to say from the outside, but Delta.Chat seems to fit more along the lines of "grant-funded volunteer project".
* **Delete**: No. \* Delta Chat does have support on servers to clear out old
  messages, and does support ephemeral messages (self-deleting messages) but as
  of writing it does not support message revocation outside of the ephemeral
  context [3] and based on how email typically works deletion across recipients
  is not a feature that is typically supported.
* **Group-centric or individual-centric**: Individual and group.

[Briar][briar]

* **Text or binary communication**: Binary.
* **Peer-to-peer or server-based**: peer-to-peer.
* **Clear specifications**: Yes. Briar has a clear set of specifications that
  outlines the protocol and its core dependencies.
* **Implementability**: Medium. There remains uncertainty around the heavy
  Java-based codebase and whether there are unspecified but existent
  dependencies that stem thereof, and there is a dearth of known alternative
  implementations.
* **Single core implementation**: There is a single implementation in Java.
* **Organizational form**: Company with grant funding.
* **Delete**: Yes.
* **Group-centric or individual-centric**: Individual-centric with group invite
functionality. Users have to mutually exchange address book information before
they are able to communicate.

[cwtch][cwtch]

* **Text or binary communication**: Binary.
* **Peer-to-peer or server-based**: Server-based (direct 1-on-1 connections possible through Tor).
* **Clear specifications**: No. It was difficult to find a protocol description
  or specification outside of the 2018 whitepaper draft [5].
* **Implementability**: Low. While the code is fully available and the project
  has good user guides in their documentation, the lack of a protocol
  description significantly affects implementability. However, implementability
  does not appear to be a core concern, rather providing the best privacy for
  the project's given constraints are at the forefront.
* **Single core implementation**: There is a single reference implementation in Golang.
* **Organizational form**: Canadian non-profit with yearly funding drives.
* **Delete**: No? cwtch's documentation states that the default behaviour is to
not persist conversation history between sessions. [4] Outside of that message
revocation does not seem to exist, presumably as a choice to deter a false
sense of security as cwtch has a strong security and privacy profile. The
communication protocol however has the property of being deniable, meaning
messages may be forged after the fact by other users as a way of introducing
plausible deniability to any message persisted from a cwtch session.
* **Group-centric or individual-centric**: Individual and group. Users have to
  mutually exchange address book information before they are able to
  communicate.

[Signal][signal]

* **Text or binary communication**: Binary-based protocol communication.
* **Peer-to-peer or server-based topology**: Server-based.
* **Clear specifications**: Yes. Signal's cryptographic protocols are very well
documented. What's needed to implement clients that interoperate with the
documented cryptographic protocols remains unclear.
* **Implementability**: Low. Signal is a large system that accomplishes the
staggering feat of a mainstream private messenger. It is comprised of a
significant set of technologies, including complex cryptographic
implementations, TURN servers, and cloud deployments.  All cryptography and
client software is public, but there is a continued discourse among interested
onlookers as to unpublished server coordination code required to run the
extensive backend driving the system. Even if only the client code and
cryptography routines need to be verified from a security and privacy point of
view, the impact to another actor implementing or spinning up an interoperable
implementation to Signal's system remains.
* **Single core implementation**: Yes. There is a single core Rust implementation at the root of all their clients.
* **Organizational form**: American non-profit with high revenue requirements due to highly paid employees and infrastructure costs.
* **Delete**: Yes.
* **Group-centric or individual-centric**: Group-centric, only on a per-channel basis.

## Notes

[matrix-critique]: https://web.archive.org/web/20240209132655/https://telegra.ph/why-not-matrix-08-07
[delta-thread]: https://support.delta.chat/t/retract-edit-sent-messages/1918
[delta-specs]: https://github.com/deltachat/deltachat-core-rust/blob/main/standards.md
[cwtch-default]: https://docs.cwtch.im/docs/chat/save-conversation-history/
[cwtch-whitepaper]: https://cwtch.im/cwtch.pdf
[smp]: https://github.com/simplex-chat/simplexmq/blob/stable/protocol/simplex-messaging.md 
[matrix-spec]: https://spec.matrix.org/latest/
[simplex-vc]: https://web.archive.org/web/20240327123447/https://simplex.chat/blog/20230422-simplex-chat-vision-funding-v5-videos-files-passcode.html
[delta-spec-spec]: https://github.com/deltachat/deltachat-core-rust/blob/main/spec.md

* [\[1\]][delta-specs] Delta Chat's technologies and set of specifications to support
* [\[2\]][matrix-critique] A critique of Matrix's protocol by user "haru" titled _"why not
Matrix?"_ posted 7, August 2023. See points 1-5 starting with "1. the graph is
append-only by design".
* [\[3\]][delta-thread] Thread on Delta.Chat's forum with activity between 2021-2023 around a message retraction functionality
* [\[4\]][cwtch-default] cwtch does not save conversation history as a default
* [\[5\]][cwtch-whitepaper] 2018 cwtch whitepaper 
* [\[6\]][smp] Simplex Chat's SMP protocol 
* [\[7\]][matrix-spec] Matrix's spec hub 
* [\[8\]][simplex-vc] Simplex Chat's venture funding announcement
* [\[9\]][delta-spec-spec] Delta Chat's 'chat-mail specification'
