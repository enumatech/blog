---
layout: post
title:  "Rolling your own Proof-of-Authority Ethereum consortium"
date:   2017-08-29 17:45:00 +0800
categories: update
author: Adam Hose
---
How do you set up a private Ethereum chain and what considerations are there when doing so?
A private chain can potentially have huge advantages over public ones, transactions are cheaper, latencies are lower and the level of control you have is higher.
Why would you pick Proof-of-Authority over the other consensus methods out there?

<!--more-->

## Consensus comparison

# Why not Proof-of-Work
One of the largest flaws in the Ethereum protocol (and other Proof-of-Work based cryptocurrencies) is that any attacker with 51% or more of the mining resources (mining hashrate) will be in control of the network.
While the blockchain is immutable and recorded blocks cannot be changed this attacker could potentially do other things such as making sure no legitimate traffic comes through (a denial of service attack).
Considering anyone with the same genesis block could join your network this type of attack is very feasible and likely to hit you sooner or later.

There is also the issue of power usage (Bitcoin now uses the same amount of [electricity as a small country](https://digiconomist.net/bitcoin-energy-consumption)).
Ethereum has been working on alternatives to Proof-of-Work such as Proof-of-Stake which also solves the power issue but it's more aimed at public networks.

For a private network Proof-of-Work simply does not make sense.

# Why not Proof-of-Stake
Ethereum will soon transition to a [Proof-of-Stake consensus algorithm](https://github.com/ethereum/wiki/wiki/Proof-of-Stake-FAQ) which resolves some of the issues with Proof-of-Work for public networks.
Someone who wants to be a validator (miner) will deposit a number of ether into a smart contract and if a validator is deemed to be malicious the network simply locks this ether away.

Because there is no race to solve hard mathematical problems there is less incentive to issue new coins all the time, a validator will instead get their ether from transaction fees.
It does solve the excessive electricity usage problem and it does protect against 51% attacks to some degree.

For a private network Proof-of-Stake might be better than Proof-of-Work but it's still does not offer us the level of control and security required as anyone on the chain with some ether can become a validator if they just bond enough ether into the validator contract.

# Why Proof-of-Authority
Proof-of-Authority is a newer concept in the blockchain world where you have a number of pre-approved authority nodes (called sealers, think of these as mining nodes).
Any new node that you want to add has to be voted on by the currently approved set of authority nodes, this gives you full control over which nodes can seal blocks (mine) on your network.
To make sure a malicious signer cannot do too much harm to the network any signer can sign at most one of a number of consecutive blocks `(floor(SIGNER_COUNT / 2) + 1)`.
The same consensus is applied when an authority node is removed from the network.

The Ethereum Proof-of-Authority protocol is called Clique and is well described in the [Clique Github issue](https://github.com/ethereum/EIPs/issues/225).
Ethereum currently uses this algorithm for the [Rinkeby test network](https://rinkeby.io/).

Proof-of-Authority is a near perfect fit for private networks but not at all suited for public networks where the trust should be as distributed as possible.

## How to set up a PoA geth network
Now that we know the why we will go on to the how, it's not all that dissimilar from setting up a Proof-of-Work chain.

# Create a new account
We will create new accounts for our new network, an account is not tied to any specific chain. It's just a private/public key.

Let's start by creating an account used for sealing blocks

`geth account new --datadir ~/.ethereum/privateconsortium`

And then a separate one used to deploy our example contract

`geth account new --datadir ~/.ethereum/privateconsortium`

# Create the genesis block
Creating a PoA genesis block is easiest using the puppeth tool that comes with the go-ethereum package.

Run `puppeth` and follow the wizard

1. Specify a network name
   For this tutorial we will use the name "PrivateConsortium"

2. Select "Configure new genesis"

    1. Select the Clique consensus engine

    2. Choose a block time

       You may choose a lower block time to get shorter wait time for new blocks

       15 seconds is the default to match the Ethereum main network but we will go with 5 for our private network.

       Beware that this is going to make the chain grow faster. What tradeoff you want to make is ultimately up to you.

    3. Add a sealing account
       We will now enter the address we got from the *first* `geth account new` command.

    4. Add any pre-funded accocunts.
       We will now enter the address we got from the *second* `geth account new` command.

    5. Specify network id
       This is just a numerical id for your network that will be encoded in the genesis block, a random one is fine.

    6. Embed a funny message in your genesis
       "Hallo welt" or something like this

    7. Save the genesis JSON

# Starting the network

In this case `0x5d5a2e1ff1cc3bea78bfc6fea9c38d1fe6e6698e` is the address returned by the *first* `geth account new` command.
1. Initialize the new chain with our custom genesis block

`geth --nodiscover --datadir ~/.ethereum/privateconsortium init PrivateConsortium.json`
2. Run our new chain

`geth --nodiscover --datadir ~/.ethereum/privateconsortium --unlock 0x5d5a2e1ff1cc3bea78bfc6fea9c38d1fe6e6698e --mine --rpcaddr 127.0.0.1 --rpcapi eth,net,web3,personal`

## Automating it with Nixops

# What is Nixos
[Nixos](http://nixos.org/) is a declarative purely functional Linux distribution that makes systems reliably upgradeable and reproducible.
It calculates the entire system state from a single declarative configuration file and applies it atomically.
It saves every generation on-disk and it's always possible to roll back to a previous system state.

While Nixos can take a while to learn as it's quite a departure from how classical unix-like systems work it makes us able to deploy new changes quickly and with confidence, and we are in full control of the whole dependency graph.

# What is Nixops
[Nixops](https://nixos.org/nixops/manual/) is a devops tool to automate Nixos deployments, much like Ansible, Puppet, Chef, et al.
It does fully automated deployments including provisioning, automatic cross-cloud tunneling etc etc.

We are going to use this to deploy a whole cluster including AWS security groups from our configuration.

# Creating deployment.nix
```nix
let
  dbDir = "/var/db/geth/";
  networkId = "888";
  genesis = ''{
    "config": {
      "chainId": 62350,
      "homesteadBlock": 1,
      "eip150Block": 2,
      "eip150Hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
      "eip155Block": 3,
      "eip158Block": 3,
      "clique": {
        "period": 5,
        "epoch": 30000
      }
    },
    "nonce": "0x0",
    "timestamp": "0x59a4e76d",
    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "extraData": "0x48616c6c6f2077656c74000000000000000000000000000000000000000000005d5a2e1ff1cc3bea78bfc6fea9c38d1fe6e6698e0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
    "gasLimit": "0x47b760",
    "difficulty": "0x1",
    "mixHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "coinbase": "0x0000000000000000000000000000000000000000",
    "alloc": {
     "5d5a2e1ff1cc3bea78bfc6fea9c38d1fe6e6698e": {
       "balance": "0x200000000000000000000000000000000000000000000000000000000000000"
      }
    }
  }'';

in {

  network.description = "Enuma tutorial ethereum network";

  resources.ec2SecurityGroups.us-west-1-ssh-allow = {
    region = "us-west-1";
    description = "Allow SSH from all hosts";
    rules = [ {
      fromPort = 22;
      toPort = 22;
      sourceIp = "0.0.0.0/0";
    } ];
  };

  resources.ec2SecurityGroups.us-west-1-geth-node = {
    region = "us-west-1";
    description = "Allow ethereum communication";
    rules = [
      {
        fromPort = 30303;
        toPort = 30303;
        protocol = "tcp";
        sourceIp = "0.0.0.0/0";
      }
      {
        fromPort = 30303;
        toPort = 30303;
        protocol = "udp";
        sourceIp = "0.0.0.0/0";
      }
    ];
  };

  geth_1 =
    { resources, ... }:
    {
      environment.etc."enuma_geth/password".text = "YOUR_SECRET_ACCOUNT_PASSWORD";
      environment.etc."enuma_geth/environmentfile".text = ''
      GETH_ACCOUNT=0x3fe7039b5b0be7973d77ff2426704348aa9a3849
      GETH_BOOTNODES=${bootNodes}
      '';

      deployment.ec2.keyPair = "adam@enuma";
      deployment.ec2.accessKeyId = "AKIAXSDHLRGXYCNSYUHK";
      deployment.ec2.instanceType = "m3.medium";
      deployment.ec2.region = "us-west-1";
      deployment.ec2.ebsInitialRootDiskSize = 15;
      deployment.ec2.securityGroups = [
        resources.ec2SecurityGroups.us-west-1-ssh-allow
        resources.ec2SecurityGroups.us-west-1-geth-node
      ];

      # Set the unstable channel
      nix.nixPath = [ "nixpkgs=https://nixos.org/channels/nixos-unstable-small/nixexprs.tar.xz" ];

      # General system security configuration
      services.openssh.passwordAuthentication = false;
      security.lockKernelModules = true;
      security.apparmor.enable = true;

      # Firewall rules
      networking.firewall.enable = true;
      networking.firewall.allowedTCPPorts = [
        30303 # Geth TCP
        22    # OpenSSH
      ];
      networking.firewall.allowedUDPPorts = [ 30303 ];  # Geth UDP

      # Geth configuration
      users.extraUsers.geth = {
        description = "Geth daemon user";
        home = dbDir;
        createHome = true;
      };

      systemd.services.geth = {
        description = "Go ethereum daemon";
        serviceConfig.Type = "simple";
        serviceConfig.Restart = "on-failure";
        serviceConfig.PrivateTmp = true;
        serviceConfig.PrivateDevices = true;
        serviceConfig.ProtectHome = true;
        serviceConfig.WorkingDirectory = dbDir;
        serviceConfig.User = "geth";
        wantedBy = [ "multi-user.target" ];
        after = [ "network.target" ];
      };

      environment.etc."enuma_geth/genesis.json".text = genesis;

      # Initialize with genesis if not already initialized
      systemd.services.geth.serviceConfig.ExecStartPre = "${pkgs.bash}/bin/bash -c \'(test ! -d ${dbDir}/.ethereum/geth && ${pkgs.go-ethereum}/bin/geth init /etc/enuma_geth/genesis.json) || true\'";
      systemd.services.geth.serviceConfig.ExecStart = "${pkgs.go-ethereum}/bin/geth --bootnodes \${GETH_BOOTNODES} --unlock \${GETH_ACCOUNT} --password /etc/enuma_geth/password --mine --syncmode full --lightserv 80 --networkid 888 --rpcaddr 127.0.0.1 --rpcapi eth,net,web3,personal,clique";

      systemd.services.geth.serviceConfig.EnvironmentFile = "/etc/enuma_geth/environmentfile";
    };
}
```

A real world deployment should have a minimum of 3 nodes and be more modularised but for the sake of a simpler tutorial we settle on one.

# Creating deployment state
`nixops create ./deployment.nix --deployment enuma_tutorial`

# Running the deployment
`nixops deploy --deployment enuma_tutorial`
Sit back, relax and watch the security groups + instance getting created.

# Copy the private key files to the server
Copy the private key from the *first* `geth account new` command to the server into /var/db/geth/.ethereum/keystore/

# Adding remote nodes to your local chain
The easiest way to add remote nodes to your local machine is to use the static nodes feature of geth.
Create a file in ~/.ethereum/privateconsortium/static-nodes.json with the following contents:

Check the output from the `geth` process. It will create a node public key on initial use.
Look for a line that starts with `INFO [08-29|20:49:32] RLPx listener up`. This should contain your public key.

```json
[
    "enode://<your node public key>@<your node public ip>:30303"
]
```

You can now use your local node with the account created previously in the *second* `geth account new` step.

## Final words
Proof-of-Authority provides many advantages for private chains: It provides more configurability in terms of block times and latency, it's not computationally expensive and it's more secure because of how authority nodes are added.

While it can take a little while to wrap your head around and get acquainted with it's well worth it if you are interested in running a private consortium or just want a local network for testing without burning CPU.

# Author
[Adam Hose](http://github.com/adisbladis/)
