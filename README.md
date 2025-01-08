# tiny-ssb-spec

This document aims to specify

1. Design Decisions
2. The Tiny SSB Feed
3. Tiny SSB Peer Interaction


## 1. Design Decisions

Tiny-SSB was born of the question "Could we make [Secure
Scuttelbutt](https://github.com/ssbc) (SSB) **tiny** enough to work over
[LoRa](https://en.wikipedia.org/wiki/LoRa)?"

Core design decisions from SSB that we replicate: - **A.** each device has a unique cryptographic signing key-pair
- **B.** each "message" published by a device is signed by that device's
  cryptographic key
- **C.** each message has a unique ID which can be derived from it's header +
  content (`msgId`)
- **D.** each message published by a device references the msgId of last message
  that device published, such that all messages can be arranged as a linear
  linked-list. This data structure is known as `feed`.

These decisions enable the following properties/ behaviors:
- **reliable "gossip"**: you can receive news from peerA "via peerB" and
  immediately check that whether it is fraudulent / has been tampered with
  (using the authoring device's public key + the message signature).
- **no history edits**: because the feed (linked-list) is grow/append-only, past
  messages cannot be removed, and messages cannot be inserted in the past.
- **easy replication coordination**: because all feeds are linear, it is enough
  to reference feeds public key and the "sequence" (how many messages) to be
  able to coordinate syncing feeds with another peer.
- **no missed messages**: because of the linked-list structure, you know
- **unique refs**: all messages have a unique ID. This cannot be guessed ahead of
  time, which has nice side-effects, such as knowing that all references to
  messages are "backlinks" (point *backwards* in time).

<details>
    <summary>More implications... (click to expand)</summary>

- you can publish to your own feed anytime... you are your own source of
- there is no password reset
    - if you lose your device / the signing keys, there is no recovering them
- your "database" is only a local, subjective snapshot based on what you've
  replicated
    - you will never have all the feeds (islands are ok!)
    - expect partitions / concurrency/ lags
    - expect eventual consistency
- there is no guarenteed ordering of messages
    - there is no central physical (or logical) machine that is "authoring",
      just many parallel peers.
    - the best you can do is "causal ordering" + an algorithm for tie-breaking
    - "timestamps" can work but any malicious device or device with a broken
      clock will wreck your system.
- "multi-device" identity is currently unsolved
    - i.e. people often want to be the same "author" on their phone AND laptop,
      such that people can @-mention, or DM with just one ID, but this is
      currently not possible 
    - you cannot use the same keys on 2 devices safely: if they both publish
      then you can break the "linear linked-list" expectaction of your feed and
      you break replication
- multiple identities per device is easy
    - just add another signing key-pair

</details>

Core design decisons we add for Tiny-SSB:
- **E.** each message must fit in a LoRa packet
- **F.** we assume no guarentee of a "connection" with a peer, there is only
  "broadcast" and "listen"
- **G.** we support "optional" side-chains off the side of a device's feed.



## 2. The Tiny SSB Feed

TODO:

- feed format
- message format
    - cryptographic signature
    - DMX header (computed, deterministic backlink?)
    - payload
- side-chain/ off-chain
- ??



## 3. peer interaction

TODO:

- how to listen
- broadcasting your state / wants??
- broadcasting responses??
- ??


