# Exploring MVP Sidechains on ETC

## Introducing sidechains

#### What even is a sidechain?

There are two critical pieces: 

1. A sidechain is a blockchain network that works very similarly to `mainnet`... _but isn't_. For whatever reason the sidechain is an insulated network that probably uses the same or similar technology as the mainnet. It could, for example, be run by a company or companies who want to distribute a consensus-based ledger/application framework, but don't want to be beholden to the unwashed and agnostic democracy that govern the economics and politics of the `mainnet`.
2. A sidechain _talks to_ `mainnet`. This is what makes a sidechain a sidechain, instead of a plain old private network. By talking to the mainnet, the sidechain is able to balance the advantages a private network with the security of the mainnet. 


#### So what are the advantages (or disadvantages) of a sidechain?

_Pros:_

1. Its cheaper. You don't have to pay market prices for gas.
2. You (or a select group) are in charge. If you want to hard fork you don't have to convice Vitalik... unless you are Vitalik.
3. You can use new or different technologies than mainnet. Want to use a different consensus method like Proof of Authority/Stake/Vita-like? Go for it. It's your chain.

_Cons:_

1. Security. Despite the smell, the unwashed democracy of the mainnet is exactly what makes it safe. A private chain is by definition not (as) distributed or diverse as a private network.

Addressing `Con#1` is where the whole networks-talking-to-each-other thing comes in. If it's too expensive/slow/annoying to use mainnet for every transaction... let's just use it for _some_ transactions. If the sidechain can once in a while tell mainnet: "Hey, here's an update about what our network looks like now," then it'll have a secure and permanent record of what the state of affairs _should look like_ on the sidechain at a given point. This is called a _checkpoint_ pattern.

#### Data flows

If the sidenet were _only to POST data_ to the mainnet, and never depend on querying that data again, there would indeed be a secure _record_ of the sidechain history... but there's nothing to ensure that the sidechain history -- the record itself -- would actually have integrity relative to the mainnet. 

The mainnet's integrity is a function of necessitating sequential congruence; the value of block B depends on the value of block A, and everyone's got to agree (or at least not rebut) that the given pieces do actually fit together. Chain progression and integrity is derived from sequential consensus.

So without checking -- and relying on -- the checkpoint data stored on mainnet, the sidenet is missing out on the security of the consensus mechanisms of the mainnet. If it doesn't depend data from the mainnet, then it doesn't get any of the value of putting data on mainnet in the first place. I repeat myself. That part is important.

For further reference, let's call a pattern where sidenet does _POST only_ to mainnet a __unilateral__ communication mechanism, and the pattern with _POST and QUERY_ a __bilateral__ mechanism.


## Building a ~~dumbest possible~~ minimum-viable sidechain

#### Why minimum-viable?

- Because I'm kind of slow and want to mess around with the concepts before tangling too much with client/protocol consensus-facing code.
- Because changes to client code are more complex and have external dependencies. Code must be written, refactored, rewritten, reviewed, heckled, rebased, merged, tagged, and released. Documentation needs to be documented, clients updated, tests tested, bugs fixed... and so forth. Eventually the best solution(s) _will_ involve significant and diverse changes to the client(s), but for now we're still in the "what does this even mean" phase, and a minimum-viable proof-of-concept should approach being runnable by anybody _now_... if possible. 
- And finally, because it's fun to see how we could bend already-existing stuff to make it do new weird stuff.

#### Conceptual requirements

There are a few key challenges in developing a sidechain-ready client.

