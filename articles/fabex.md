---
title: 'FabEx tutorial: an introduction to the right Hyperledger Fabric explorer'
description: >-
  Here we will learn the basics of using FabEx — blockchain explorer for
  Hyperledger Fabric written purely in Go with a focus on simplicity.
date: '2021-01-30T10:28:02.476Z'
categories: []
keywords: []
slug: >-
  /@vadiminshakov/fabex-tutorial-an-introduction-to-the-right-hyperledger-fabric-explorer-cd9ee1848cd9
---

![](/Users/vadiminshakov/Downloads/medium-export-089ba9aeef850c38170bac7334f267b895b384bab6263df16f3c4e1dbe450357/posts/md_1705862990838/img/1__ACvm0vLGezyQqryEA__u1fg.png)

Here we will learn the basics of using FabEx — blockchain explorer for Hyperledger Fabric written purely in Go with a focus on simplicity.

Let’s start an example HLF network. First make sure you have the required Hyperledger Fabric binaries and Docker images. If not, follow the instructions [on this page](https://hyperledger-fabric.readthedocs.io/en/release-2.2/install.html). And then start network:

make fabric-test

Start CassandraDB (you can choose MongoDB instead, but here we will only consider working with СassandraDB):

```
make cassandra
```

Now we can start FabEx. For this we need a configuration file. Take a look at file tests/config.yaml. As you can see, it points to ./tests/connection.yaml:

connectionProfile: ./tests/connection.yaml

It’s just a HLF connection profile prepared by me in advance. When you work with your own HLF network, just specify the path to your connection profile.

Now add aliases to _/etc/hosts:_

sudo bash -c 'echo "127.0.0.1 org1.example.com peer0.org1.example.com peer1.org1.example.com orderer.org1.example.com ca.org1.example.com" >> /etc/hosts'

Then untar vendored dependencies_:_

tar xvf vendor.tar

Run FabEx_:_

go run fabex.go -task=grpc -configpath=tests -configname=config -enrolluser=true -db=cassandra

![](/Users/vadiminshakov/Downloads/medium-export-089ba9aeef850c38170bac7334f267b895b384bab6263df16f3c4e1dbe450357/posts/md_1705862990838/img/1__zFUNO2bsZh6B909X7jyzVQ.png)

It will work with test Fabric network specified in tests/config.yaml.

Look at **_\-task=grpc_** parameter. This parameter means that FabEx will work as a service with gRPC interface. It will parse blockchain, specified in connection profile, save data to database and respond to RPC queries. But FabEx can work without gRPC interface exposed. It can work in CLI mode. You can instruct FabEx to just parse the blockchain, add data to the database and do nothing else:

go run fabex.go -task=explore -configpath=tests -configname=config -enrolluser=true -db=cassandra

Then you can execute CLI commands in another terminal.

For example, you can query block with specific block number:

go run fabex.go -task=getblock -blocknum=1 -configpath=tests -configname=config -enrolluser=true -db=cassandra

![](/Users/vadiminshakov/Downloads/medium-export-089ba9aeef850c38170bac7334f267b895b384bab6263df16f3c4e1dbe450357/posts/md_1705862990838/img/1__L2SQ9hQMe59evoAe9zOwuA.png)

Get all saved data:

go run fabex.go -task=getall -configpath=tests -configname=config -enrolluser=true -db=cassandra

Get channel info:

go run fabex.go -task=channelinfo -configpath=tests -configname=config -enrolluser=true -db=cassandra

Get channel config:

go run fabex.go -task=channelconfig -configpath=tests -configname=config -enrolluser=true -db=cassandra

Ok, back to our FabEx in gRPC mode. Now FabEx parses blockchain and saves all collected data to the CassandraDB. Let’s check it in another terminal:

docker exec -it cassandra bash  
cqlsh  
use blocks;  
select blocknum, channelid, txid, payloadkeys from txs;

![](/Users/vadiminshakov/Downloads/medium-export-089ba9aeef850c38170bac7334f267b895b384bab6263df16f3c4e1dbe450357/posts/md_1705862990838/img/1__8bW3GwkVUHOsuhg2Fncrag.png)

Ok, you see raw data from database (transaction ID, block number, write set keys), it means everything is going well. But how to interact with FabEx from your service? We need a client.

All you need is inside [github.com/hyperledger-labs/fabex/client](https://pkg.go.dev/github.com/hyperledger-labs/fabex/client) package. Yes, that’s all. In fact, nothing else is needed. I’ll show you everything now.

Let’s see example of client package usage: [https://github.com/hyperledger-labs/fabex/blob/master/client/example/client.go](https://github.com/hyperledger-labs/fabex/blob/master/client/example/client.go)

You just need to import [github.com/hyperledger-labs/fabex/client](https://pkg.go.dev/github.com/hyperledger-labs/fabex/client), instantiate client object and use methods it has. Run client:

go run client/example/client.go

![](/Users/vadiminshakov/Downloads/medium-export-089ba9aeef850c38170bac7334f267b895b384bab6263df16f3c4e1dbe450357/posts/md_1705862990838/img/1__20fvyQoXuZJq7P41tQHY3w.png)

Feel free to uncomment blocks from client.go to try other queries, it’s very simple. You can integrate this logic to the your application. The client uses gRPС under the hood for effective interaction with FabEx, so you don’t have to worry about network overhead.

UI is available on [http://localhost:5252](http://localhost:5252/).