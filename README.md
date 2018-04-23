# What sidechains mean to me


__What even is a sidechain?__ Let's start with the obvious; there are two critical pieces: 

1. A sidechain is a blockchain network that works very similarly to `mainnet`... _but isn't_. For whatever reason the sidechain is an insulated network that probably runs the same or similar technology as the mainnet. It could, for example, be run by a company or companies who want to distribute a consensus-based ledger/application framework, but don't want to be beholden to the unwashed and agnostic democracy that governs the economics and politics of the `mainnet`.
2. A sidechain _talks to_ `mainnet`. This is what makes a sidechain a sidechain, instead of a plain old private network. By talking to the mainnet, the sidechain is able to balance the advantages a private network with the security of the mainnet. 

I personally run tiny networks like this all the time in development, and we at ETCDEV use tiny private networks for testing integrating/use-case code changese that would impact client protocol and "plays well with others" fixes, improvements, and changes.

Here's an example of a private network that can be programmatically configured and deployed to a k8s cloud service:

> https://github.com/ETCDEVTeam/ecip1041test


