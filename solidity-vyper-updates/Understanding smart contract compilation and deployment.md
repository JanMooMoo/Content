# Understanding smart contract compilation and deployment

As discussed earlier in the series, when developing dapp and writing smart contracts, there are many repetitive tasks you will undertake. These include compiling source code, generating ABIs, testing, and deployment.

Development frameworks hide the complexity of these tasks and enable you as a developer to focus on developing your idea.

Before we take a look at frameworks such as [truffle](https://truffleframework.com/), [embark](https://embark.status.im/) and [populous](https://github.com/ethereum/populus), let's take a detour and have a look at the tasks performed and hidden by these frameworks. Understanding whats happening under the hood is useful when you run into issues or bugs with these frameworks.

This article walks you through how to manually compile and deploy your _Bounties.vy_ smart contract from the command line, to a local development blockchain.

## Steps

Before deployment, we need to encode a smart contract needs into EVM friendly binary called bytecode, much like a compiled Java class. The following steps typically need to take place before deploying a contract:

1.  Smart contract is written in a human friendly language (e.g Vyper). For this article, the [Bounties.vy](https://github.com/kauri-io/kauri-fullstack-dapp-tutorial-series/blob/master/manual-compilation-and-deploy/Bounties.vy) file contract
2.  The code is compiled into bytecode and a set of function descriptors (Application Binary Interface, known as ABI) by a compiler (e.g Vyper)
3.  The bytecode is packed with other parameters into a transaction
4.  The transaction is signed by the account deploying the contract
5.  The signed transaction is sent to the blockchain and mined

## Vyper Compiler

To compile Vyper we need to use the Vyper compiler. Typically frameworks such as [truffle](https://truffleframework.com/), [embark](https://embark.status.im/) and [populous](https://github.com/ethereum/populus) come with a version of Vyper preconfigured, since we are compiling without a framework, we need to install Vyper manually.

### Installing Vyper

#### Using pip

```shell
pip install Vyper
```

#### Installing JQ

To help with processing json content during compilation and deployment, install JQ
Using homebrew:

```shell
brew install jq
```

Or on ubuntu:

```shell
sudo apt-get install jq
```

[Read more about installing jq](https://stedolan.github.io/jq/download/)

### Compiling Vyper

Once we've installed Vyper we can now compile our smart contract. Here we want to generate

-   The bytecode (binary) to deploy to the blockchain
-   The ABI (Application Binary Interface) which tells us how to interact with the deployed contract

Let's setup our directory to work from and copy the _Bounties.vy_ smart contract

```shell
mkdir dapp-series-bounties
cd dapp-series-bounties
touch Bounties.vy
```

Now copy the contents of [Bounties.vy](https://github.com/kauri-io/kauri-fullstack-dapp-tutorial-series/blob/master/manual-compilation-and-deploy/Bounties.vy) into the file.

And compile it:

```shell
vyper -f json Bounties.vy
```

This command tells the compiler to combine both the abi and binary output into one json file, _Bounties.json_. To get the abi from the Vyper contract, use

```shell
vyper -f abi Bounties.vy
```

We can view the output using jq:

```shell
$ cat Bounties.json| jq

[
  {
    "name": "BountyIssued",
    "inputs": [
      {
        "type": "int128",
        "name": "_id",
        "indexed": false
      },
      {
        "type": "address",
        "name": "_issuer",
        "indexed": true
      },
      {
        "type": "uint256",
        "name": "_amount",
        "indexed": false,
        "unit": "wei"
      },
      {
        "type": "bytes32",
        "name": "data",
        "indexed": false
      }
    ],
    "anonymous": false,
    "type": "event"
  },...]
```

## Deployment

Now its time to deploy our smart contract!

The command `vyper Bounties.vy` returns the contract's bytecode which you can use to deploy through mist, geth or with myetherwallet.

## Next Steps

<!-- TODO: Update -->

-   Read the next guide: [Truffle: Smart Contract Compilation & Deployment](https://kauri.io/article/cbc38bf09088426fbefcbe7d42ac679f/truffle:-smart-contract-compilation-and-deployment)
-   Learn more about the Truffle suite of tools from the [website](https://truffleframework.com/)

> If you enjoyed this guide, or have any suggestions or questions, let me know in the comments.
>
> If you have found any errors, feel free to update this guide by selecting the **'Update Article'** option in the right hand menu, and/or [update the code](https://github.com/kauri-io/kauri-fullstack-dapp-tutorial-series/tree/master/truffle-compilation-and-deploy)
