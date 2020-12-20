# Creating and Replicating Hypercores

This walkthrough will go through the basics of using Hypercore as a standalone module. 

Hypercore gives you many knobs to tweak, so if you'd like something with more "batteries included", head on over to the [Hyperspace Walkthrough](/hyperspace).

### Introduction

Hypercore is the bread-and-butter of the Hypercore Protocol. It is a lightweight, secure append-only log with several powerful properties:
1. __Secure__: Hypercore builds a Merkle tree out of its blocks, so readers can always verify that the log hasn't been tampered with. 
2. __Easy Replication__: The `replicate` method returns a Duplex stream that can be piped over an arbitrary transport stream.
3. __On-Demand Downloading__: With "sparse mode", readers will download blocks from peers when they are first requested. 
4. __Caching__: Once peer has downloaded a block, it will be cached locally. The download will only happen once.
5. __Bandwidth Sharing__: As with BitTorrent, readers download blocks from many connected peers in parallel.
6. __Live Updating__: Peers can be notified whenever a Hypercore has grown.

A Hypercore can only have a __single writer, on a single machine__ -- the creator of the Hypercore is the only person who can modify to it, because they're the only one with the private key. That said, the writer can replicate to __many readers__, in a manner similar to BitTorrent.

Unlike with BitTorrent, a Hypercore can be modified after its initial creation, and peers can receive live notifications whenever the writer adds new blocks.

In this walkthrough, we'll create two Hypercores: one writable and one that's a read-only clone of the first, which will simulate a remote peer. We'll then show you how to initiate replication between the two cores, and have the reader download and display live updates.

### Trying it out

This example only has two dependencies: `hypercore` and `hypercore-promisifier`, which gives you a Promises interface for Hypercore.

Each step builds on the previous, and you can run them all from the command line with `node step-N.js`.

### [Step 1](/hypercore/step-1.js): Create a Writable Hypercore 

The Hypercore constructor has two main parameters, `storage` and `key`, as well as a handful of options, several of which we'll work with in the next steps.

Let's create a new core with UTF-8 encoded blocks, stored in the `./main` directory:
```js
const core = toPromises(hypercore('./main', {
  valueEncoding: 'utf-8' // The blocks will be UTF-8 strings.
}))
```

Hypercore currently only has a callback API, so we use the `hypercore-promisifier` module to wrap it in a Promises API. This keeps things more concise.

We can append two new blocks to our core using the `append` method, which accepts either a single block or an Array of blocks:
```js
await core.append(['hello', 'world'])
```

Now that our main core has two blocks in it, let's create a clone and start replicating.

#### The Storage Parameter

Cores can be stored in any instance of [`random-access-storage`](https://github.com/random-access-storage). Inside the random-access-storage GitHub repo, you'll storage modules for disk, memory-only, S3, IndexedDB, and more.

If you pass in a String, Hypercore will use on-disk storage by default, and will treat the argument as the storage directory.

#### Hypercore Keys

All Hypercores are identified by two properties: A __public key__ and a __discovery key__, the latter of which is derived from the public key. Importantly, the public key gives peer __read capability__ -- if you have the key, you can exchange blocks with other peers.

Without the key, a peer is unable to validate blocks, and the Hypercore transport protocol will block replication.

Since the public key is also a read capability, it can't be used to discover other readers (by advertising it on a DHT, for example), as that would lead to capability leaks. The discovery key, being derived from the public key but lacking read capability, can be shared openly for peer discovery.

### [Step 2](/hypercore/step-2.js): Create a Read-Only Clone

Let's simulate a remote peer by creating a read-only clone of the Hypercore from the previous step. 
```js
const clone = toPromises(hypercore('./clone', core.key, {
  valueEncoding: 'utf-8',
  sparse: true, // When replicating, don't eagerly download all blocks.
  eagerUpdate: true // But eagerly fetch length updates from peers.
}))
```

Since this clone is going to be downloading data from the original core, we've specified a few additional options, `sparse` and `eagerUpdate`, described below.

The `replicate` method can be used to create a Duplex stream that implements the [Hypercore Protocol](https://github.com/hypercore-protocol/hypercore-protocol). We can pipe together two replication streams, one from the original core and one from the clone, in order to start exchanging blocks:

```js
const firstStream = core.replicate(true, { live: true })
const cloneStream = clone.replicate(false, { live: true })

// Pipe the stream together to begin replicating.
firstStream.pipe(cloneStream).pipe(firstStream)
```

These replication streams are end-to-end encrypted using the [NOISE protocol](https://noiseprotocol.org/). The first argument (`true` or `false`) indicates which peer initiated the replication process.

In this example, we're directly piping the two replication streams together, but in real-world scenarios you'll typically pipe into network streams (such as TCP or UTP peer connections), which we'll explore in the Hyperswarm walkthrough.

Both replication streams are "live", meaning they'll continue replicating indefinitely.

Now that the clone has a connected peer, we can start requesting blocks. These blocks will be lazily downloaded, because the clone was constructed in sparse mode:
```js
console.log('First clone block:', await clone.get(0)) // 'hello'
console.log('Second clone block:', await clone.get(1)) // 'world'
```

#### Sparse Downloading

Readers do not need to download complete Hypercores in order to read individual blocks. A Hypercore might be massive (containing a huge database, say), but often readers will only be interested in a small subset of its blocks. This is especially true when working with data structures built on top of Hypercore.

The `sparse` flag turns on "sparse mode", which instructs a Hypercore to disable block downloading unless a block is explicitly requested (through `get` or `createReadStream`). If `sparse` is false, then Hypercore will automatically download as many blocks as possibly from every peer it connects to.

#### Updating




### [Step 3](/hypercore/step-3.js): See Live Updates

```js
```

### Next Steps

In the [Hyperswarm Walkthrough](/hyperswarm), we'll see how to replicate Hypercores to remote peers over a P2P network.
