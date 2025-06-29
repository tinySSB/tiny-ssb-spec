:warning: DRAFT VERSION!

---

# tiny-ssb-spec


This document outlines the history + thinking which leads us to this point, as
well as getting into the details required to implement the specification.


## 1. Design Decisions

Tiny-SSB was born of the question "Could we make [Secure
Scuttelbutt](https://github.com/ssbc) (SSB) **tiny** enough to work over
[LoRa](https://en.wikipedia.org/wiki/LoRa)?"

Core design decisions we replicate from SSB:
- **A.** each device has a unique cryptographic signing key-pair
- **B.** each "message" published by a device is signed by that device's
  cryptographic key
- **C.** each message has a unique ID which can be derived from it's header +
  content (`msg_id`)
- **D.** each message published by a device references the msg_id of last message
  that device published, such that all messages can be arranged as a linear
  linked-list. This data structure is known as `feed`.

Core design decisons we add for Tiny-SSB:
- **E.** each message must fit in a LoRa packet
- **F.** we assume no guarentee of a "connection" with a peer, there is only
  "broadcast" and "listen"
- **G.** we support "optional" side-chains off the side of a device's feed.


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
- **unique references**: all messages have a unique ID. This cannot be guessed
  ahead of time, which has nice side-effects, such as knowing that all
  references to messages are "backlinks" (point *backwards* in time).
- **new transports**: with LoRa as the benchmark tinySSB can access extremely
  long-distance (~10km), low energy (solar) communication, as well as
  pocket-to-pocket communication with Bluetooth Low-energy.

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


## 2. The Tiny SSB Feed

TODO:

- feed format
- message format
    - cryptographic signature
    - DMX header (computed, deterministic backlink?)
    - payload
- side-chain/ off-chain
- ??


### Packet / Message

<!--
src: https://github.com/ssbc/tinySSB/blob/fee99079ac2820711e1e853cba0e7aacc8765fea/android/tinySSB/app/src/main/java/nz/scuttlebutt/tremolavossbol/tssb/Replica.kt#L378-L403
-->

A "packet" 120 Bytes of data which consists of the following data concatenated in order:

| name           | bytes                   |
| :------------- | :------                 |
| `dmx`          | 7                       |
| `packet_type`  | 1                       |
| `content`      | 48 (or more if chunked) |
| `signature`    | 64                      |

For `content` up to 48 bytes in size, a single packet suffices, for more than that, see "side-chains" later.


#### DMX

The DMX is a header designed to encode all the information required to build up
a feed (signed linked list), while avoiding having to explicitly include the
full `feed_id`, `sequence`, and `prev_message_id` in the message. Instead of linking back with these 3 data points, we link back to the *hash* of those. 

e.g. This means that if you have message 23 for a feed, you will be able to calculate the expected DMX for message 24. When you receive a new packet you can check to see if it matches the DMX for the "next message" in any feed you're tracking.

The DMX is defined as the first 7 bytes of the SHA256 hash of the following
data concatenated in order:

| name               | bytes   | details                                                                         |
| :----------------- | :------ | :--------                                                                       |
| `dmx_prefix`       | 10      | prefix for versioning packet formats (default: `tinyssb-v0`) ?? TODO: Encoding? |
| `feed_id`          | 32      | ed25519 public key                                                              |
| `sequence`         | 4       | the sequence of this packet in the feeds linked-list, in big-endian format      |
| `prev_message_id`  | 20      | the id of the previous message in this feeds linked-list                        |



Pseudo code:
```
dmx_material = dmx_prefix + feed_id + sequence + prev_message_id
dmx = sha256(dmx_material).slice(0, 7)
```
_where `+` denotes "concatenate"_


#### Msg_id

Pseudo code:
```
msg_id_material = dmx_prefix + feed_id + sequence + prev_message_id + dmx +
                  packet_type + content + signature
msg_id = sha256(msg_id_material).slice(0, 20)
```

Notes:
- is 20 bytes
- if `sequence` is 0, then `prev_message_id` is all zeros (20 bytes)


#### Packet Type

The `packet_type` code exists to support different types of packets. Currently
there are only main-chain and side-chain packet types

| code  | meaning             |
| :---- | :------------------ |
| 0     | main chain packet   |
| 1     | side chain packet   |


#### Content

TODO rename payload?

intro + ptr

- intro:
    - <= 28: sz_enc + content + fill
    - > 28: sz_enc + fill
- ptr: 
    - <= 28: ??? empty?
    - > 28: hash of chunks?


WARNING: this is actually more complex
- core chain packets all have similar anatomy
- side chain blob packets have different anatomy
    - no DMX
    - no sig
    - just `content + hash`
    - Currently design in flux!


#### Signature

The Signature is defined as the ed25519 signature of the concatenation of the
following data in order:

| name                | details                                              |
| :------------------ | :------                                              |
| `dmx_prefix`        | (default: `tinyssb-v0`)                              |
| `feed_id`           | 32 bytes                                             |
| `sequence`          | sequence of this packet in the feeds linked-list     |
| `prev_message_id`   | id of the previous message in this feeds linked-list |
| `dmx`               | 7 bytes                                              |
| `packet_type` code  | 1 bytes                                              |
| `content`           | 48 bytes                                             |


The resulting signature will be 64 Bytes:

Pseudo code:
```
signing_material = (
  dmx_prefix + feed_id + sequence + prev_message_id +
  dmx + packet_type + content
)
signature = sign(signing_material, signing_key)
```
_where `+` denotes "concatenate"_


## 3. peer interaction

TODO:

- how to listen
- broadcasting your state / wants??
- broadcasting responses??
- ??

### Wants

Wants packets are ... TODO

A want packet is encoded as the concatenation of:

| section   | bytes                | description                                     |
|:----------|:---------------------|:------------------------------------------------|
| `dmx`     | 7                    | identifies the feeds you're coordinating around |
| `payload` | <= 113               | bipf encoded data                               |
| `padding` | 113 - payload.length | topping the packet up to 120 Bytes              |

```
  |<----------------120------------------>|

     7             x              113-x
  +-----+----------------------+----------+
  | dmx | payload              | padding? |
  +-----+----------------------+----------+
```

#### Want `dmx`

`dmx` is the calculated the first 7 Bytes of the sha256 hash of the string
`"want"` (encoded as :fire: TODO), concatenated with the ordered set of
`feed_ids` you are coordinating around. 

In pseudocode:

```
dmx_material = 'want' + feed_1_id + ... + feed_N_id
dmx = sha256(dmx_material).slice(0, 7)
```

#### Want `payload`

`payload` is the bipf encoded array/ list where the first entry is an `offset`
(how far through the goset are you starting) and all subsequent entries are feed
sequences `feed_j_seq`, `feed_k_seq`, ... etc

In pseudocode:
```
payload_content = [offset, feed_j_seq, feed_k_seq...]
payload = bipf.encode(payload_content)
```

As an example, suppose you decoded `payload_content`:

```
[3, 302, 104, 27]
```

Mapping this against our goset of feed_ids, this tells us:

| `offset`  | `feed_id`   | `feed_seq`        |
| :-------- | :---------- | :---------------- |
| 0         | feed_1_id   | ??                |
| 1         | feed_2_id   | ??                |
| 2         | feed_3_id   | ??                |
| **3** <<  | feed_4_id   | **302**           |
| 4         | feed_5_id   | **104**           |
| 5         | feed_5_id   | **27**            |
| ...       | ...         | ...               |
| N-1       | feed_N_id   | ??                |

#### Want `padding`

`padding` is added (as zeros) to until the size of the packet is 120 Bytes


## Acknowledgements

TinySSB was originally designed by [Christian
Tschudin](https://github.com/tschudin) (aka @cft)


