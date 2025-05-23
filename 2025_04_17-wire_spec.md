# tinySSB Wire Protocol Spec | 2025-04-17

present: Nanomonkey, Kevin, cft


Table of Contents
- 1 Goset
- 2 WANT
- 3 CHNK
- ...

## 1) GOSET

Context: There is a paper that cft presented at the DICG workshop in 2022, link is https://dicg-workshop.github.io/2022/papers/tschudin.pdf

Rationale: in the sync protocol (comparing what peers have and what they are missing), using 32Byte public keys (= Feed ID or FID) is killing the protocol if it wants to have packets of 120 or less bytes.

Hence we need a compression mechanism where peers can refer to public keys (name of the append-only log) by short numbers.

We use a grow-only set (GOSET), where the public keys are lexigographically sorted. Hence peers sharing the same GOSET can now refer to a feed ID (FID) by its index into that list.

Having synced on the GOSET is a prerequisit for peers to talk the replication protocol.

### Claim

The GOSET protocol has only one message type: CLAIM

The purpose for CLAIM messages is to sync nodes on a common GROW-ONLY-SET (GOSET).
With Claims, nodes which lack entries can respond with their claim (for the same region which lacks a FID), so that other peers can send claims for two sub-regions, helping to narrowing down where a FID is missing in the set.

SOMEBODY-SHOULD-CHECK: does a reply to a claim "shed" the outer FIDs? Would make sense.

A claim has the format:

[GOSET_DMX_CONST (7 bytes) | 'c' (1 byte)| lowest FID (32 bytes) | highest FID (32 bytes) | XOR (32 bytes) | cnt (1 Byte) ]

A claim covers a region of the lexicographically sorted GOSET. It gives the lowest FID and the highest FID of that region, plus the XOR-sum of all the FIDs in the region. The size field holds the length of the region (number of FIDs included in the XOR). The size is (currently) encoded as one byte, limiting the size of the GOSET to 255.

Total length in Bytes: ```7 + 1 + 3*32 + 1 = 103```

GOSET_DMX_STR = "tinySSB-0.1 GOset 1"
GOSET_DMX_CONST = first 7 bytes of SHA256(GOSET_DMX_STR)