__Consensus__. It's likely that on the side network it won't be desirable to use Proof of Work (at least for the immediate use-cases -- it doesn't take too far of a visionary leap to imagine a range of reasons and contexts accomodating the gamut of consensus patterns). First use-cases are insulated, federated, and are eager to gain the benefits that come without reverence to fully distributed consensus. A likely and probably first-use case is Proof of Authority, so we'll tend that direction in this experiment.

__Chain-to-chain communication__. For sidechains, this will mean establishing the mechanism that is able to inform block validation relative to the state of the mainnet. As a client/protocol implementation this will likely mean changes or additions to RLPx protocol, p2p patterns, and the refactoring of clients towards simultaneous multi-chain support.

__Checkpoints__. Tying together points `1` and `2`, implementing checkpoints will be in large part a function of how we decide to roll consensus and communication between chains. The interesting part will be building the logistics a system that can be used to efficiently "fingerprint" a state of affairs on the sidenet, store that data in the mainnet, and then use a query from mainnet of that data to validate the integrity of the sidenet later on. There will be hashes. There will be signatures. And, since we're ETC, there will probably be a smart contract or two.

> Aside: IMO it's really dumb that they're called smart contracts. They're not smart and they're not really contracts. They're just programs.

#### Things we'll build

1. A Proof of Authority mechanism to solve the `Consensus` requirement. 
2. A mechanism to enable chain-to-chain communication. 
3. A mechanism to create and validate checkpoints on the sidechain.
4. Smart contracts for mainnet and sidenet to store and delegate checkpoint fingerprints and fingerprint validation.

> TODO: Write these smart contracts. The PoC is far stronger with viable contracts to make data handling exemplary and explicit.

Please note that in some aspects these projects are interdependent. For example, the checkpoint mechanism will be interdependent with the consensus mechanism, since both require and facilitate block validation. And chain-to-chain communication will be dependent on the the timing and logistics of the checkpointing scheme, so those will need to integrate smoothly as well.

The chain-to-chain "liason" is the least natively-accessible challenge for pre-existing clients, and to address it we'll write a tiny third-party "sidecar" application that will live next door to the client.

### Getting started

#### Geth's JS Console

As a primary tool for messing with the [go-ethereum client (aka Geth)](https://github.com/ethereumproject/go-ethereum)'s consensus mechanism and checkpointing, I'm going to use the built-in Javascript Console tools. These tools are closely related with the JSON-RPC API, and allow interactive and/or programmatic interfacing with a running client via WS, RPC, or IPC.

> TODO: Is that right? RPC, really?

Geth has three subcommands built around the JS console: `console`, `attach`, and `js`.

1. `$ geth console` starts geth and begins an _interactive_ JS console session. 
2. `$ geth attach` connects to an _already-running_ geth client and then begins an interactive session.
3. `$ geth js program.js` begins an _ephemeral_ JS session, running a given `program.js` _without interactivity_.

One of the nuances of geth, and a good friend of ours on this journey, is that __geth sends all normal logs to `stderr`__, but __JS console output goes to `stdout`__. This difference of data stream contexts will play a tiny but important role for us.

There's a lot of options and different ways to use these three subcommands, but, while tempting, a deep dive in that direction is out of scope here. I hope the brief associated commentary in the following examples are sufficient for the purpose at hand.

> TODO: Add example session.

> TODO: Add link to docs.

> TODO: Touch on _how_ the JS console command connects and relates to a geth instance materially, eg what even is IPC and why does it matter.

#### A ~~hacky bash script~~ "sidecar" application

One of the most significant limitations of geth's console and it's primary facilitator `web3.js` is the notable absense of web-based calls. The libraries don't include standard web protocol tools akin to `AJAX` or `curl`. This is because the web is a murky, untrustworthy place, filled with fake news and flip facebook friends, and the thought is that it's kind of antithetical to the design intentions for the fundamentals of a blockchain.

> There are a few approaches to solutions around this limitation, like "oracles" and even a few distributed protocols that hope to integrate the two worlds in a trustable/trustless way, and in the future might be an interesting avenue of further exploration as far as chain integrations initiated from within the content of the blockchain.

Web3 aside, current clients are designed to initialize their configuration and behavior around a single `network_id` value to differentiate and identify nodes participating in a matching chain. Support for a range of chains is on the horizon, but not yet actualized, meaning that client p2p protocols can't be readily used to communicate between chains.

So for the time being we're going to use a simple external application `liason` to handle the inter-chain communication responsibility. 

The `liason` program has three primary event-based behaviors to implement:

1. On a sidenet checkpoint-creation event, POST data delegated from a sidechain node to a mainnet endpoint. For this example, this will mean __POSTing a transaction__ containing a fingerprint hash of a sidechain block or state to a contract address on the mainnet.
2. IFF the _success_ of `1`, POST a transaction to a corresponding smart contract on the sidechain providing validation of the successful integration.
3. IFF the _failure_ of `1`, do not provide the validation. This could be accomplished either with a transaction to the sidechain providing a negative status and associated error data around the failure of `2`, or, more simply, by the notable absence of the proof-positive expected from `2`.

__NOTE__ that the `liason` application bears _a lot_ of responsibility. It's a lynch-pin, and as engineers we're going to place as much trust in the functionality of this program as we do in the reliability of the chain client.



