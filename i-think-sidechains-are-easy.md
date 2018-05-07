# I think sidechains are easy.
## The second part of an exploration of what MVP sidechains on ETC might look like.

I wrote an [introduction to thinking about sidechains](https://medium.com/etcdev/exploring-minimum-viable-sidechains-on-etc-3f4b06246aaf) that documented some of my preliminary and exploratory sketches for working with sidechains with _adhoc_ solutions in mind: Can we build sidechains with the tools that we have already? What might they look like, and what can we learn or induce about best next-steps for inter-chain operability?

I think now that sidechains are even easier than I thought.

I had outlined a couple of critical elements for a sidechain:
1. (Custom) consensus. Probably PoA, but the door is open. Also not even technically a requirement.
2. Chain-to-chain communication.
3. ~~Checkpoints~~ This, in my mind, no longer qualifies as a requirement since _arbitrary_ data can be posted to or queried from mainnet. A sidechain use-case doesn't _have to_ use checkpoints specifically to be a sidechain.

Adhoc __custom consensus__ has been proven with [Tx2PoA](https://github.com/ETCDEVTeam/sidekick-tx2poa). While imperfect, it's a working proof-of-concept. It could, for example, be improved:
- add "electable" authority candidacy/sealer-eligibility procedure (which would imitate the "ducks-in-a-row" pattern used, for example, on Ropsten at least). This would reduce the number of required "incomplete authority proof" transactions to one per block.
- work more effectively around the limitation of `debug_setHead` as a sole way of purging invalid blocks, which _might_ mean augmenting or otherwise improving the client API around block validation
- add additional consensus-transaction metadata to communicate network node states, block states, checkpoint blocks and verifications, and improved misbehaving peer/attacker responses
- refactoring, make it available on Docker, ...

But these bullet points are not critical to establishing a viable alternative consensus mechanism with Javascript. At the time I wrote that article, __inter-chain communication__ was the weak link.

> Chain-to-chain communication. For sidechains, this will mean establishing the mechanism that is able to inform block validation relative to the state of the mainnet. As a client/protocol implementation this will likely mean changes or additions to RLPx protocol, p2p patterns, and the refactoring of clients towards simultaneous multi-chain support. Our implementation will outsource this responsibility to a “sidecar” application.

> ... we’re going to use a simple external script liaison.sh to handle the inter-chain communication responsibility.

[This design](https://github.com/ETCDEVTeam/sidekick-liaison/blob/master/liaison.sh) had placed a lot of emphasis on the logic of the liaison script -- using [emerald-cli](https://github.com/ETCDEVTeam/emerald-cli) to sign and post upstream transactions, `sleep`s and incrementing `while` loops to handle their response(s), and, effectively and in essence, exclusively handling the logic around managing the relevant contract and inter-chain transaction data.

This new design is able to trim the 110 lines of complex (or at least verbose) bash code to approximately 10 simple lines -- or the equivalent of a blistering one-liner -- and enables the CALL-RESPONSE logic to be shifted back into the Javascript code to be run by an ephemeral ETC client console, eg. [geth](https://github.com/ethereumproject/go-ethereum/).

The benefit of this is that transactional request and response data can be handled with native `web3` or auxiliary Javascript logic purpose-built for this, and that the consensus and inter-chain-communication mechanisms can readily work hand-in-hand.

Before going ahead I want to emphasize that the following code snippets represent an opinion toward proving a generic pattern, as opposed to cut-and-pastes from production-ready code. There are lot of ways to modify, improve, and custom-fit these solutions, and they should be considered more as hints or clues than deliverable examples.

### First steps: `console.log && while read`

Geth's ephemeral JS console writes it's `console.log` statements to the processes's `stdout`. This means that we can readily use a pattern like this:

```shell
$ geth --exec="console.log('hello');" console 2> /dev/null | while read -r line; do echo "got $line"; done
got hello
got undefined
```


> Note that I'm just sending `stderr` to `/dev/null` because it makes the snippet more legible. You don't have to redirect `stderr` otherwise (and probably better not to so you can see normal client display logging like normal).

And if we want to get a little more fancy, we can handle our incoming line with some structure thanks to bash's `while` pairing with (the default) `IFS` variable:

```shell
$ geth --exec="console.log('hello', 'there');" console 2> /dev/null | while read -r a b; do echo "got a=$a b=$b"; done
got a=hello b=there
got a=undefined b=
```

> For a great resource introducing some bash one-liner magic, check out [http://www.catonmat.net/blog/bash-one-liners-explained-part-one/](http://www.catonmat.net/blog/bash-one-liners-explained-part-one).

I started using this pipe pattern with an adjacent "upstream" mainnet client to be the recipient of a "sidestream" sidenet client over IPC.

By default, a mainnet geth will create and expose it's entire API over `<datadir>/mainnet/geth.ipc`. So to put the sidenet node in touch with the mainnet node, you'd just start them both up and use something like

```js
// sendMainnetTx.js
console.log(JSON.stringify({"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":[{
  "data": "0xd46e8dd67c5d32be8d46e8dd67c5d32be8058bb8eb970870f072445675058bb8eb970870f072445675"
}],"id":1}));
```

```shell
$ geth --chain sidenet js sendMainnetTx.js | while read -r tx; do  nc -U <datadir>/mainnet/geth.ipc <<< "$tx"; done
```

And now you've got the sidenet node POSTing (any) data to mainnet. But a POST is not enough; we've got to capture the response.

First, we'll make sure we persist the response somewhere:

```shell
$ geth --chain sidenet js sendMainnetTx.js | while read -r tx; do nc -U <datadir>/mainnet/geth.ipc <<< "$tx" > /icc/response/data.js;
done
```

Now, to _load_ the response back into sidenet geth, we can use the handy `loadScript`.

```js
// sendMainnetTx.js
var didLoad = loadScript("/icc/response/data.js");
```

There's a gotcha here. `loadScript` loads and _executes_ the text from the given file as Javascript. And since we've only got a JSON response sitting in `/icc/response/data.js`, we'll see that `didLoad === false`.

> PTAL: If there is a better way to successfully use a `require("/icc/response/data.js)`... I still haven't figured it out. Please let me know if you do; it would cut out this next ugly workaround.

The ugly workaround:

```shell
$ geth --chain sidenet js sendMainnetTx.js | while read -r tx; do
 echo -n "res=" > /icc/response/data.js # assuming js 'var res={};' has already been declared
 nc -U <datadir>/mainnet/geth.ipc <<< "$tx" >> /icc/response/data.js
 echo ";" >> /icc/response/data.js
done
```

So now in our Javascript file we'll get `didLoad === true` and a ready-to-use variable `res` with the JSON response data:

```json
{
  "id":1,
  "jsonrpc": "2.0",
  "result": "0xe670ec64341771606e55d6b4ca35a1a6b75ee3d5145a99d05921026d1527331"
}
```

That's about it.

If you wanted to actually use this pattern for sidechaining, you could just
1. post _arbitrary_ data (in this example maybe a hash of the latest "checkpoint" block and it's number) to a contract on mainnet by making a transaction like above,
~~2. wait for this transaction hash to come through in the form of the response~~
~~3. post another query (or series of) to ensure that the transaction is no longer pending and is sufficiently deep in the canonical chain~~
4. have any node post a query to get the current state of your newly-updated mainnet contract,
5. use the response about the current state of your contract on mainnet to verify and validate the progress of your sidechain (a check which could readily be done by any node on the sidenet)

A quick off-the-cuff example of a query any sidenet node can make upstream to derive and enforce checkpoint validation data:

```shell
var getUpstreamContract = buildGetMainnetContractCodeQuery();
console.log(getUpstreamContract); // sends to our fancy pipe reader
after(awaitReadResponse(getUpstreamContract),
 function(res) {
     var checkpointedUpstream = eth.call(res, ...);
     var checkpoints = parseCheckpoints(checkpointedUpstream);
     for block, hash in checkpoints {
         if (eth.getBlock(block).hash !== hash) {
             debug.setHead(block-checkpointInterval);
         }
     }
 }
);
```

### A few steps further

__ensure match call/responses__

If our sidenet Javascript wants to be interacting with the mainnet more than, say, once, it'll be prudent to add some logic to ensure that we're able to match calls 1:1 to their upstream response. The first low-hanging check is that the call/response `id` matches. After that, we could also use a custom uuid for each  query.

```js
// sendMainnetTx.js
var uuid = (Math.random()*0xFFFFFF<<0).toString(16);
var tx = {"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":[{
  "data": "0xd46e8dd67c5d32be8d46e8dd67c5d32be8058bb8eb970870f072445675058bb8eb970870f072445675"
}],"id":1};
console.log(uuid, JSON.stringify(tx));
awaitResponse(uuid, tx);
```

```shell
$ geth --chain sidenet js sendMainnetTx.js | while read -r uuid tx; do
 echo -n "var res=" > /icc/response/$uuid.js # or even "var res['"+$uuid+"']="
 echo -n "tx" | nc -U <datadir>/mainnet/geth.ipc >> /icc/response/$uuid.js
 echo ";' >> /icc/response/$uuid.js
done
```

__get a little fancy to persist or propagate arbitrary data__

```js
// upstreamTx.js
function send(typeof, target, data, out) {
 // eg args: "rpc", "http://gastracker.io/api", "{'jsonrpc': '2.0' ... }", "/icc/res/got.js"
 console.log(typeof, admin.nodeInfo.id, target, data, out);
}
```

```shell
$ geth --chain sidenet js sendMainnetTx.js | while read -r typeof origin target data respond_here; do
 case $typeof in
     ipc)
         echo -n "res=" > "$respond_here"
         echo -n "$data" | nc -U "$target" >> "$respond_here"
         echo ";" >> "$respond_here"
     rpc)
         echo -n "res=" > "$respond_here"
         curl -X POST --data "'""$data""'" "$target" >> "$respond_here"
         echo ";" >> "$respond_here"
     ws)
         # make websocket POST
     state)
         # persist arbitrary state between sessions (eg. requiredHash for checkpoint configuration)
     log)
         # handle logging
     err)
         # handle errors
         pkill -f geth -9
 esac
done
```


### problem/discussions

1.  __Reading a response from a file is clumsy.__ Yea, kind of. At least a working `require` would be nice, as mentioned above. Alternatively, it might not be too much trouble to implement an API method which would listen for arbitrary JSON-response calls via IPC/RPC and be able to delegate them then as "JSON responses" for handling. And on the upside of file-based reads, you first-class persist your stateful data and kind of follow what [Rob Pike might agree follows in a central realization of the Plan 9 OS: "\[...\] since the most important resources of the network are files, the model of that view is file-oriented."](https://9p.io/sys/doc/9.html)
2. __`debug_setHead`__ still looms as an inadequate way of marking a given block as "invalid" (but this has nothing to with inter-chain communication specifically).
3. It's still __kind of a sidecar__. Yea... but
    - It's pretty portable (windows excluded, of course)
    - It's a few lines and has arguably no business logic
    - It's hacker- and developer-friendly; no need to wait for the schmucks over at ECIP to get their human-consensus shit together; and between the JS and bash-pipe side it's not more complex than a basic web app.



