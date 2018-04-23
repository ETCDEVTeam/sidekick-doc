# Exploring MVP Sidechaining on ETC

### Introducing sidechains


__What even is a sidechain?__ 

There are two critical pieces: 

1. A sidechain is a blockchain network that works very similarly to `mainnet`... _but isn't_. For whatever reason the sidechain is an insulated network that probably uses the same or similar technology as the mainnet. It could, for example, be run by a company or companies who want to distribute a consensus-based ledger/application framework, but don't want to be beholden to the unwashed and agnostic democracy that govern the economics and politics of the `mainnet`.
2. A sidechain _talks to_ `mainnet`. This is what makes a sidechain a sidechain, instead of a plain old private network. By talking to the mainnet, the sidechain is able to balance the advantages a private network with the security of the mainnet. 


__So what are the advantages (or disadvantages) of a sidechain?__

_Pros:_

1. Its cheaper. You don't have to pay market prices for gas.
2. You (or a select group) are in charge. If you want to hard fork you don't have to convice Vitalik... unless you are Vitalik.
3. You can use new or different technologies than mainnet. Want to use a different consensus method like Proof of Authority/Stake/Vita-like? Go for it. It's your chain.

_Cons:_

1. Security. Despite the smell, the unwashed democracy of the mainnet is exactly what makes it safe. A private chain is by definition not (as) distributed or diverse as a private network.


Addressing `Con#1` is where the whole networks-talking-to-each-other thing comes in. If it's too expensive/slow/annoying to use mainnet for every transaction... let's just use it for _some_ transactions. If the sidechain can once in a while tell mainnet: "Hey, here's an update about what our network looks like now," then it'll have a secure and permanent record of what the state of affairs _should look like_ on the sidechain at a given point. This is called a _checkpoint_ pattern.


