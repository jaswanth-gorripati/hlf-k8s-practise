# fabric-ca

Starting fabric-ca with tls enabled

## Creating a channel

```bash
export CHANNEL_NAME=mychannel  && ./configtxgen -profile MainChannel -outputCreateChannelTx ./channel.tx -channelID $CHANNEL_NAME
```

## Creating a Genesis block

```bash
./configtxgen -profile OrdererGenesis -channelID byfn-sys-channel -outputBlock ./channel-artifacts/genesis.block
```

## Copy configtxgen into fabric-client folder

```bash
kubectl cp ~/fabric-bins/1.4.3/configtxgen org1/$CA_CLIENT:/config/configtxgen
```
