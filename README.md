![N|Solid](https://hnrddp064o-flywheel.netdna-ssl.com/wp-content/uploads/2018/11/Tendermint.png)

[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

Tendermint Core is a blockchain application platform; it provides the equivalent of a web-server, database, and supporting libraries for blockchain applications written in any programming language. Like a web-server serving web applications, Tendermint serves blockchain applications. This document provides more extensive analysis of its codebase. 

Sole purpose of this document is to understand how tendermint works from technical perspective and present:
  - Workflow and execution analysis
  - Modules analysis
  - Efficiency assesment
  - Configuration possibilities

# Tendermint modules:
- ABCI - Tendermint acts as an ABCI client with respect to the application and maintains 3 connections: mempool, consensus and query. 
- Blockchain - Provides storage, pool (a group of peers), and reactor for both storing and exchanging blocks between peers. 
- Consensus - The heart of Tendermint core, which is the implementation of the consensus algorithm. Includes two "submodules": wal (write-ahead logging) for ensuring data integrity and replay to replay blocks and messages on recovery from a crash. 
- Events - Simple event notification system. The list of events can be found here. You can subscribe to them by calling subscribe RPC method. 
- Mempool - Mempool module handles all incoming transactions, whenever they are coming from peers or the application. 
- P2P - Provides an abstraction around peer-to-peer communication. 
- RPC - Tendermint's RPC implementation 
- RPC-Server - Tendermmint's RPC-Server implementation 
- State - Represents the latest state and execution submodule, which executes blocks against the application. 
- Types - A collection of the publicly exposed types and methods to work with them. 


![N|Solid](https://tendermint.com/docs/assets/img/abci.3542de28.png)

# Mempool 
 
Mempool is a pool of transactions received from the ABCI.  The mempool maintains a list of potentially valid transactions, both to broadcast to other nodes, as well as to provide to the consensus reactor when it is selected as the block proposer.

Main roles of the membool module: 
- broadcasting transactions across the peers 
- validating received transactions by calling CheckTx onto ABCI
- provide transactions to a new proposed block

###### There are two sides to the mempool state:
###### 1. External functionality is exposed via network interfaces to potentially untrusted actors.
- CheckTx - triggered via RPC or P2P - ask node to checkTx
- Broadcast - gossip messages after a successful check
###### 2. Internal functionality is exposed via method calls to other code compiled into the tendermint binary.
- Reap - get tx to propose in next block
- Update - remove tx that were included in last block
- ABCI.CheckTx - call ABCI app to validate the tx

Mempool instance is an ordered in-memory pool for transactions before they are proposed in a consensus 
Round. Transaction validity is checked using the CheckTx abci message before the transaction is added to the pool. The Mempool uses a concurrent list structure for storing transactions that can be efficiently accessed by multiple concurrent readers. 
 
Mempool Reactor instance receives transaction from other peers and calls CheckTx on the abci application. 
When new peer is added to reactor, broadcast go routine is run which constantly waits for new transaction appearing in the mempool and broadcasts it to registered peers. 

There are following configurations for the mempool:
```go
mempool.recheck=true (default: true)
```
Recheck transactions in the mempool. Recheck determines if the mempool rechecks all pending transactions after a block was committed. Once a block is committed, the mempool removes all valid transactions that were successfully included in the block. If recheck is true, then it will rerun CheckTx on all remaining transactions with the new block state.
```go
 mempool.broadcast=true (default: true)
```
Broadcast determines whether this node gossips any valid transactions that arrive in mempool. Default is to gossip anything that passes checktx. If this is disabled, transactions are not gossiped, but instead stored locally and added to the next block this node is the proposer.
```go
mempool.wal_dir=$TM_HOME/data/mempool.wal
```
Waldir  defines the directory where mempool writes the write-ahead logs. These files can be used to reload unbroadcasted transactions if the node crashes.


# Blockchain 
 
Blockchain is a module responsible for storing correct order of blocks of transactions linked together by a cryptographic hash.  
 
Main roles of the Blockchain module: 
- store transaction blocks  
- sync blockchain with other nodes: communicate with other nodes on blockchain height and share missing blocks 
 
Blockchain pool constantly crawls registered peers for their blockchain height in order to sync its own blockchain. 
Blockchain reactor handles long-term catchup syncing. It can ask and respond for the height of the blockchain, and request and respond new available blocks. Blocks requested are then added to Blockchain pool. 
 
Blockchain reactor when started runs poolRoutine. One of the responsibilities of the PoolRoutine is to rebuild correct blockchain from the pool. While the blockchain pool is in sync (requesting missing blocks), PoolRoutine picks two consecutive blocks and validates their cryptographic commits. If the blocks requested from the pool are valid, iterative pulling of two consecutive blocks should always rebuild entire blockchain correctly. 
 
Blockchain package has also persistent storage mechanism to store valid blockchain replayed from the pool. 

# Blocks Structure 
 
The tendermint consensus engine records all agreements by a supermajority of nodes into a blockchain, which is replicated among all nodes. This blockchain is accessible via various rpc endpoints, mainly:
To get the full block:
```
 /block?height= 
```
To get a list of headers:
```
 /blockchain?minHeight=_&maxHeight=_ 
``` 
 But what exactly is stored in these blocks? 
 
A Block contains: 
- Header - contains merkle hashes for various chain states 
- Data - all transactions which are to be processed 
- LastCommit - supermajority 2/3+ signatures for the last block (The signatures returned along with block H are those validating block H-1) 

Header contains LastBlockId (chain), hashes of data, app state and validator set. Header is the only thing that is signed by the validators, data is validated separately against one of the merkle hashes in the header.  
 
The Commit contains a set of Votes that were made by the validator set to reach consensus on this block. This is the key to the security in any PoS system, and actually no data that cannot be traced back to a block header with a valid set of Votes can be trusted. Thus, getting the Commit data and verifying the votes is extremely important. 
Each vote only stores the validators Address, as well as the Signature. Assuming we have a local copy of the trusted validator set, we can look up the Public Key of the validator given its Address, then verify that the Signature matches the SignBytes and Public Key. Then we sum up the total voting power of all validators, whose votes fulfilled all these stringent requirements. If the total number of voting power for a single block is greater than 2/3 of all voting power, then we can finally trust the block header, the AppHash, and the proof we got from the ABCI application. 
 
Transmiting block data might be heavy, that is why block data is divided into parts in order to orchestrate the p2p propogation. The PartSetHeader structure contains the total number of pieces in a PartSet, and the Merkle root hash of those pieces.  PartSet is used to split a byteslice of data into parts (pieces) for transmission. By splitting data into smaller parts and computing a Merkle root hash on the list, you can verify that a part is legitimately part of the complete data, and the part can be forwarded to other peers before all the parts are known. In short, it's a fast way to securely propagate a large chunk of data (like a block) over a gossip network. 
 
More details on the block structure can be found [here ](https://github.com/tendermint/tendermint/blob/master/docs/spec/blockchain/blockchain.md)
 


# P2P 
 
P2P module is responsible for establishing and managing connections with other peers and handling incoming and outgoing data traffic. 
 
Main roles of the P2P package: 
- managing address book 
- nodes discovery 
- establishing connection with known peers. Known peers are dialled and MConn is established between the peers. 
- managing connections with peers. Reconnecting, Stopping and Removing peers. 
- handling incoming and outgoing data exchange between the nodes. 


### Peer tracking

Peers are tracked via their ID (their PubKey.Address()). Address Book is used for tracking peers so we can gossip about them to others and select peers to dial. Address Book is a file based mechanism of keeping track ip addresses of known nodes in the system. The address book is arranged in sets of buckets, and distinguishes between vetted (old) and unvetted (new) peers. It keeps different sets of buckets for vetted and unvetted peers. Buckets provide randomization over peer selection. If we're trying to add a new peer but there's no space in its bucket, we'll remove the worst peer from that bucket to make room. 

When a peer is first added, it is unvetted. For Tendermint, a Peer becomes vetted once it has contributed sufficiently at the consensus layer; ie. once it has sent us valid and not-yet-known votes and/or block parts for NumBlocksForVetted blocks. 
When we need more peers, we pick them randomly from the addrbook with some configurable bias for unvetted peers. The bias should be lower when we have fewer peers and can increase as we obtain more, ensuring that our first peers are more trustworthy, but always giving us the chance to discover new good peers.
 
### Peer Dialing
Certain peers are special in that they are specified by the user as persistent, which means we auto-redial them if the connection fails, or if we fail to dial them. Some peers can be marked as private, which means we will not put them in the address book or gossip them to others. All peers except private peers and peers coming from them are tracked using the address book.

On startup, we will also immediately dial the given list of persistent_peers, and will attempt to maintain persistent connections with them. If the connections die, or we fail to dial, we will redial every 5s for a few minutes, then switch to an exponential backoff schedule, and after about a day of trying, stop dialing the peer.
So long as we have less than ```MinNumOutboundPeers```, we periodically request additional peers from each of our own. If sufficient time goes by and we still can't find enough peers, we try the seeds again.

We track the last time we dialed a peer and the number of unsuccessful attempts we've made. If too many attempts are made, we mark the peer as bad. If we fail to connect to the peer after 16 tries, we remove from address book completely.
Peers listen on a configurable ListenAddr that they self-report in their NodeInfo during handshakes with other peers. Peers accept up to (MaxNumPeers - MinNumOutboundPeers) incoming peers.


### P2P Config
```go
--p2p.seed_mode
```

The node operates in seed mode. In seed mode, a node continuously crawls the network for peers, and upon incoming connection shares some peers and disconnects.

```go
--p2p.seeds “1.2.3.4:26656,2.3.4.5:4444”
```

Dials these seeds when we need more peers. They should return a list of peers and then disconnect. If we already have enough peers in the address book, we may never need to dial them.
```go
--p2p.persistent_peers “1.2.3.4:26656,2.3.4.5:26656”
```

Dial these peers and auto-redial them if the connection fails. These are intended to be trusted persistent peers that can help anchor us in the p2p network. The auto-redial uses exponential backoff and will give up after a day of trying to connect.

Note: If seeds and persistent_peers intersect, the user will be warned that seeds may auto-close connections and that the node may not be able to keep the connection persistent.

```go
--p2p.private_persistent_peers “1.2.3.4:26656,2.3.4.5:26656”
```

These are persistent peers that we do not add to the address book or gossip to other peers. They stay private to us.

Checks if ip addresses are valid before adding to address book 
```go
--p2p.addr_book_strict
--p2p.flush_throttle_timeout 
--p2p.max_packet_msg_payload_size 
--p2p.send_rate 
--p2p.recv_rate 
```
 
### tendermint recommended 
```go
  send_rate=20000000 # 2MB/s 
  recv_rate=20000000 # 2MB/s 
  flush_throttle_timeout=10 
  max_packet_msg_payload_size=10240 # 10KB 
``` 



# Peer Exchange (PEX) Reactor 

Tendermint borrows from Bitcoin’s peer discovery protocol. More specifically, Tendermint adopts the p2p AddressBook from btcd, the Bitcoin alternative implementation in Go. The PEX enables dynamic peer discovery by default. 
  
PEX (Peer Exchange) gossips to known peers in the address book asking for address book update.  Because PEX exchanges data between the peers, we need reactor implementation so that switch can divert the traffic correctly. PexReactor accesses information of known seeds in the system and tracks which were already contacted for address book update. 

Seeds are the first point of contact for a new node. They return a list of known active peers and then disconnect. Seeds should operate full nodes with the PEX reactor in a "crawler" mode that continuously explores to validate the availability of peers.

In order to limit traffic exchange and prevent amplification attacks, PexReactors keeps track of requests it made and responses it sent, and also which nodes it communicated. 

PexReactor can react to get address request made by other peers, it then simply calls its address book and responds with random set of known peer addresses. Also when PEX is looking for new addresses and sends get address requests, PexReactor reacts to get address response by calling address book and updating it with new addresses. 
   
Running PEX peer discovery mechanism is not compulsory, Tendermint supports persistent peers communication mode. By adding a list of known node addresses to the config file, node will not ask other peers for address book update, however it can still respond to other curious nodes that does not run in persistent nodes mode. Tendermint also allows to switch off PEX completely. Such node won't participate in peer address book exchange, it won't be asking for new nodes nor it will respond with its current addres book state.   
 
#### Preventing SPAM
There are various cases where we decide a peer has misbehaved and we disconnect from them. When this happens, the peer is removed from the address book and black listed for some amount of time. We call this "Disconnect and Mark". Note that the bad behaviour may be detected outside the PEX reactor itself (for instance, in the mconnection, or another reactor), but it must be communicated to the PEX reactor so it can remove and mark the peer.

In the PEX, if a peer sends us an unsolicited list of peers, or if the peer sends a request too soon after another one, we Disconnect and MarkBad.




# P2P Workflow

Once peers are known, they can be dialled in. Tendermint P2P exchange is managed by the Switch object.  The Switch handles peer connections and exposes an API to receive incoming messages on Reactors. Each Reactor is responsible for handling incoming messages of one or more Channels. So while sending outgoing messages is typically performed on the peer, incoming messages are received on the reactor. Reactor expose Receive method that is called on particular reactor every time MConn receives new data.

Peers are dialled in with DialPeersAsync method called on the switch which dials a list of known peers from the address book asynchronously in random order. When dialling, switch creates Peer object and MConnection (MConn, multiplex connection) instance for each peer. 
 
MConnection is a multiplex connection that supports multiple independent streams with distinct quality of service guarantees atop a single TCP connection. Each stream is known as a Channel and each Channel has a globally unique byte id. Each Channel also has a relative priority that determines the quality of service of the Channel compared to other Channels. The byte id and the relative priorities of each Channel are configured upon initialization of the connection. Inbound message bytes are handled with an onReceive callback function. 
 
How `Mconnection` works? 
It has reader, writer (buffers), registered channels to read for outbound data transmission and onReceive handler.   
When MConn is initialised it spins receive goroutine that reads from the buffer and calls onReceive method that calls Receive method on reactor registered on that switch. 
Mconn also starts send goroutine that listens on the send `Channel` and writes to te buffer sending data to the peers. 



# Secure P2P 

The Tendermint p2p protocol uses an authenticated encryption scheme based on the Station-to-Station Protocol. The implementation uses golang's nacl box for the actual authenticated encryption algorithm. 
 
Each peer generates an `ED25519` key-pair to use as a persistent (long-term) id. 
 
When two peers establish a TCP connection, they first each generate an ephemeral `ED25519` key-pair to use for this session, and send each other their respective ephemeral public keys. This happens in the clear. 
 
They then each compute the shared secret. The shared secret is the multiplication of the peer's ephemeral private key by the other peer's ephemeral public key. The result is the same for both peers by the magic of elliptic curves. The shared secret is used as the symmetric key for the encryption algorithm. 
 
The two ephemeral public keys are sorted to establish a canonical order. Then a 24-byte nonce is generated by concatenating the public keys and hashing them with Ripemd160. Note Ripemd160 produces 20byte hashes, so the nonce ends with four 0s. 
 
The nonce is used to seed the encryption - it is critical that the same nonce never be used twice with the same private key. For convenience, the last bit of the nonce is flipped, giving us two nonces: one for encrypting our own messages, one for decrypting our peer's. Which ever peer has the higher public key uses the "bit-flipped" nonce for encryption. 
 
Now, a challenge is generated by concatenating the ephemeral public keys and taking the SHA256 hash. 
 
Each peer signs the challenge with their persistent private key, and sends the other peer an AuthSigMsg, containing their persistent public key and the signature. On receiving an AuthSigMsg, the peer verifies the signature. 
 
The peers are now authenticated. 
 
All future communications can now be encrypted using the shared secret and the generated nonces, where each nonce is incremented by one each time it is used.  
 
 
 
# Running Full Nodes

A new node needs a few things to connect to the network:
- a list of seeds, which can be provided to Tendermint via config file or flags, or hardcoded into the software by in-process apps
- a ChainID, also called Network at the p2p layer
- a recent block height, H, and hash, HASH for the blockchain.

The values H and HASH must be received and corroborated by means external to Tendermint, and specific to the user - ie. via the user's trusted social consensus. This requirement to validate H and HASH out-of-band and via social consensus is the essential difference in security models between Proof-of-Work and Proof-of-Stake blockchains.

With the above, the node then queries some seeds for peers for its chain, dials those peers, and runs the Tendermint protocols with those it successfully connects to. When the peer catches up to height H, it ensures the block hash matches HASH. If not, Tendermint will exit, and the user must try again - either they are connected to bad peers or their social consensus is invalid.

In case a node is restarted, such node checks its address book on startup and attempts to connect to peers from there. If it can't connect to any peers after some time, it falls back to the seeds to find more. Restarted full nodes can run the blockchain or consensus reactor protocols to sync up to the latest state of the blockchain from wherever they were last. 






# Validators 
 
A validators are active participants in the consensus with a cryptographic key-pair and an associated amount of "voting power". Voting power need not be the same.  Validators are responsible for validating new blocks proposed during consensus. They participate in the consensus by broadcasting votes which contain cryptographic signatures signed by each validator's private key. Not every node has to be a validator. Initial set of Validators are defined in the genesis.json file.
 
A node may not have a corresponding validator private key, but it nevertheless plays an active role in the consensus process by relaying relevant meta-data, proposals, blocks, and votes to its peers. A node that has the private keys of an active validator and is engaged in signing votes is called a validator-node. All nodes (not just validator-nodes) have an associated state (the current height, round, and step) and work to make progress.
Between two nodes there exists a Connection, and multiplexed on top of this connection are fairly throttled Channels of information. An epidemic gossip protocol is implemented among some of these channels to bring peers up to speed on the most recent state of consensus. 

For example,
- Nodes gossip PartSet parts of the current round's proposer's proposed block. A LibSwift inspired algorithm is used to quickly broadcast blocks across the gossip network.
- Nodes gossip prevote/precommit votes. A node `NODE_A` that is ahead of `NODE_B` can send `NODE_B` prevotes or precommits for - `NODE_B's` current (or future) round to enable it to progress forward.
- Nodes gossip prevotes for the proposed PoLC (proof-of-lock-change) round if one is proposed.
- Nodes gossip to nodes lagging in blockchain height with block commits for older blocks.
- Nodes opportunistically gossip HasVote messages to hint peers what votes it already has.
- Nodes broadcast their current state to all neighboring peers (but is not gossiped further).

Tendermint advises to run validator nodes independently with highest security in mind. Validators should not accept incoming connections, and should maintain outgoing connections to a controlled set of ‘Sentry Nodes’ that serve as their proxy shield to the rest of the network.

Sentry nodes are guardians of a validator node and provide it access to the rest of the network. They should be well connected to other full nodes on the network. Sentry nodes may be dynamic, but should maintain persistent connections to some evolving random subset of each other. They should always expect to have direct incoming connections from the validator node and its backup(s). They do not report the validator node's address in the PEX and they may be more strict about the quality of peers they keep.


# Rotating Leader Election

Tendermint rotates through the validator set, i.e. block proposers, in a weighted round-robin fashion. The more stake, i.e. voting power, that a validator has delegated to them, the more weight that they have, and the proportionally more times they will be elected as leaders. To illustrate, if one validator has the same amount of voting power as another validator, they will both be elected by the protocol an equal amount of times. 
 
More information on the algorithm for leader election can be found on tendermint [documentation](https://github.com/tendermint/tendermint/blob/master/docs/spec/reactors/consensus/proposer-selection.md)


Byzantine Consensus Algorithm 
Tendermint is, mostly asynchronous, BFT consensus protocol. The protocol follows a simple state machine that looks like this: 
![N|Solid](https://tendermint.com/docs/assets/img/consensus_logic.e9f4ca6f.png)


Participants in the protocol are called validators; they take turns proposing blocks of transactions and voting on them. Blocks are committed in a chain, with one block at each height. A block may fail to be committed, in which case the protocol moves to the next round, and a new validator gets to propose a block for that height.  Two stages of voting are required to successfully commit a block; we call them pre-vote and pre-commit. A block is committed when more than ⅔  of validators pre-commit for the same block in the same round (and ⅔+ we call a supermajority). 
 
There is a picture of a couple doing the polka because validators are doing something like a polka dance. When more than two-thirds of the validators pre-vote for the same block, we call that a polka. Every pre-commit must be justified by a polka in the same round. 
 
Validators may fail to commit a block for a number of reasons; the current proposer may be offline, or the network may be slow. Tendermint allows them to establish that a validator should be skipped. Validators wait a small amount of time to receive a complete proposal block from the proposer before voting to move to the next round. This reliance on a timeout is what makes Tendermint a weakly synchronous protocol, rather than an asynchronous one. However, the rest of the protocol is asynchronous, and validators only make progress after hearing from more than two-thirds of the validator set. A simplifying element of Tendermint is that it uses the same mechanism to commit a block as it does to skip to the next round.

So let’s dive in a bit deeper to understand how consensus algorithm works.


# State Machine Overview
At each height of the blockchain a round-based protocol is run to determine the next block. Each round is composed of three steps (Propose, Prevote, and Precommit), along with two special steps Commit and NewHeight. NewHeight, Propose, Prevote, Precommit, and Commit represent state machine states of a round. (aka RoundStep or just "step").

In the optimal scenario, the order of steps is:

```NewHeight -> (Propose -> Prevote -> Precommit)+ -> Commit -> NewHeight ->...```


The sequence ```(Propose -> Prevote -> Precommit)``` is called a round. There may be more than one round required to commit a block at a given height. Examples for why more rounds may be required include:
- The designated proposer was not online.
- The block proposed by the designated proposer was not valid.
- The block proposed by the designated proposer did not propagate in time.
- The block proposed was valid, but `+2/3` of prevotes for the proposed block were not received in time for enough validator nodes by the time they reached the Precommit step. Even though `+2/3` of prevotes are necessary to progress to the next step, at least one validator may have voted `<nil>` or maliciously voted for something else.
- The block proposed was valid, and `+2/3` of prevotes were received for enough nodes, but `+2/3` of precommits for the proposed block were not received for enough validator nodes.

Some of these problems are resolved by moving onto the next round & proposer. Others are resolved by increasing certain round timeout parameters over each successive round. Depending on the configuration of `EmptyBlock`, propose round may run only when there are transactions in the mempool. By default it will create empty blocks.

Each node entering round has its own timeout set up for each round. Upon entering Propose, the designated proposer proposes a block at `(H,R)`. In case of `timeoutProposer`, we come back to propose state. After receiving proposal block and all prevotes at supermajority we progress to `Prevote(H,R)`
 
Upon entering Prevote, each validator broadcasts its prevote vote. If the proposed block from `Propose(H,R)` is good, it prevotes that. Else, if the proposal is invalid or wasn't received on time, it prevotes `<nil>`.

The Prevote step ends: After `+2/3` prevotes for a particular block or nil and in case of `timeoutPrevote`. Round is then moved to `Precommit(H,R)`.

Upon entering `Precommit`, each validator broadcasts its precommit vote. A precommit for `<nil>` means "I didn’t see a PoLC for this round, but I did get `+2/3` prevotes and waited a bit".

Precommit step ends after timeoutPrecommit and nodes not receiving `+2/3` on any block. Also precommit step can end after `+2/3` precommits for `<nil>`. In those two cases nodes did not reach consensus and new round is started at Propose(H,R+1). Precommit step can end successfully after timeoutPrecommit and receiving `+2/3` precommits for a particular block. Then round is moved to the final stage `Commit(H,R)`
 
Commit step executes the block increasing the height of the blockchain and closing the round. It also calls mempool to remove commited transactions from the mempool.

Assuming less than one-third of the validators are Byzantine, Tendermint guarantees that safety will never be violated - that is, validators will never commit conflicting blocks at the same height. To do this it introduces a few locking rules which modulate which paths can be followed in the flow diagram. Once a validator precommits a block, it is locked on that block. Then, 
1. it must prevote for the block it is locked on 
2. it can only unlock, and precommit for a new block, if there is a polka (supermajority) for that block in a later round

Important to note is that every validator votes, gossips on its decision and than tracks number of votes for each block. Supermajority is calculated based on the validators weighted average, weighted by the voting power.



# Stake 
 
In many systems, not all validators will have the same "weight" in the consensus protocol. Thus, we are not so much interested in one-third or two-thirds of the validators, but in those proportions of the total voting power, which may not be uniformly distributed across individual validators. 
 
Since Tendermint can replicate arbitrary applications, it is possible to define a currency, and denominate the voting power in that currency. When voting power is denominated in a native currency, the system is often referred to as Proof-of-Stake. Validators can be forced, by logic in the application, to "bond" their currency holdings in a security deposit that can be destroyed if they're found to misbehave in the consensus protocol. This adds an economic element to the security of the protocol, allowing one to quantify the cost of violating the assumption that less than one-third of voting power is Byzantine. 

# Consensus Reactor

Consensus Reactor defines a reactor for the consensus service. It contains the ConsensusState service that manages the state of the Tendermint consensus internal state machine. When Consensus Reactor is started, it starts Broadcast Routine which starts ConsensusState service. Furthermore, for each peer that is added to the Consensus Reactor, it creates (and manages) the known peer state (that is used extensively in gossip routines) and starts the following three routines for the peer p: Gossip Data Routine, Gossip Votes Routine and QueryMaj23Routine. Finally, Consensus Reactor is responsible for decoding messages received from a peer and for adequate processing of the message depending on its type and content. The processing normally consists of updating the known peer state and for some messages (ProposalMessage, BlockPartMessage and VoteMessage) also forwarding message to ConsensusState module for further processing. 

# Consensus State Service

Consensus State handles execution of the Tendermint BFT consensus algorithm. It processes votes and proposals, and upon reaching agreement, commits blocks to the chain and executes them against the application. The internal state machine receives input from peers, the internal validator and from a timer. 

Inside Consensus State we have the following units of execution: Timeout Ticker and Receive Routine. Timeout Ticker is a timer that schedules timeouts conditional on the height/round/step that are processed by the Receive Routine. Receive Routine of the ConsensusState handles messages which may cause internal consensus state transitions. It is the only routine that updates RoundState that contains internal consensus state. Updates (state transitions) happen on timeouts, complete proposals, and 2/3 majorities. It receives messages from peers, internal validators and from Timeout Ticker and invokes the corresponding handlers, potentially updating the RoundState. 


# WAL

Consensus module writes every message to the WAL (write-ahead log). WAL is a replay mechanism to synch chains and plays crucial role in crash recovery. Wal writes RoundStateEvent structure that has all information about RoundState and therefore consensus process of a proposed block. Whenever new phase of consensus is achieved (propose, preVote, preCommit, Commit) consensus.NewStep() is called which writes to WAL current consensus RoundState.
Replay

Consensus module will replay all the messages of the last height written to WAL before a crash (if such occurs).
The private validator may try to sign messages during replay because it runs somewhat autonomously and does not know about replay process. For example, if we got all the way to precommit in the WAL and then crash, after we replay the proposal message, the private validator will try to sign a prevote. But it will fail. That's ok because we’ll see the prevote later in the WAL. Then it will go to precommit, and that time it will work because the private validator contains the LastSignBytes and then we’ll replay the precommit from the WAL.

# State 

State is a short description of the latest committed block of the Tendermint consensus. It keeps all information necessary to validate new blocks, including the last validator set and the consensus params. When proposalBlock is made new Block is created from the current state instance. 
 
BlockExecutor handles block execution and state updates. It exposes `ApplyBlock()`, which validates & executes the block executes against the ABCI app (BeginBlockSync, DeliverTxAsync, EndBlockSync), saves ABCI responses, then commits and updates the mempool atomically, then saves state. Mempool is updated with transactions that were committed so that mempool can remove committed transactions from memory. 
 
Consesus Config
```go
consensus.skip_timeout_commit
```
Node is sleeping between gossip rounds
```go
consensus.peer_gossip_sleep_duration 
```
Sleep time before proposing next block 
```go
consensus.timeout_commit
```
Maximum number of transactions per block 
```go
consensus.max_block_size_txs
```
![](https://tendermint.com/docs/assets/img/tm-transaction-flow.258ca020.png)
