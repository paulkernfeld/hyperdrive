# Hyperdrive Specification

Mathias Buus (@mafintosh). December 10th, 2015.

## DRAFT Version 0

A design and specification document for Hyperdrive. Hyperdrive is a protocol and network for distributing and replicating feeds of binary data. It is meant to serve as the main file and data distribution layer for [dat](https://dat-data.com) and is being developed by the same team.

The protocol itself draws heavy inspiration from existing file sharing systems such as BitTorrent and [ppspp](https://tools.ietf.org/html/rfc7574)

## Goals

The goals of hyperdrive is to distribute feeds of binary data to a swarm of peers in an as efficient as possible way.

It uses merkle trees to verify and deduplicate content so that if you share the same file twice it'll only have to downloaded one time. Merkle trees also allows for partial deduplication so if you make changes to a file only the changes and a small overhead of data will have to be replicated.

A core goal is to be as simple and pragmatic as possible. This allows for easier implementations of clients which is an often overlooked property when implementing distributed systems. First class browser support is also an important goal as p2p data sharing in browsers is becoming more viable every day as WebRTC matures.

It also tries to be modular and export responsibilities to external modules whenever possible. Peer discovery is a good example of this as it handled by 3rd party modules that wasn't written with hyperdrive in mind. A benefit of this is a much smaller core implementation that can focus on smaller and simpler problems.

Prioritized synchronization of parts of a feed is also at the heart of hyperdrive as this allows for fast streaming with low latency of data such as structured datasets (wikipedia, genomic datasets), linux containers, audio, videos, and much more. To allow for low latency streaming another goal is also to keep verifiable block sizes as small as possible - even with huge data feeds.

## Overview

As mentioned above hyperdrive distributes feeds of binary data (blocks). A *static feed* is a series of `n` related blocks of binary data that are identified by an incrementing number starting at 0.

```
// a static feed with n blocks of data

block #0, block #1, block #3, block #4, ..., block #n
```

These binary blocks of data can be of any reasonable size (strictly speaking they just have to fit in memory but implementations can choose to limit the max size to a more manageable number) and they don't all have to be the same fixed size. A static feed is identified by a hash of the roots of the the merkle trees it spans which means that a feed id (or link) is uniquely identified by it's content. See the "Content Integrity" section for a description on how these merkle trees are generated and the feed id is calculated.

A dynamic feed is like a static feed, but new blocks of data can be appended to a dynamic feed over time. Each dynamic feed is associated with a cryptographic key pair: only the owner of the private key can update a dynamic feed, but anyone with the public key can verify the authenticity of dynamic feed content. In fact, a dynamic feed is really only an updateable pointer to a point in a static feed.

## Content Integrity

When downloading a static feed, a peer must easily and efficiently verify that it has received the correct data. To do this, hyperdrive produces a single hash that summarizes the feed up to the `n`th block. By using Merkle trees, hyperdrive allows users to verify data without requiring that the user download the whole feed.

Given a feed with `n` blocks we can generate a series of merkle trees that guarantee the integrity of our feed when distributing it to untrusted peers.

For every block, hash it with a cryptographic hash function (draft 0 uses sha256), prefixed with the type identifier `0` to indicate that this is a block hash.

Using [node.js](https://nodejs.org) this would look like this

``` js
var blockHash = crypto.createHash('sha256')
  .update(new Buffer([0]))
  .update(blockBuffer)
  .digest()
```

Hyperdrive uses a [flat-tree](https://github.com/mafintosh/flat-tree) to represent a series of trees as a flat list.

```
// a flat tree representing six blocks (the even numbers)
// In this case, there are two Merkle trees, with roots at 3 and 9

                    3
                1       5       9
Tree index:   0   2   4   6   8  10

Block index:  0   1   2   3   4   5
```

This means that every block hash will be identified by `2 * blockIndex` in the flat tree structure as they are bottom nodes. Every odd numbered index is the parent of two children. In the above example `1` would be the parent of `0` and `2`, `5` would be parent of `4` and `6`, and `3` would be the parent of `1` and `5`. Note that `7` cannot be resolved as it would require `12` and `14` to be present as well.

To calculate the value of a parent node (odd numbered) we need to hash the values of the left and right child prefixed with the type identifier `1`.

Using [node.js](https://nodejs.org) this would look like this

``` js
var parentHash = crypto.createHash('sha256')
  .update(new Buffer([1]))
  .update(leftChild)
  .update(rightChild)
  .digest()
```

The type prefixing is to make sure a block hash cannot pretend to be a parent hash (a ["Second preimage attack"](https://en.wikipedia.org/wiki/Merkle_tree#Second_preimage_attack)).

Unless the feed contains a strict power of 2 number of blocks, we will end up with more than 1 root hashes when we are building this data structure.
To turn these hashes into the feed id simply do the following. Hash all the roots prefixed with the type identifier `2`, left to right, with the hash values and a uint64 big endian representation of the parent index.

Using [node.js](https://nodejs.org) this would look like this

``` js
var uint64be = require('uint64be') // available on npm
var feedId = crypto.createHash('sha256')
  .update(new Buffer([2]))

for (var i = 0; i < roots.length; i++) {
  feedId.update(roots[i].hash)
  feedId.update(uint64be.encode(roots[i].index))
}

feedId = feedId.digest()
```

Note that the amounts of roots is always `<= log2(number-of-blocks)` and that the total amount of hashes needed to construct the merkle trees is `2 * number-of-blocks`.

By trusting the feed id we can use that to verify the roots of the merkle trees given to us by an untrusted peer by trying to reproduce the same feed id using the above algorithm. When verifying we should also check that the root indexes correspond to neighboring merkle trees (see [flat-tree/fullRoots](https://github.com/mafintosh/flat-tree#roots--treefullrootsindex) for a way to get this list of indexes when verifying). For this reason the first response sent to a remote request should always contain the root hashes and indexes if the remote peer does't have any blocks yet. See the [response message](#messagesresponse) in the "Wire Protocol" section for more information.

The index of the left most block in the last merkle tree also tells us how many blocks that are in the feed which is useful if we want to show a progress bar or similar when downloading a feed (see [flat-tree/rightSpan](https://github.com/mafintosh/flat-tree/blob/master/index.js#L76) for more info on how to calculate this).

Assuming we have verified the root hashes, the remote only needs to send us the first "sibling" and all "uncle" hashes from the block index we are requesting to a valid root for us verify the block.

For example, using the above example feed with 6 blocks, assuming we have already verified the root at index 3, if we wanted to verify block #2 (tree index 4) we would need the following hashes

```
// we need 6, the sibling and 1, the uncle

                   3
               (1)          5
Tree index:  0     2     4    (6)

Block #:     0     1     2     3
```

We also need the content for block #2 of course. Given these values we can verify the block by doing the following

``` js
var block4 = blockHash(block #2)
var parent5 = parentHash(block4, block6)
var parent3 = parentHash(parent1, parent5)
```

If parent3 is equal to the root at tree index 3 (that we already trust) we now know block #2 was correct and we can safely store it and share it with other peers.

If we had received the hash for tree index 5 before and verified it then we would not need to re-verify the root and we would be able to insert block #2 after validating parent5.

It should be noted that this allows us to have random access to any block in the feed with content verification in a single round trip. In the worst case, verifying the a single block requires only `O(log(N))` time with respect to the number of blocks in the feed.

As an optimization we can use the remote's "have" messages (see the ["Wire Protocol" section](#messageshave)) to figure out which hashes it already has. For example if the remote has block #3 then we do not need to send any hashes since it will already have verified hash 6 and 5.

Hyperdrive's content integrity approach is similar to parts of [ppspp](https://tools.ietf.org/html/rfc7574) and additional information can found in that specification.

## Deduplication

Assuming you have two different static feeds that are sharing similar data, we want to minimize fetching of duplicate data. The merkle tree structure helps us achieve this by content addressing the feeds.

For example if we were to produce the exact same feed on two different computers using the technique [described in the **Content Integrity** section](#content-integrity), the feeds would end up with the same root tree hashes and the same feed id. The two feeds would therefore be the same feed.

A more interesting case is sharing two similar feeds that are not 100% the same but share partial sections. This could be two different versions of the same file, an old one, and an updated one with a few changes. In this case, feed ids would be different. However, since every block is being delivered with the tree hashes necessary to verify it against the root of the merkle tree, we can maintain a simple index that points from the parent hashes in the merkle trees to the data blocks they represent and check against this index to see if we have already fetched other parts a feed before shared by another feed.

For example assume we have two feeds, A and B that share all blocks except the last one. If we previously fetched A and now are trying to fetch B, we will notice that by fetching any block in B we would already have most of the hashes contained in the proof for the block

```
// A and B both share 0, 2, and 4 meaning that 1 will be the same.
// If block #3 is requested (tree index 6) the sibling hash 4 and uncle hash 1
// will be returned. We would then notice we already have the hash for 4 and 1
// and no other blocks would need to be downloaded

      3
  1        5
(0   2   4)   6
```

It should be noted that this means that we might still download redundant data since we only receive the tree hashes necessary to deduplicate other parts of the feed by downloading blocks. However, assuming that continuous ranges of the feed are shared with another peer, the redundant block downloads should be limited as we discovery enough tree hashes on every request to deduplicate at least half of the tree between a block and a verified parent hash.


```
// If block #1 is requested (tree index 2) and we have already
// verified 7, we will receive 5 which tells us if half the tree
// spanned by 3 is part of a different feed

                (7)
        3
  1         5
0   (2)   4   6   ....
```

Another consequence of this is the no round trips are wasted. Using only a single round trip we will always receive data and additional information that helps us deduplicate other parts of the same feed as an optimization.

## Dynamic Feeds

The id of a static feed refers to a specific point in that static feed; updating the content of a static feed will produce a new id. By contrast, a dynamic feed always keeps the same id, but new content can be added to it over time. Each dynamic feed corresponds to a cryptographic key pair. Only the owner of the private key can update a dynamic feed, but anyone with the public key can verify an update to the feed.

The id of a dynamic feed is a public key. When broadcasting an update of a dynamic feed, the owner of the node sends a message containing a link to a static feed and an incrementing sequence number. The message is signed with the private key for the feed.

When exchanging a dynamic feed, two nodes share their data about the feed. Whichever message is valid and has the highest sequence number will be accepted as the new value for that feed.

A dynamic feed only allows content to be appended, not removed or edited. That is, if the owner of the feed broadcasts an update containing `block 0, block 1, ..., block n`, then all future updates to that feed must refer to a static feed that begins with blocks `0...n`. This makes a dynamic feed an append-only log store.

## Wire Protocol

Hyperdrive peers can communicate over any reliable binary duplex stream using a message oriented wire protocol.

All messages sent are prefixed with a [varint](https://developers.google.com/protocol-buffers/docs/encoding?hl=en#varints) containing the length of the payload when sent over the wire

```
<varint-length-of-payload><payload>
```

In general payload messages are small and an implementation should make sure to only buffer messages that can easily fit in memory. The javascript implementation currently rejects all messages >5mb. If an empty payload is received (a payload with length `0`) it should be ignored. This means empty payloads can used as keep-alive messages to ensure connections are kept open when no other messages are being sent.

All messages, except for the first one which is always a handshake, are also prefixed with a varint describing the message type

```
<varint-length-of-payload>(<varint-type><message>)
```

If you receive a message with an unknown type you should ignore it for easier compatibility with future updates to this spec.

The messages are encoded into binary values using [protobuf](https://developers.google.com/protocol-buffers).
A single stream using the wire protocol can be used to share multiple swarms of data using a simple multiplexing scheme that is described below.

#### messages.Handshake

The first message sent should always be a handshake message. It is also the only message not prefixed with a type indicator.

The protobuf schema for the message is

``` proto
message Handshake {
  required string protocol = 1;
  optional uint64 version = 2;
  required bytes id = 3;
}
```

`protocol` string should be set to `"hyperdrive"`. This is to allow for early failure incase the other peer is sending garbled data.

`version` should be set to the version of the hyperdrive protocol you are speaking.

`id` should be a short binary id of the peer. The javascript implementation uses a 32 byte random binary value for this.

#### messages.Join

A join message has type `0` and the following schema

``` proto
message Join {
  required uint64 channel = 1;
  required bytes link = 2;
}
```

Should be sent when you are interested in joining a specific swarm of data specified by the `link`.

`channel` should be set to the lowest locally available channel number (starting at `0`) and all subsequent messages sent referring to the same `link` should contain the same channel number. When receiving a Join message, if you wish to join the swarm, you should reply back with a new Join message if you haven't sent one already containing the same link and your lowest locally available channel number.

When having both sent and received a remote Join message for a specific swarm `link` you will have both a local and remote channel number that can be used to separate swarm messages from each other and allows for reuse of the same connection stream for multiple swarms. Note that since you only pick your local channel number and a channel has both a local and remote channel id, there are no risk of channel id clashes.

#### messages.Leave

A leave message has type `1` and the following schema

``` proto
message Leave {
  required uint64 channel = 1;
}
```

Should be sent when you are no longer interested in a data swarm and wish to leave a channel.

#### messages.Have

A have message has type `2` and the following schema

``` proto
message Have {
  required uint64 channel = 1;
  repeated uint64 blocks = 2;
  optional bytes bitfield = 3;
}
```

Should be sent to indicate that you have new blocks ready to be requested.

`blocks` can optionally be set to an array of new block indexes you have.

`bitfield` can optionally be set to a binary bitfield containing a complete overview of the blocks you have.

#### messages.Pause

A pause message has type `3` and the following schema

``` proto
message Pause {
  required uint64 channel = 1;
}
```

Should be sent if you want to pause a channel.

If you are pausing a channel you are indicating to the remote that you will not accept any requests.
All channels should start out paused so you should only send this if you've previously sent an `unpause` message. Note that pausing is a one-way relationship: when you're pausing a remote, the remote might not be pausing you.

#### messages.Unpause

An unpause message has type `4` and the following schema

``` proto
message Unpause {
  required uint64 channel = 1;
}
```

Should be sent if you want to unpause a channel.

As mentioned above all channels start out paused so you'll need to send this if you want to allow the remote to start requesting blocks.

#### messages.Request

A request message has type `5` and the following schema

``` proto
message Request {
  required uint64 channel = 1;
  required uint64 block = 2;
}
```

Should be sent if you want to request a new block.

If you receive a request while you are pausing a remote you should ignore it.

`block` should be set to the index of the block you want to request.

#### messages.Response

A response message has type `6` and the following schema

``` proto
message Response {
  message Proof {
    required uint64 index = 1;
    required bytes hash = 2;
  }

  required uint64 channel = 1;
  required uint64 block = 2;
  required bytes data = 3;
  repeated Proof proof = 4;
}
```

Should be sent to respond to a request.

`block` should be set to the index of the block you are responding with.

`data` should be set to the binary block.

`proof` should be set to the needed integrity proof needed to verify that this block is correct. The proof should contain the first sibling and uncles hashes and indexes needed to verify this block. If this is the first response the remote is gonna receive (they has sent no have message, or the bitfield was empty) you should include the merkle tree roots and the end of the proof. See the "[Content Integrity](#content-integrity)" section for more information.

After receiving a verified response message you should send a have messages to other peers you are connected, except for the peer sending you the response, to indicate you now have this piece. If you are the peer sending a response you should record that the remote peer now have this block as well.

#### messages.Cancel

An cancel message has type `7` and the following schema

``` proto
message Cancel {
  required uint64 channel = 1;
  required uint64 block = 2;
}
```

Should be sent to cancel a previously sent request.

`block` should be set to the same block index as the request you want to cancel.

#### dynamic

A dynamic message has type `8` and the following schema

``` proto
message dynamic {
  required uint64 channel = 1;
  required uint64 pubkey = 2;
  required sequence = 3;
  optional hash = 4;
  optional signature = 5;
}
```

This message serves as a request for the update of a dynamic feed, as well as a way for sending updates to a dynamic feed. The pubkey identifies the dynamic feed. The sequence number is the local sequence number. The hash is the id of a static feed. The signature is a concatenation of the sequence number and the hash, signed by the public key.

## Downloading / uploading flow

Lets assume we receive a static feed id through an external channel (such as text message or IM). To start downloading and sharing this feed we first need to find some peers sharing it. This could be accomplished in many ways. The way the javascript implementation encourages this is using another [dat](https://dat-data.com) developed module for peer discovery called [discovery-channel](https://github.com/maxogden/discovery-channel) that uses multicast-dns and the bittorrent dht to announce and lookup peers from a key.

Once we are connected to a swarm of peers we need to decide which peers we want to allow requesting data from us.
This is where the pausing messages become useful. All channels start out paused and it is up to the sending peer to unpause a remote peer when it is ready to accept requests. When to unpause a peer can depend on a lot of different parameters and multiple strategies exists for this. The Hyperdrive javascript implementation currently uses a tit-for-tat and optimistic unpausing based strategy similar to the one used in BitTorrent.

Once a remote peer unpauses us we can start requesting blocks of data that we are missing.

## Block prioritization

When deciding which block to download from a remote peer, a couple of things should be taken into account.

To allow for features like live booting linux containers, real-time video and audio streaming, etc... Hyperdrive allows for higher prioritization of specific ranges of blocks to download first.

If no blocks ranges are prioritized, a strategy such as ["Rarest first" or "Random first piece"](http://www.bittorrent.org/bittorrentecon.pdf) should be used similar to the strategy used in BitTorrent. This allows blocks to be spread out evenly across peers without having single peers that are the only uploaders of a specific piece.

If a range is prioritized, then blocks should be chosen from this range first. Ranges can be prioritized with different weights, CRITICAL, HIGH, NORMAL. For example if we are streaming video we might want to prioritize the first megabyte of the video as CRITICAL as the playback latency depends on this. We might also want to prioritize the next 10 mb with HIGH as we want to download that as a buffer. For ranges marked as CRITICAL, it is acceptable to make requests for the same block from this range to multiple peers.  For example, if a critical block is requested from peer A, and a newer peer B comes along that has much higher bandwidth available than peer A, but no free critical blocks to fetch, then it's ok to request the same critical block from peer B. This is referred to as "hotswapping".

```
// pseudo code for hotswapping a slow peer to reduce latency

if (critical-range) {
  if (all-blocks-being-requested && freePeer.downloadSpeed / slowestPeer.downloadSpeed > threshold) {
    freePeer.request(block-that-is-also-requested-by-slowest-peer)
  }
}
```

## Dynamic feed update flow

Dynamic feeds may be exchanged by push or pull mechanisms. A feed may be requested, or the originator of a feed may proactively broadcast updates to the dynamic feed when it makes them.

In the case of requesting a dynamic feed, if a node doesn't have any data from the dynamic feed, it sends a `dynamic` message where the sequence number is `0`, and the hash and signature are omitted. If it has data, it sends a `dynamic` message containing its own copy of the state of the feed.

When a peer receives a `dynamic` message, first it compares the sequence number received with its own local sequence number. If they are the same, it can ignore the message.

If the received sequence number is smaller than the local one, the local node has a newer copy of the dynamic feed state than the remote node. The local node may respond with another `dynamic` message to tell the remote about the new state of the feed.

If the received sequence number is larger than the local one, the remote has a newer copy of the dynamic feed. First, the node validates the message by checking that the signature is valid. Then, it may check that the feed referred to by the new message is a strict superset of our local copy (this would only be useful if we're worried about the owner sabotaging their own feed). If the received sequence number passes this validation, we accept it as our own local state of the feed.

## File sharing

As mentioned earlier, Hyperdrive distributes feeds of binary data. This allows it to have applications outside of file sharing although file sharing is a key feature that we want to support. This sections describes how a file sharing system could easily be implemented on top of Hyperdrive whilst still leaving the core protocol data agnostic.

Distributing files requires two things. A way to distribute the actual file content and a way to distribute the file metadata (filename, file permissions etc).

#### Distributing file content

To distribute file content we need turn a file into blocks of data so we can store every file in an individual feed. A simple way of doing this is dividing a file into fixed size blocks. This is very easy to implement and is also the technique used by BitTorrent. Unfortunately by dividing a file into fixed size blocks results in all blocks being changed if we to insert a single new byte into the beginning of a file. The consequence of this would be that we would have to re-download all blocks if we tried to download this new file even though we previously had downloaded the old (see the "Deduplication" section).

```
// Assume file contains "abcdef" and we divide into blocks of 2

ab,cd,ef

// If we insert g at the beginning of the file the blocks will be

ga,bc,de,f

// Which means that no blocks are shared between the files
// even though they are similar
```

To fix this we use a chunking strategy called "Rabin fingerprinting" which is implemented by the [rabin](https://github.com/maxogden/rabin) npm module (also maintained by the [dat](https://dat-data.com) team). Rabin fingerprinting is a content defined chunking strategy that splits a file into varied size blocks based on the file content. The rabin npm module is tuned to produce blocks around 20 kilobytes on average per default. Content defined chunking means that only the neighboring blocks are likely to change when changing a file. So in the above case where we insert a byte to front of a file only the first block is likely to change. This is a very powerful property as it allows us to easily deduplicate a file between multiple versions. However a consequence of the blocks not being the same size is that random access based on a byte offset becomes difficult. To fix this we record the length of each chunk as an unsigned 16 bit big endian integer into an index that we append to the end of the feed after writing the content blocks. A 16 bit integer takes up two bytes and we store the block lengths in buffers of up to 16 kilobytes. 16 kilobytes means that we are able to fit the lengths of 8192 blocks of content into a single "length index" block corresponding to roughly 210 megabytes of content if every content block is around 20 kilobytes. For files that are gigabytes or terrabytes in size multiple "length index" blocks would be needed.

```
// Divide the file content into blocks using a rabin fingerprinter
// and suffix the feed with an index containing the length of each content block

content #0, content #1, ..., content #n, length index #1, length index #2
```

If you only know the total number of blocks in a feed the amount of length index blocks can easily be calculated

```
length-indexes = content-blocks / 8192
feed-blocks = content-blocks + length-indexes

=>
feed-blocks = content-blocks + content-blocks / 8192

=>
content-blocks = 8192 / 8193 * feed-blocks

~> (a block cannot be a fraction)
content-blocks = floor(8192 / 8193 * feed-blocks)
```

By downloading this small index we can use it to find which block contains a specific byte offset. As an optimization the total length of all blocks stored in a "length index" block can be stored in the files data metadata (next section). This is referred to as the index digest. By doing this we would only need to download a single index block to find the block we are looking for, for any given byte offset. Similarly the length indexes can also be used to determine the byte offset to which a block should be written inside a file when downloading it.

Since both the length indexes and the index digest are essentially delta compressed lists of byte offsets they can also be inflated after download to allow for algorithms such as binary search to find the block corresponding to a byte offset.

Since every file will be stored in individual feeds it also means that if you share the same file twice it will result in the same feed id meaning stronger swarms and more deduplication.

#### Distributing file metadata

A file metadata feed can be thought of as a feed of all the headers of every entry in a tarball. They contain all the information necessary to describe a file but without the actual file content. The metadata is encoded using this protobuf schema:

``` proto
message Metadata {
  message Link {
    required bytes id = 1;
    required uint64 blocks = 2;
    repeated uint64 index = 3;
  }

  enum TYPE {
    FILE = 0;
    DIRECTORY = 1;
  }

  optional TYPE type = 1 [default = FILE];
  required string name = 2;
  optional uint32 mode = 3;
  optional uint64 size = 4;
  optional uint32 uid = 5;
  optional uint32 gid = 6;
  optional uint64 mtime = 7;
  optional uint64 ctime = 8;
  optional Link link = 9;
}
```

If a file is not empty the `link` should be set and `link.id` should be the feed id of the content feed. `link.blocks` and `link.index` should be set to the amount of content blocks and the index digest of the content feed to reduce the amount of data needed to start downloading a block at a specific byte offset (see the above file content discussion).

`name` should be set to a full path of the entry relative to the folder you are sharing. A possible optimization would be to sort the file metadata by the name of entry as this allows for distributed binary search to find a single entry you are looking for without downloading all available metadata first.

Since the metadata feed contains all the information needed to create a file system stat object it is possible to mount a file archieve shared with Hyperdrive as a fully features file system using fuse (or a similar user space file system) by only syncing the metadata and then lazily fetching the file data on demand or in the background.

## Key differences to BitTorrent

Although file sharing using Hyperdrive on the surface could seem similar to tools such as BitTorrent there are a few key differences.

#### Not all metadata needs to synced up front

BitTorrent requires you to fetch all metadata relating to magnet link from a single peer
before allowing you to fetch any file content from any other peer. This also makes it difficult to share archives of 1000s of files using BitTorrent.

#### Flexible and consistently small block sizes

BitTorrent requires a fixed block size that usually grows with the size of the content you are sharing.
This is related the above mentioned fact that all metadata needs to be exchanged from a single peer before any content can be exchanged. By increasing the block size you decrease the number of hashes you need to exchange up front. Flexible block sizes also allow for more non file related use cases, such as the file metadata feed described above.

#### Deduplication

BitTorrent inlines all files into a single feed. This combined with the fixed block sizes makes deduplication hard and you often end up downloading the same files multiple times if an update to a torrent is published.

#### Multiplexed swarms

Unlike BitTorrent, wires can be reused to share multiple swarms which results in a smaller connection overhead when downloading or uploading multiple feeds shared between two or more peers.
