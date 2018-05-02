# VIPSTARCOINJS Wallet

This is a client-side wallet library that can generate private keys from a mnemonic, or import private keys from other VIPSTARCOIN wallets.

It can sign transactions locally, and submit the raw transaction data to a remote VIPSTARCOIN node. The blockchain data is provided by the Insight API, rather than the raw VIPSTARCOINd RPC calls.

This library makes it possible to run DApp without the users having to run a full VIPSTARCOINd node.

## Install

```
yarn add vipstarcoinjs-wallet
```

## Implementation Notes

There are some differences from the original web wallet repo.

* Removed VUE specific code.
* Removed reactive data setters that are intended to trigger view updates, to make this a plain-old JavaScript module.
* Each wallet instance is instantiated with a network explicitly. This allows simultaneous use of different networks.
* TypeScript for type hinting.
* Uses satoshi (1e8) as internal units
  * Can represent up to ~90 million VIPSTARCOIN accurately.
* Uses [coinselect](https://github.com/bitcoinjs/coinselect) to select utxos.
  * Taking into account the size of a transaction, and multiplies that by fee rate per byte.
  * Uses blackjack algorithm, and fallbacks to simple accumulative.
* Set tx relay fee automatically from fee rate reported by the network.
* send-to-contract transaction can transfer value to the contract

# API

+ [Networks](#networks)
  + [fromWIF](#fromwif)
  + [fromMnemonic](#frommnemonic)
+ [Wallet](#wallet)
  + [async wallet.getInfo](#async-walletgetinfo)
  + [async wallet.send](#async-walletsend)
  + [async wallet.generateTx](#async-walletgeneratetx)
  + [async wallet.contractSend](#async-walletcontractsend)
  + [async wallet.generateContractSendTx](#async-walletgeneratecontractsendtx)
  + [async wallet.contractCall](#async-walletcontractcall)


# Examples

## Create Mnemonic+Password Wallet

```js
import { networks, generateMnemonic } from "vipstarcoinjs-wallet"

async function main() {
  const network = networks.testnet
  const mnemonic = generateMnemonic()
  const password = "covfefe"

  const wallet = network.fromMnemonic(mnemonic, password)

  console.log("mnemonic:", mnemonic)
  console.log("public address:", wallet.address)
  console.log("private key (WIF):", wallet.toWIF())
}

main().catch((err) => console.log(err))
```

Example Output:

```
mnemonic: enforce scan turkey forget foam lab inmate edit skate tray diary stem
public address: VS7vudWLtRf4P9ecDS5DfpZyB8q8gF5R6J
private key (WIF): Kx5rzxPEXBBHL2KBd1mUGY494fMUmzAdMsNjkPNDeBCMszijcuug
```

## Send Fund

This example restores a wallet from a private key (in [WIF](https://en.bitcoin.it/wiki/Wallet_import_format) format), then sending value to another address.

The transaction is signed locally, and the transaction submitted to a remote API.

The currency unit used is `satoshi`. To convert VIPSTARCOIN to satoshi you should multiply the amount you want with `1e8`.

```js
import { networks } from "vipstarcoinjs-wallet"

async function main() {
  // Use the test network. Or `networks.mainnet`
  const network = networks.testnet

  const wif = "Kx5rzxPEXBBHL2KBd1mUGY494fMUmzAdMsNjkPNDeBCMszijcuug"
  const wallet = network.fromWIF(wif)

  console.log(wallet.address)

  const toAddr = "VRNgVKgdgKY8kxgaL1pf6vB8e4VgqYa5GU"
  // Sending 0.1 VIPS
  const sendtx = await wallet.send(toAddr, 1, 0.1 * 1e8)
  console.log("sendtx", sendtx)
}

main().catch((err) => console.log(err))
```

## Send To Contract

Let's burn some money using the `Burn` contract:

```solidity
pragma solidity ^0.4.18;

contract Burn {
  uint256 public totalburned;
  event DidBurn(address burnerAddress, uint256 burnedAmount);

  function burnbabyburn() public payable {
    totalburned = msg.value;
    DidBurn(msg.sender, msg.value);
  }
}
```

The ABI encoding for the `burnbabyburn()` invokation is `e179b912`. We'll burn 0.05 VIPS, expressed in unit of satoshi.

```ts
import { networks } from "vipstarcoinjs-wallet"

async function main() {
  const network = networks.testnet

  const privateKey = "Kx5rzxPEXBBHL2KBd1mUGY494fMUmzAdMsNjkPNDeBCMszijcuug"

  const wallet = network.fromWIF(privateKey)


  const contractAddress = "b10071ee33512ce8a0c06ecbc14a5f585a27a3e2"
  const encodedData = "e179b912" // burnbabyburn()

  const tx = await wallet.contractSend(contractAddress, encodedData, {
    amount: 0.05 * 1e8, // 0.05 VIPS in satoshi
  })

  console.log(tx)
}

main().catch((err) => console.log(err))
```

# Networks

Two networks are predefined:

```js
import { networks } from "vipstarcoinjs-wallet"

// Main Network
networks.mainnet

// Test Network
networks.testnet
```

## fromPrivateKey

Alias for `fromWIF`.

## fromWIF

`fromWIF` constructs a wallet from private key (in [WIF](https://en.bitcoin.it/wiki/Wallet_import_format) format).

Suppose you want to import the public address `VS7vudWLtRf4P9ecDS5DfpZyB8q8gF5R6J`. Use `VIPSTARCOIN-cli` to dump the private key from wallet:

```
VIPSTARCOIN-cli dumpprivkey VS7vudWLtRf4P9ecDS5DfpZyB8q8gF5R6J

Kx5rzxPEXBBHL2KBd1mUGY494fMUmzAdMsNjkPNDeBCMszijcuug
```

```js
const network = networks.testnet

const privateKey = "Kx5rzxPEXBBHL2KBd1mUGY494fMUmzAdMsNjkPNDeBCMszijcuug"

const wallet = network.fromWIF(privateKey)
console.log("public address:", wallet.address)
```

Output:

```
public address: VS7vudWLtRf4P9ecDS5DfpZyB8q8gF5R6J
```

## fromMnemonic

`fromMnemonic` constructs a wallet from mnemonic. User can optionally specify a `password` to add to the mnemonic entropy.

```ts
const network = networks.testnet
const mnemonic = "hold struggle ready lonely august napkin enforce retire pipe where avoid drip"
const password = "covfefe"

const wallet = network.fromMnemonic(mnemonic, password)

console.log("public address:", wallet.address)
console.log("private key (WIF):", wallet.toWIF())
```

Example Output:

```
public address: VDa8YRLs9wo25cy5bigsyk1f69N4a8bxut
private key (WIF): Kx3L9hYgyuaFypg9aRD5wLR674gGw7itxS7e8W25KVEDhLSG7koz
```

# Wallet

Wallet manages blockchain access for an address. It is able to create and sign transactions locally for sending a payment or interacting with a smart contract.

You would typically construct a Wallet instance using the factory methods provided by `Network`.

## async wallet.getInfo

Get basic information about the wallet address.

Example:

```ts
const info = await wallet.getInfo()
console.log(info)
```

Output:

```
{ addrStr: 'VDa8YRLs9wo25cy5bigsyk1f69N4a8bxut',
  balance: 881.77926876,
  balanceSat: 88177926876,
  totalReceived: 2041.03887636,
  totalReceivedSat: 204103887636,
  totalSent: 1159.2596076,
  totalSentSat: 115925960760,
  unconfirmedBalance: 0,
  unconfirmedBalanceSat: 0,
  unconfirmedTxApperances: 0,
  txApperances: 14,
  transactions:
   [ '6dcfed9716f162b61bd60bf9a1a0b67226dbbdb20678d2a28fe3d30ba383f96e',
     '20c6afe3312ca99e17b40e841bdec64ee4cf1393cfd43df6bb75787ef6ac3720',
     '8e4e76d11d1214ee54d651b2ea5f84db4e71702b57df7bd3af4829710be565fa',
     '408ad3cde1975e387f2740d388cdd668b2e862ef472bf4ef6bfc4c8cf055e946',
     '66862bcba133c9313eb1de407037afbbc3ef572929b66e2025d60408a19f4e34',
     'cdb02d5ca36c52c767b58c470b72a59c3565644cc2e3715f85ded2df15f6cc50',
     '261520a53619b0ea2fed4e94942cb404c12540d0026b2b2a086f1d92df0107d1',
     'a1e1664afc5566f302fcd569ff82d2fde29e0062ac0b18dc3ee0c00ff38f5296',
     '48d72bc841062491553bf4fc691e7da38f279478bb2206da2ee26cf703a923eb',
     'cb44aad24f6830fbbf117824ec2aa62f841880f5408bd8afbc8bbab33fa55f11',
     '19518f35bb14cf7f783dffe4714a6e0025e5b2e50ce9d9bb40006bb880cea0b5',
     '4acb02d22856f5ce8a562dfef22b0a464f69a8b9e306a25ef742833e928dd31a',
     '1152960117d4f7ab9813b6be3f399bbba388d957c01e5c392dbf5d3579131aa8',
     'e5745de218d9625e8374e15edcfe1af5c077d01c163951eb26439306313b6628' ] }
```

## async wallet.send

Send payment to a receiving address. The transaction is signed locally using the
wallet's private key, and the raw transaction submitted to a remote API (without
revealing the wallet's secret).

Method signature:

```ts
/**
 * @param to The receiving address
 * @param amount The amount to transfer (in satoshi)
 * @return The raw transaction as hexadecimal string
 *
 */
public async send(
  to: string,
  amount: number,
  opts: ISendTxOptions = {},
): Promise<Insight.ISendRawTxResult>
```

Example:

```ts
const toAddress = "VRNgVKgdgKY8kxgaL1pf6vB8e4VgqYa5GU"
const amount = 0.15 * 1e8 // 0.15 VIPS

const tx = await wallet.send(toAddress, amount)
console.log(tx)
```

Output:

```
{ txid: '40fec162e0d4e1377b5e6744eeba562408e22f60399be41e7ba24e1af37f773c' }
```

#### async wallet.send options

```ts
export interface ISendTxOptions {
  /**
   * Fee rate to pay for the raw transaction data (satoshi per byte). The
   * default value is the query result of current network's fee rate.
   */
  feeRate?: number
}
```

Setting tx fee rate manually:

```ts
const tx = await wallet.send(toAddress, amount, {
  // rate is 400 satoshi per byte, or  ~0.004 VIPS/KB, as is typical.
  feeRate: 400,
})
```

## async wallet.generateTx

Generate and sign a payment transaction.

Method signature:

```ts
/**
 * @param to The receiving address
 * @param amount The amount to transfer (in satoshi)
 * @param opts
 *
 * @returns The raw transaction as hexadecimal string
 */
public async generateTx(
  to: string,
  amount: number,
  opts: ISendTxOptions = {},
): Promise<string>
```

Example:

```ts
const toAddress = "VRNgVKgdgKY8kxgaL1pf6vB8e4VgqYa5GU"
const amount = 0.15 * 1e8

const rawtx = await wallet.generateTx(toAddress, amount)
console.log(rawtx)
```

Example output, the raw transaction as hexadecimal string:

```
01000000016ef983a30bd3e38fa2d27806b2bddb2672b6a0a1f90bd61bb662f11697edcf6d010000006b483045022100bd144e7e2038fb5894cfa38f479399f5159151b9a9b8ee1b0ed04bcdd6ae517f02201ead32040ed4a4f5ab9fddd53c61ba3e209ae36ea0aecb52814627e931f09f0301210326ede40223dec4e07135859f55dd287f5f6eb2fb03ac87e9cc45e67bac7aacc7ffffffff02c0e1e400000000001976a914a168fd73adc40e18f4647b6408f599adba593d2188ac99524328120000001976a9148f2c219e6e626e0132a08eae64ea8b3b8960f1d288ac00000000
```

You can decode the raw transaction using `VIPSTARCOIN-cli`:

```
VIPSTARCOIN-cli decoderawtransaction 01000000016ef983a30bd3...

{
  // ...
  "vout": [
    {
      "value": 0.15000000,
      "n": 0,
      "scriptPubKey": {
        "asm": "OP_DUP OP_HASH160 a168fd73adc40e18f4647b6408f599adba593d21 OP_EQUALVERIFY OP_CHECKSIG",
        "hex": "76a914a168fd73adc40e18f4647b6408f599adba593d2188ac",
        "reqSigs": 1,
        "type": "pubkeyhash",
        "addresses": [
          "VRNgVKgdgKY8kxgaL1pf6vB8e4VgqYa5GU"
        ]
      }
    },
    {
      "value": 779.84912025,
      "n": 1,
      "scriptPubKey": {
        "asm": "OP_DUP OP_HASH160 8f2c219e6e626e0132a08eae64ea8b3b8960f1d2 OP_EQUALVERIFY OP_CHECKSIG",
        "hex": "76a9148f2c219e6e626e0132a08eae64ea8b3b8960f1d288ac",
        "reqSigs": 1,
        "type": "pubkeyhash",
        "addresses": [
          "VPiFRLCUcNA9V5rAKjwqchqRzvn3s69RVv"
        ]
      }
    }
  ]
}
```

There are two vouts:

1. pubkeyhash 0.15. This is the amount we want to send.
2. pubkeyhash 80.69822025. This is the amount we going back to the original owner as change.

## async wallet.contractSend

Create a send-to-contract transaction that invokes a contract's method.

```ts
/**
  * @param contractAddress Address of the contract in hexadecimal
  * @param encodedData The ABI encoded method call, and parameter values.
  * @param opts
  */
public async contractSend(
  contractAddress: string,
  encodedData: string,
  opts: IContractSendTXOptions = {},
): Promise<Insight.ISendRawTxResult>
```

Example:

Invoke the `burn()` method, and transfer 5000000 satoshi to the contract.

* The `burn()` method call ABI encodes to `e179b912`
* The 5000000 is `msg.value` in contract code.


```ts
const contractAddress = "1620cd3c24b29d424932ec30c5925f8c0a00941c"
// ABI encoded data for the send-to-method transaction
const encodedData = "e179b912"

// Invoke a contract's method, and transferring 0.05 to it.
const tx = await wallet.contractSend(contractAddress, encodedData, {
  amount: 0.05 * 1e8,
})

console.log(tx)
```

Output:

```
{ txid: 'd12ff9cfd76836d8eb5a39bc40f1dc5e6e2032bfa132f66cca638a7e76f2b6e7' }
```

## async wallet.generateContractSendTx

Generate a raw a send-to-contract transaction that invokes a contract's method.

Method signature:

```ts
/**
  * @param contractAddress
  * @param encodedData
  * @param opts
  */
public async generateContractSendTx(
  contractAddress: string,
  encodedData: string,
  opts: IContractSendTXOptions = {},
): Promise<string>
```

Example:

```ts
const contractAddress = "1620cd3c24b29d424932ec30c5925f8c0a00941c"
const encodedData = "e179b912"

const rawtx = await wallet.generateContractSendTx(contractAddress, encodedData, {
  amount: 0.01 * 1e8,
})

console.log(rawtx)
```

Example output:

```
0100000001e7b6f2767e8a63ca6cf632a1bf32206e5edcf140bc395aebd83668d7cff92fd1010000006b483045022100b86c4cbb2aecab44c951f99c0cbbf6115cf80881b39f33b4efd4d296892c1c15022062db1f681e684616e55303556577c9242102ff7a6815894dfb3090a7928fa13a012103c12c73abaccf35b40454e7eb0c4b5760ce7a720d0cd2c9fb7f5423168aaeea03ffffffff0240420f000000000022540390d003012804e179b912141620cd3c24b29d424932ec30c5925f8c0a00941cc2880256e0010000001976a914c78300c58ab7c73e1767e3d550464d591ab0a12888ac00000000
```

Decode the raw transaction:

```
VIPSTARCOIN-cli decoderawtransaction 0100000001e7b6f2767e8a6...
```

Decoded Raw TX:

```ts
{
  // ...
  "vout": [
    {
      "value": 0.01000000,
      "n": 0,
      "scriptPubKey": {
        "asm": "4 250000 40 314145249 1620cd3c24b29d424932ec30c5925f8c0a00941c OP_CALL",
        "hex": "540390d003012804e179b912141620cd3c24b29d424932ec30c5925f8c0a00941cc2",
        "type": "call"
      }
    },
    {
      "value": 779.88895212,
      "n": 1,
      "scriptPubKey": {
        "asm": "OP_DUP OP_HASH160 8f2c219e6e626e0132a08eae64ea8b3b8960f1d2 OP_EQUALVERIFY OP_CHECKSIG",
        "hex": "76a9148f2c219e6e626e0132a08eae64ea8b3b8960f1d288ac",
        "reqSigs": 1,
        "type": "pubkeyhash",
        "addresses": [
          "VPiFRLCUcNA9V5rAKjwqchqRzvn3s69RVv"
        ]
      }
    }
  ]
}
```

There are two vouts:

1. call 0.11. This is the amount we want to send to the contract.
2. pubkeyhash 80.58700424. This is the amount we going back to the original owner as change.

## async wallet.contractCall

Query a contract's method. It returns the result and logs of a simulated execution of the contract's code.

Method signature:

```ts
/**
 * @param contractAddress Address of the contract in hexadecimal
 * @param encodedData The ABI encoded method call, and parameter values.
 * @param opts
 */
public async contractCall(
  contractAddress: string,
  encodedData: string,
  opts: IContractSendTXOptions = {},
): Promise<Insight.IContractCall>
```

Example:

```ts
const contractAddress = "b10071ee33512ce8a0c06ecbc14a5f585a27a3e2"
const encodedData = "e179b912"

const result = await wallet.contractCall(contractAddress, encodedData, {
  amount: 0.01 * 1e8,
})

console.log(JSON.stringify(result, null, 2))
```

Output:

```ts
{
  "address": "b10071ee33512ce8a0c06ecbc14a5f585a27a3e2",
  "executionResult": {
    "gasUsed": 27754,
    "excepted": "None",
    "newAddress": "b10071ee33512ce8a0c06ecbc14a5f585a27a3e2",
    "output": "",
    "codeDeposit": 0,
    "gasRefunded": 0,
    "depositSize": 0,
    "gasForDeposit": 0
  },
  "transactionReceipt": {
    "stateRoot": "c04b98dbd1a38be8ecfb71e40072c90a1ee9f5961bb80fa6262f8a32979427bb",
    "gasUsed": 27754,
    "bloom": "000000000000000000002000400000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000
00000000000000000000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000000000000000000000000",
    "log": [
      {
        "address": "b10071ee33512ce8a0c06ecbc14a5f585a27a3e2",
        "topics": [
          "9c31339612219a954bda4c790e4b182b6499bdf1464c392cb50e61d8afa1f9f2"
        ],
        "data": "000000000000000000000000ffffffffffffffffffffffffffffffffffffffff0000000000000000000000000000000000000000000000000000000000000000"
      }
    ]
  }
}
```