### Novelty
[GOSET_DMX_CONST (7 bytes) | 'n' (1 byte) | FID (32 bytes)]
(this is optional and just an optimization, we can disregard for now. It's used when a peer learns about a new FID from outside the clique which runs the GOSET protocol).

### Zap

Zap messages were introduced for debugging i.e., resetting all peers that hear each other to zero. This is not used or developed/maintained anymore.

['z'| index in network byte order (big endian?...htonl(index))]


## 2) WANT Vector

WANT vectors serve two purposes: announce to others where our latest sequence number is for some feeds (hoping that others will help out), and to wait for new log extensions.
In the Android app, we use the highest announced sequence number to derive a guestimate how many log entries we lack.

### DMX
Where DMX = DeMultipleX field

"want packet" := [WANT_DMX|Payload]

WANT_DMX is a dynamic value, depending on the latest GOSET_state. WANT_DMX is computed as follows:

DMX_PREFIX := b'tinyssb-v0'
WANT_STR := b'want'
GOSET_state := XOR of all feed IDs

WANT_DMX := first 7 bytes of SHA256(WANT_PREFIX | WANT_STR | GOSET_state)


### Payload of a WANT vector

The WANT vector has one offset integer plus a variable number of integers:
[ offs, s0, s1, ... sk ]

where
- sN is the first sequence number (we want) of the feed which has index value ```offs+N``` in the GOSET
- offs is chosen by the sende such that we cover all feeds (their sequence numbers may not fit into one WANT message)

- bipf 

## 3) CHNK Vector

### CHNK_DMX
first 7 bytes of  sha256 hash of:
  dmx_prefix (b'tinyssb-v0') + b'blob' + GOSet "state" (32 bytes XOR of all feed id's)

"chnk packet" := [CHNK_DMX|Payload]

CHNK_DMX is a dynamic value, depending on the latest GOSET_state. CHNK_DMX is computed as follows:

CHNK_PREFIX := b'tinyssb-v0'
CHNK_STR := b'blob'
GOSET_state := XOR of all feed IDs

CHNK_DMX := first 7 bytes of SHA256(CHNK_PREFIX | CHNK_STR | GOSET_state)


### Payload

- feed_index (int) + seq# (int) + cnr?

### Payload of a CHNK vector

[ (f1,s1,c1), (f2,s2,c2), ... (fn,sn,cn) ]

Each triplet specifies the "name" of a missing chunk, which needs a FID, a sequence number, and the chunk number. The integers, tuples, and the outer sequence is encoded with BIPF.


## 4) Main Chain Packets

In response to a WANT vector, a peer may decide to send a series of main chain packets that start with the requested sequence number. Typically, 3 main chain packets are sent (if available) before the other side has to issue a new WANT vector.

The following description of the wire format is for completness, and copied from Section FIXME:

- DMX: first 7 bytes of sha256 hash of 
  [dmx_prefix + feed_id + sequence_num + previous hash]
- Feed ID: ed25519 public key (32 bytes)
- signature: 64 bytes
- [DMX (7 bytes) | '\x00' (type byte)| payload (48 bytes bipf encoded)| signature (64 bytes)] (for side-chain-less packets)
- [DMX (7 bytes) | '\x01' (type byte)| payload (48 bytes) | signature (64 bytes)]
  with payload: varlength(littleEnding) + sidechain_pointer

scenario 1: long content
VL | CONT1 | PTR (total of 48 Bytes)
where:
VL = varlength
CONT1 = first content bytes of the sidechain
PTR = 20 bytes hash pointer

scenario 2: short-enough content
VL | CONT | PADDING (total of 48 Bytes)
If the varlen plus content fit in less than
48 bytes, no pointer is used. Padding is
necessary to make the payload 48 bytes.

scenario 3: not-short-enough content

if VL+content is longer than 48 bytes, you must use scenario 1.

## 5) Side Chain Packets

In response to a CHNK vector, a peer may decide to send a series of side chain packets that start with the requested sequence+chunk number. Typically, 3 side chain packets are sent per feed (if available) before the other side has to issue a new CHNK vector.

- format of a side chain packet:
 [content (up to 100 bytes)| pointer_hash (20 bytes)]
- last message has all zero pointer hash, while the content is padded to bring it to 100 bytes

Note on how to construct the side chain packets:

a) cut the content in two pieces: the first one is what fits into the main chain packet (be careful to consider the lenght of the varlen field); the second part is the rest of the content
b) pad the "rest of the content" so that it is a multiple of 100 bytes, let's call them "content fragments"
c) start with the last fragment, add 20 zero bytes - this is the last side chain packet
d) compute the hash of that last side chain packet
e) pick the second-last fragment, add the hash of step (d) - this is the second-last side chain packet
f) repeat with (e) until you have worked backwards on all fragments
g) the last hash value goes into the main chain packet.


## 6) Peer Neighborhood Protocol (LoRA only)

The "peer messages" are currently only used for LoRA in order to give awarness who is around. It has no role in the synchronization of data and is a mere smoke test, in most cases.

There are two constant DMX values used (regardless of GOSET or other peer state): one for sending a ping Request, the other for sending back a ping Reply.

Only implemented in LoraMesh (tinySSBlib/peers.cpp)

### Ping Request

PING_REQ_DMX = first 7 bytes of SHA256(b'peers 1.0 request')

The payload of the request has the following format:

```"Q $MYMAC t=$T gps=$GPS"```

The GPS entry is optional, the $GPS value is a comma separated triplet LAT,LNG,ALT (altitude is in meters), as provided by tinyGPS++ library.


### Ping Reply

PING_REP_DMX = first 7 bytes of SHA256(b'peers 1.0 reply')

The payload of the reply has the following format:

```"R $MYMAC t=$T rssi=$R snr=$S gps=$GPS [$REQUEST]"```

The GPS entry is optional, the $GPS value is a comma separated triplet LAT,LNG,ALT (altitude is in meters), as provided by tinyGPS++ library.


## Appendix: TODOs

- make GPS coordinates in ping request and reply optional/configurable, for privacy reasons
- add an ID to ping request (and reply) in order to better match received replies, or encrypt with a management key
- "partial replication"/ "log-hole-punching" / - what would by a protocol (message formats) look for these? Do we need a second feed per author for announcing deleted log entries? One scenario is: allow downstream peers to pick up replication at any position oin a feed, without having to replicate all the past.
