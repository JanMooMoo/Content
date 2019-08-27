# Sending Ethereum Transactions with Rust

This tutorial will walk you through the code required to send an Ethereum transaction within a Rust application.

## Prequisites

We assume that you already have a Rust IDE available, and have a reasonable proficiency in Rust programming.  It also assumes some basic knowledge of Ethereum and does not cover concepts such as the contents of an Ethereum transaction.  

## Libraries Used

We will be heavily using the MIT licensed rust-web3 library of which sourcecode can be found [here](https://github.com/tomusdrw/rust-web3).

To use this library in your application, it must be added to your `Cargo.toml` file:

```toml
[dependencies]
web3 = { git = "https://github.com/tomusdrw/rust-web3" }

```

The library can then be added to your crate:

```rust
extern crate web3;
```

## Starting an Ethereum Node

We need access to a node that we can connect to in order to send transactions.  In this tutorial we will use `ganache-cli`, which will allow you to quickly start a personal Ethereum network, with a number of unlocked and funded accounts.

Taken from the `ganache-cli` [installation documentation](https://github.com/trufflesuite/ganache-cli#installation), to install with npm, use the command:

```
npm install -g ganache-cli
```

or if you prefer to use Yarn:

```
yarn global add ganache-cli
```

Once installed, simply run the command, to quickly start a private Ethereum test network:

```
ganache-cli -d
```

Note: The `-d` argument instructs `ganache-cli` to always start with the same accounts pre-populated with ETH.  This will be useful in the Raw Transaction section of this tutorial as we will know the private keys of these accounts.

## Sending a Transaction from a Node-Managed Account

The easiest way to send a transaction is to rely on the connected Ethereum node to perform the transaction signing.  This is generally a less secure approach however, as it relies on the account being "unlocked" on the node.

### Required `Use` Declarations

```rust
use web3::futures::Future;
use web3::types::{TransactionRequest, U256};
```

### Connecting to the Node

```rust
let (_eloop, transport) = web3::transports::Http::new("http://localhost:8545").unwrap();

let web3 = web3::Web3::new(transport);
```

First we must create a transport object that will be used to connect to the node. In this example we are going to connect via `http`, to `localhost` on port `8545`, which is the default port for Ganache, and most if not all Ethereum clients.

**Note:** An EventLoop is also created, but that is out of the scope of this guide.

Next we construct a web3 object, passing in the previously created transport variable, and thats it!  We have now have a connection to the Ethereum node!

### Obtaining Account Details

Ganache-cli automatically unlocks a number of accounts and funds them with 100ETH, which is useful for testing.  The accounts differ on every restart though, so we need a way to programmatically obtain the account information:

```rust
let accounts = web3.eth().accounts().wait().unwrap();

```

The [Eth namespace](https://tomusdrw.github.io/rust-web3/web3/api/struct.Eth.html), obtained via `web3.eth()` contains many useful functions for interacting with the Ethereum node.  Obtaining a list of managed accounts via `accounts()` is one of them.  An asynchronous future is returned, so we wait for the task to complete (`wait()`), and obtain the result (`unwrap()`).

### Sending the Transaction

The parameters of the transaction to be sent is defined via a `TransactionRequest` structure:

```rust
let tx = TransactionRequest {
        from: accounts[0],
        to: Some(accounts[1]),
        gas: None,
        gas_price: None,
        value: Some(U256::from(10000)),
        data: None,
        nonce: None,
        condition: None
    };
```

Most of the fields within this struct are optional, with sensible default values being used if not manually specified.  As we are sending a simple ETH transfer transaction, the data field will be empty, and in this example we use the default `gas` and `gas_price` values.  We also do not specify a `nonce`, as the `rust-web3` library will query the Ethereum client for this latest nonce value by default.  The `condition` is a `rust-web3` specific field and allows you to delay sendind the transaction until a certain condition is met, such as a specific block number being reached for example.

Once the `TransactionRequest` has been initiated, its a one-liner to send the transaction:

```rust
let tx_hash = web3.eth().send_transaction(tx).wait().unwrap();
```

The `TransactionRequest` is passed to the `send_transaction(..)` function within the `Eth` namespace, which returns a `Future` that completes once the transaction has been broadcast to the network.  On completion, the `Promise` returns the transaction hash `Result`, which can then be unwrapped.

### Putting it all Together...

```rust
extern crate web3;

use web3::futures::Future;
use web3::types::{TransactionRequest, U256};

fn main() {
    let (_eloop, transport) = web3::transports::Http::new("http://localhost:8545").unwrap();

    let web3 = web3::Web3::new(transport);
    let accounts = web3.eth().accounts().wait().unwrap();

    let balance_before = web3.eth().balance(accounts[1], None).wait().unwrap();

    let tx = TransactionRequest {
        from: accounts[0],
        to: Some(accounts[1]),
        gas: None,
        gas_price: None,
        value: Some(U256::from(10000)),
        data: None,
        nonce: None,
        condition: None
    };

    let tx_hash = web3.eth().send_transaction(tx).wait().unwrap();

    let balance_after = web3.eth().balance(accounts[1], None).wait().unwrap();

    println!("TX Hash: {:?}", tx_hash);
    println!("Balance before: {}", balance_before);
    println!("Balance after: {}", balance_after);
}

```

Run this code, and you should see that the `accounts[1]` balance is 10000 wei greater after the transaction was sent...a successful ether transfer!

## Sending a Raw Transaction

Sending a raw transaction means signing a transaction with a private key on the Rust side, rather than on the node.  The node will then simply forward this transactions to the Ethereum network. 

There is a library called [ethereum-tx-sign](https://github.com/synlestidae/ethereum-tx-sign) that can help us with this off-chain signing, but it not particularly easy to use alongside `rust-web3` because of a lack of shared structs.  In this section of the guide I'll walk you through getting these libraries to play nicely together.

### Additional Libraries Used

As mentioned we will be using the `ethereum-tx-sign` library to sign transactions offline.  This library depends on the `ethereum-types` library when constructing a `RawTransaction`.  We will also be using the `hex` library to convert a hexidecimal private key into bytes.

Add these entries to your `cargo.toml` file:

```toml
ethereum-tx-sign = "0.0.2"
ethereum-types = "0.4"
hex = "0.3.1"
```

They can then be added to your crate:

```rust
extern crate ethereum_tx_sign;
extern crate ethereum_types;
extern crate hex;
```

### Signing the Transaction

The `ethereum_tx_sign` libraries contains a `RawTransaction` struct that can be used to easily sign an Ethereum transaction once initialised.  Its the initialisation thats the tricky part, as we need to convert between the `rust-web3` and `ethereum_types` structs.

Some conversion functions have been implemented to convert H160 (for Ethereum account addresses) and U256 (for the nonce value) structs from the `web3::types` returned by `rust-web3` functions to the`ethereum_types` expected by `ethereum-tx-sign`:

```rust
fn convert_u256(value: web3::types::U256) -> U256 {
    let web3::types::U256(ref arr) = value;
    let mut ret = [0; 4];
    ret[0] = arr[0];
    ret[1] = arr[1];
    U256(ret)
}

fn convert_account(value: web3::types::H160) -> H160 {
    let ret = H160::from(value.0);
    ret
}
``` 
We can now construct a `RawTransaction` object:

```rust
let nonce = web3.eth().transaction_count(accounts[0], None).wait().unwrap();

let tx = RawTransaction {
    nonce: convert_u256(nonce),
    to: Some(convert_account(accounts[1])),
    value: U256::from(10000),
    gas_price: U256::from(1000000000),
    gas: U256::from(21000),
    data: Vec::new()
};

```

Note that the nonce is not automatically calculated when constructing a `RawTransaction`.  We need to obtain the nonce for the sending account by calling the `transaction_count` function in the `Eth` namespace.  This value subsequently needs to be converted, to be in the format that `RawTransaction` expects.

Unlike in the `TransactionRequest` struct, we must also provide some sensible `gas` and `gas_price` values manually.

#### Obtaining a Private Key

Before signing, we need to have access to a private key that will be used to sign.  In this example we will hard code the private key of the first ETH populated account in `ganache` (remember to start with the `-d` argument).  This is ok for testing, but **you should never expose a private key in a production environment!**

```rust
fn get_private_key() -> H256 {
    let private_key = hex::decode(
        "4f3edf983ac636a65a842ce7c78d9aa706d3b113bce9c46f30d7d21715b23b1d").unwrap();

    return H256(to_array(private_key.as_slice()));
}

fn to_array(bytes: &[u8]) -> [u8; 32] {
    let mut array = [0; 32];
    let bytes = &bytes[..array.len()];
    array.copy_from_slice(bytes);
    array
}
```

The `hex:decode` function converts a hexidecimal string (make sure to remove the `0x` prefix) into a `Vec<u8>` but the `sign` function of `RawTransction` takes a private key in `ethereum_types::H256` format.  Unfortunately, the `H256` takes a `[u8; 32]` rather than a `Vec<T>` during construction so we need to do another conversion! 

The private key is passed to `to_array` as a slice, and this slice is then converted to a `[u8: 32]`.

#### Signing

Now that we have a function that returns a private key in the correct format, we can sign the transaction by simply calling:

```rust
let signed_tx = tx.sign(&get_private_key());
```

### Sending the Transaction

After signing, broadcasting the transaction to the Ethereum network is also a one-liner:

```rust
let tx_hash = web3.eth().send_raw_transaction(Bytes::from(signed_tx)).wait().unwrap()
```

Note, we have to perform another conversion here!  The `send_raw_transaction` takes a `Bytes` value as the argument, whereas the `sign` function of `RawTransaction` returns a `Vec<u8>`.  Luckily, this conversion is easy as the `Bytes` struct has a `From` trait out of the box to convert from a `Vec<u8>`.

Like the `send_transaction` equivalent, this function returns a `Future`, which in turn returns a `Result` object containing the transaction hash of the broadcast transaction on completion.

### Putting it all Together...

```rust
extern crate web3;
extern crate ethereum_tx_sign;
extern crate ethereum_types;
extern crate hex;

use web3::futures::Future;
use web3::types::Bytes;
use ethereum_tx_sign::RawTransaction;
use ethereum_types::{H160,H256,U256};

fn main() {
    let (_eloop, transport) = web3::transports::Http::new("http://localhost:8545").unwrap();

    let web3 = web3::Web3::new(transport);
    let accounts = web3.eth().accounts().wait().unwrap();

    let balance_before = web3.eth().balance(accounts[1], None).wait().unwrap();

    let nonce = web3.eth().transaction_count(accounts[0], None).wait().unwrap();

    let tx = RawTransaction {
        nonce: convert_u256(nonce),
        to: Some(convert_account(accounts[1])),
        value: U256::from(10000),
        gas_price: U256::from(1000000000),
        gas: U256::from(21000),
        data: Vec::new()
    };

    let signed_tx = tx.sign(&get_private_key());

    let tx_hash = web3.eth().send_raw_transaction(Bytes::from(signed_tx)).wait().unwrap();

    let balance_after = web3.eth().balance(accounts[1], None).wait().unwrap();

    println!("TX Hash: {:?}", tx_hash);
    println!("Balance before: {}", balance_before);
    println!("Balance after: {}", balance_after);
}

fn get_private_key() -> H256 {
    let private_key = hex::decode(
        "4f3edf983ac636a65a842ce7c78d9aa706d3b113bce9c46f30d7d21715b23b1d").unwrap();

    return H256(to_array(private_key.as_slice()));
}

fn convert_u256(value: web3::types::U256) -> U256 {
    let web3::types::U256(ref arr) = value;
    let mut ret = [0; 4];
    ret[0] = arr[0];
    ret[1] = arr[1];
    U256(ret)
}

fn convert_account(value: web3::types::H160) -> H160 {
    let ret = H160::from(value.0);
    ret
}

fn to_array(bytes: &[u8]) -> [u8; 32] {
    let mut array = [0; 32];
    let bytes = &bytes[..array.len()];
    array.copy_from_slice(bytes);
    array
}
```

## Summary
In this tutorial you have learnt how to send a basic Ether value transfer transaction from one account to another using Rust.  Two signing approaches were explained; signing on a node by an unlocked account, and signing a transaction on the Rust side.  

The full source code that has been covered in this guide is available on GitHub [here](TODO).

This is just scratching the surface of Ethereum transaction sending, and in a future tutorial I will walk you through sending transactions that manipulate data within an Ethereum smart contract.  Watch this space!