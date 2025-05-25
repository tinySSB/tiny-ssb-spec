# tinySSB test vectors: GoSET

This file and directory contains some test vectors that emerge when
two peers synchronize their sets of feed IDs (FID). There is are two
message types in the synchronization protocol for which we provide
test vectors: ```novelty``` and ```claim```.

## Notation

We use [s-expressions](https://www.ietf.org/archive/id/draft-rivest-sexp-13.html) for representing constants, data records, sequences and packets.

We denote packet types by the name of its encoding function,
followed by the resulting byte sequence (test vector) e.g.,

```
(NOVELTY_ENCODE #...7_DMX_bytes...# "N" #...32_novelty_bytes...#")
==
#....#
```

## The Two GoSET Packet Types

### Novelty

A ```novelty``` message is used to announce a new FID. It has three fields:

```
dmx   7B DMX value for the GoSET protocol (constant value)
t     1B packet type (fixed to 'N' for a novelty)
f    32B FID
```

which results in a packet with 40 bytes. It is assumed that the link
media provides the packet's length and verifies the integrity of the
```novelty``` packet (using some CRC value) before processing it.

Using a ```novelty``` message is necessary when a peer only knows
about one FID. ```Novelty``` packets can also be used when a node can
assume that its peers do _not_ know yet about a FID. This is
beneficial because it avoids the set reconciliation process via
```claim``` messages.

### Claim

A ```claim``` message has five fields:
```
dmx   7B DMX value for the GoSET protocol (constant value)
t     1B packet type (fixed to 'C' for a claim)
f1   32B smallest known FID
f2   32B highest known FID
fx   32B XOR of all known FIDs
cnt   1B number of known FIDs (0..255)
```

which results in a packet with 105 bytes. It is assumed that the link
media provides the packet's length and verifies the integrity of the
```claim packet``` (using some CRC value) before processing it.

The ```dmx``` field for all GoSET packets is a constant and is
computed as the first 7 bytes of the SHA256 value of the ASCII string

```
tinySSB-0.1 GOset 1
```

with length 19. The value is:

```
GoSET_DMX
===
#??????????????#
```


## Four Scenario:

Peer A knows about five feeds:
```
#1000001f4fb57f5d84506379a31c0ab5f24c435c63f2df9b8c373817975eed1f#
#020000a391c773f2e9e22b03df4731b64616024bbd2893f436db8fc1c922d5e8#
#003000142e693f47f7b09940552bc05a58282491e3f09fa19749fabf500f4990#
#000400eb4dadc66e1d871278fe92aa1f4e52e7e64bda6fda52b8623ab887785a#
#0000502302156b0061f1ca08d296d2367cfb40c88c005b73413bc394c4022aca#
```

Initially, peer B only knows about its own feed:
```
#0000069ee3e69964fc50391b1c1d80e918a419f3a592a8eb80cfc549dc61be99#
```

### case a) Peer A sends a ```claim``` about its five feeds (#1 to #5)

...

### case b) Peer B sends a ```claim``` about its single feed (#6)

...


### case c) Peer A reacts after including feed #6 into its set

...


### case d) Peer B reacts after including feeds #1 and #5

...