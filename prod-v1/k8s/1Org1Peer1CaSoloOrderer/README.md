# 1Org-1Peer-1Ca-SoloOrderer Network

## Instructions to run the network

> NOTE:
>
> > Configtxgen tool is not in the repository. you can download the binary tools by below code.
> > Run below commands form home directory

```bash
mkdir binaryTools && cd binaryTools
curl -sSL http://bit.ly/2ysbOFE | bash -s -- 1.4.3
```

```bash
cd <this-git-repo>/prod-v1/k8s/1Org1Peer1CaSoloOrderer/

kubectl apply -f org1-namespace.yaml

kubectl apply -f .

kubectl apply -f ./ca-org1/

```

Now we need to generarte the Crypto files and onfiguration files for orderers and peers

```bash
CA_CLIENT=$(kubectl get pods -n org1 -o=name | grep ca-client-deployment | sed "s/^.\{4\}//")

kubectl exec -n org1 $CA_CLIENT bash /scripts/genAllCerts.sh
```

Now copy the binary tool inside the container

```bash

kubectl cp ~/binaryTools/fabric-samples/bin/configtxgen org1/$CA_CLIENT:/config/configtxgen
```

Generate Configuration files through below commands

```bash

kubectl exec -n org1 $CA_CLIENT bash /scripts/genconfig.sh
```

Start orderer and Peer pods

```bash

kubectl apply -f ./orderer0/

kubectl apply -f ./peer0-org1/
```

Creating Channel

```bash

ORG1_CLI=$(kubectl get pods -n org1 -o=name | grep cli-peer0-org1 | sed "s/^.\{4\}//")

kubectl exec -ti -n org1 $ORG1_CLI -- bash -c 'peer channel create -o orderer0:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/orderer/tls/tlsca.default.svc.cluster.local-cert.pem'
```

Joining Channel

```bash
kubectl exec -n org1 $ORG1_CLI  -- bash -c 'peer channel join -b mychannel.block'
```

Install chaincode

```bash
kubectl exec -n org1 $ORG1_CLI -- bash -c 'peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/'
```

Instantiate chaincode

```bash
kubectl exec -n org1 $ORG1_CLI -- bash -c "peer chaincode instantiate -n mycc -v 1.0 -o orderer0:7050 -C mychannel -c '{\"Args\":[\"init\",\"a\",\"100\",\"b\",\"200\"]}' --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/orderer/tls/tlsca.default.svc.cluster.local-cert.pem"
```

Check if chaincode deployed

```bash
kubectl exec -n org1 $ORG1_CLI -- bash -c 'peer chaincode list --instantiated -C mychannel'
```

Invoke a transaction

```bash
kubectl exec -n org1 $ORG1_CLI -- bash -c "peer chaincode invoke -n mycc -c '{\"Args\":[\"invoke\",\"a\",\"b\",\"10\"]}' -C mychannel -o orderer0:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/orderer/tls/tlsca.default.svc.cluster.local-cert.pem"
```

Query transaction

```bash
kubectl exec -n org1 $ORG1_CLI -- bash -c "peer chaincode query -n mycc -c '{\"Args\":[\"query\",\"a\"]}' -C mychannel"
```
