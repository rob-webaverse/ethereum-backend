# Ethereum Backend - Quick Start Guide

## Introduction

The Webaverse Ethereum backend consists of a side chain that we mine using Proof-of-Stake.

To start a mining node, you must have an authorized miner address with a certificate installed in the  `geth`  data directory -- ask [Avaer](https://github.com/avaer) for the keys.

To validate/replicate/sync you don't need any keys.

---

## Quick Install

Once you've installed all basic building blocks, you're just a few steps away from starting to develop your application. To clone and run this repository execute these command using a command line:


```bash

# Clone this repository

git clone https://github.com/webaverse/ethereum-backend.git

# Go into the repository

cd ethereum-backend/

```
To install the dependencies, run this in the application folder from the command-line:
```bash

# Install dependencies

$ npm install

```
---

## Running Your Application


Run your application using npm:

```bash

# Run the app (in background)

$ npm start

```
This command will run your application in background using [forever](https://www.npmjs.com/package/forever)

To view the status of the running app run:

```bash

# View running forever tasks

$ sudo forever list

```

>You can stop this app by running this command:
```bash

# Stop the app running in background

$ npm stop

```

### Doesn't Re-compile automatically

The application won't hot reload itself automatically if changes are made to any files while it's running. You have to re-run the application to reflect those changes.

```bash

# First stop the application

$ npm stop

# Then run it again

$ npm start

```
---

## Development Environment Setup

  
> :memo: We prefer using [VSCode](https://code.visualstudio.com/download) for development, so the below notes reflect that toolset; however you should be able to adapt this guide to apply to any other IDEs.
  
### Directory Structure

```bash

**Root**

├───	index.js <--- Main Application Logic Resides Here

```

---

### Setup ESLint


* Within VSCode, go to your extensions tab and search for `ESLINT`

![VSCode ESLint Setup Example](/img/VSCodeESLintSetup.png?raw=true)

OR From the command line:

```bash
npm install eslint -g
eslint --init
```

---

### Setup Cutom Host

Please follow this [tutorial](setup-custom-host.md) to setup custom host.

---

## Commands

**To bootstrap a mainnet validation node:**

```bash

geth --datadir mainnet init genesis-mainnet.json

cp ./static-nodes-mainnet.json ./mainnet/static-nodes.json

cp account-mainnet.json ./mainnet/keystore/UTC--2021-02-17T10-32-36.272770958Z--bb0827ee9b0b459e1b5dd6dbea0f55bf578dbbd2

geth --datadir mainnet --http --http.addr 172.31.2.5 --http.corsdomain '*' --syncmode full --networkid 1338

```

`static-nodes-mainnet.json` has some bootstrap nodes listed so you should be able to start syncing from those. Your chain will be "reorganized" a lot while you sync up, which is normal.

---

## Blockchains

There are currently 4 chains that we use:

- `mainnet` (ETH mainnet)
- `mainnetsidechain` (our Geth nodes)
- `rinkeby` (ETH Rinkeby)
- `rinkebysidechain` (our Geth nodes)

`mainnetsidechain`: http://ethereum1.exokit.org:8545 chainId 1338 

`rinkebysidechain`: http://ethereum1.exokit.org:8546 chainId 1337

You can use these details with MetaMask to interat directly with the chains. 
> :grey_exclamation:There are no gas fees.

These networks also have HTTPS proxy support for secure frontend development:

https://mainnetsidechain.exokit.org
https://rinkebysidechain.exokit.org

> :memo: Note that the port on these is the standard HTTPS port, `443`.

## Contracts

The contracts we deploy onto all chains are available at https://github.com/webaverse/contracts.

---

## Atomic saves

Replication is accomplished by having multiple nodes mine on that address at the same time.

> :warning: `geth` does _not_ stream blocks to disk eagerly. A system crash will lose blocks on that node, though other miners will not be affected.

### Restarting geth Servers

It is important that any restart of these nodes follows the correct order:

```
for (i in [2, 3, 1]) { // order matters
1. shut down node i with SIGTERM
2. make sure node i saved its state in the logs
3. start node i again
4. ensure node i is replicating and synced and do not proceed unless it is
```
---

## How Transfers Work

### Chains

There are two parallel blockchains for each Ethereum source of truth. There are two sources of truth (mainnet and rinkeby) and they do not interact. Therefore (2 chains x 2 sources) there are 4 chains.

```
mainnet
mainnetsidechain
rinkeby
rinkebysidechain
```

---

### Common Case

The common case is `mainnet` (ETH) and `mainnetsidechain` (our `geth`).

These talk to each other via a signature scheme enforced in the contracts. Basically, each contract is deployed twice, once on each chain. We mint on the side chain usually (enforced by constructor arguments). Transfers occur via assignment of the token away from the user to the contract's address, logging a deposit event on the sidechain. The client then asks the signing server to read the sidechain and sign off on the fact that this deposit occurred. If successful the signature is sent back to the client. The client then takes that signature and submits it in a mainnet transaction. This should be confirmed by the user in Metamask.

#### User Accepts

If the user accepts, the mainnet should accept the signature and assign ownership of the token to the user on mainnet.

---

#### User Rejects

If the user does not accept then the token is stuck in between. The way to fix this is to continue the transfer from the point where you asked the signing server for the signature.

---

### Reversal

The way back from mainnet to mainnetsidechain is the same procedure, except with contracts switched. You would first write to the mainnet to move the mainnet token to the mainnet contract (requires user confirmation). Once this succeeds you can ask the signing server for the signature (uses a different endpoint than last time). Then that signature can be written to the sidechain to have the contract return the token that you initially deposited. At this point you are back where you started and the procedure can be repeated.

---