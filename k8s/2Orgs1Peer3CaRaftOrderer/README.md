# Network topology

| Org-Name | CA-count | Peers | couchdbs | orderers |
| -------- | -------- | ----- | -------- | -------- |
| orderer  | 1        | -     | -        | 3        |
| org1     | 1        | 2     | 2        | -        |
| org2     | 1        | 2     | 2        | -        |

- We use **orderer** org ca to generate all certificates in the network

> NOTE:
>
> > Configtxgen tool is not in the repository. you can download the binary tools by below code.
> > Run below commands form home directory

```bash
mkdir -p ~/binaryTools && cd ~/binaryTools
curl -sSL http://bit.ly/2ysbOFE | bash -s -- 1.4.3
sudo mkdir -p /var/k8s-volume/
cp ~/binaryTools/fabric-samples/bin/configtxgen /var/k8s-volume
```

## Generating certificates for all nodes in the network

Run below commands to start the CAs of all organisation _(orderer, org1, org2)_

```bash
cd <this-git-repository-path>/k8s/2Orgs1Peer3CaRaftOrderer/
kubectl apply -f ./ordererOrg/orderer-namespace.yaml
kubectl apply -f ./ordererOrg/pv-orderer/
kubectl apply -f ./ordererOrg/ca-orderer/
kubectl apply -f ./org1/org1-namespace.yaml
kubectl apply -f ./org1/pv-org1/
kubectl apply -f ./org1/ca-org1/
kubectl apply -f ./org2/org2-namespace.yaml
kubectl apply -f ./org2/pv-org2/
kubectl apply -f ./org2/ca-org2/
```

Now we will create the certificates for the network

```bash
CA_ORDERER=$(kubectl get pods -n orderer -o=name | grep ca-orderer-deployment | sed "s/^.\{4\}//")

kubectl exec -n orderer $CA_ORDERER bash /scripts/genAllCerts.sh
```

Generating Configuration files

```bash
kubectl exec -n orderer $CA_ORDERER bash /scripts/genconfig.sh
```

## Start network Nodes

Below command starts 3 orderers in a cluster. Wait for orderers to start as it may takes some time to elect a leader among the orderers in the cluster _( Refer RAFT consenses for more information)_.

```bash
kubectl apply -f ./ordererOrg/orderers/
```

After orderes have started, Run below commands to start Org1 & org2 peers

For Org1 :

```bash
kubectl apply -f ./org1/peer0-org1/
kubectl apply -f ./org1/peer1-org1/
kubectl apply -f ./org1/cli-org1/
```

For Org2 :

```bash
kubectl apply -f ./org2/peer0-org2/
kubectl apply -f ./org2/peer1-org2/
kubectl apply -f ./org2/cli-org2/
```

At this point all the network nodes are up and running

## Setting up network channel

Creating a channel _mychannel_ in the network.

```bash
ORG1_CLI=$(kubectl get pods -n org1 -o=name | grep cli-peer0-org1 | sed "s/^.\{4\}//")

kubectl exec -ti -n org1 $ORG1_CLI -- bash -c 'peer channel create -o orderer0.orderer.svc.cluster.local:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/orderer/tls/tlsca.orderer-cert.pem'
```

## Joining channel ( org1 )

```bash
kubectl exec -n org1 $ORG1_CLI  -- bash -c 'peer channel join -b mychannel.block'

kubectl exec -n org1 $ORG1_CLI  -- bash -c 'export CORE_PEER_ADDRESS=peer1-org1:7051 && peer channel join -b mychannel.block'
```

> Test if peer joined channel
>
> > ```bash
> > kubectl exec -n org1 $ORG1_CLI  -- bash -c 'peer channel list'
> > ```

## Joining channel ( org2 )

```bash
ORG2_CLI=$(kubectl get pods -n org2 -o=name | grep cli-peer0-org2 | sed "s/^.\{4\}//")

kubectl exec -n org2 $ORG2_CLI  -- bash -c 'peer channel fetch 0 mychannel.block -o orderer0.orderer.svc.cluster.local:7050 -c mychannel --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/orderer/tls/tlsca.orderer-cert.pem'

kubectl exec -n org2 $ORG2_CLI  -- bash -c 'peer channel join -b mychannel.block'

kubectl exec -n org2 $ORG2_CLI  -- bash -c 'export CORE_PEER_ADDRESS=peer1-org2:7051 && peer channel join -b mychannel.block'
```

> Test if peer joined channel
>
> > ```bash
> > kubectl exec -n org2 $ORG2_CLI  -- bash -c 'peer channel list'
> > ```

## Install chaincode ( org1 & org2 )

Installing chaincode in org1 :

```bash
kubectl exec -n org1 $ORG1_CLI -- bash -c 'peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/'

kubectl exec -n org1 $ORG1_CLI -- bash -c 'export CORE_PEER_ADDRESS=peer1-org1:7051 &&peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/'
```

> Check if chaincode is installed
>
> > ```bash
> > kubectl exec -n org1 $ORG1_CLI -- bash -c 'peer chaincode list --installed'
> > ```

Installing chaincode in org2 :

```bash
kubectl exec -n org2 $ORG2_CLI -- bash -c 'peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/'

kubectl exec -n org2 $ORG2_CLI -- bash -c 'export CORE_PEER_ADDRESS=peer1-org2:7051 &&peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/'
```

> Check if chaincode is installed
>
> > ```bash
> > kubectl exec -n org2 $ORG2_CLI -- bash -c 'peer chaincode list --installed'
> > ```

## Instantiating the chaincode

```bash
kubectl exec -n org1 $ORG1_CLI -- bash -c "peer chaincode instantiate -n mycc -v 1.0 -o orderer0.orderer.svc.cluster.local:7050 -C mychannel -c '{\"Args\":[\"init\",\"a\",\"100\",\"b\",\"200\"]}' --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/orderer/tls/tlsca.orderer-cert.pem"
```

The above command take some time to deploy the chaincode.

> Check if chaincode is deployed
>
> > ```bash
> > kubectl exec -n org1 $ORG1_CLI -- bash -c 'peer chaincode list --instantiated -C mychannel'
> > ```

## Invoke a transaction from Org1

```bash
kubectl exec -n org1 $ORG1_CLI -- bash -c "peer chaincode invoke -n mycc -c '{\"Args\":[\"invoke\",\"a\",\"b\",\"10\"]}' -C mychannel -o orderer0.orderer.svc.cluster.local:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/orderer/tls/tlsca.orderer-cert.pem"
```

> Response
>
> > You should see **_200 OK_** response

## Query transaction from Org1

```bash
kubectl exec -n org1 $ORG1_CLI -- bash -c "peer chaincode query -n mycc -c '{\"Args\":[\"query\",\"a\"]}' -C mychannel"
```

## Invoke a transaction from Org2

```bash
kubectl exec -n org2 $ORG2_CLI -- bash -c "peer chaincode invoke -n mycc -c '{\"Args\":[\"invoke\",\"a\",\"b\",\"10\"]}' -C mychannel -o orderer0.orderer.svc.cluster.local:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/orderer/tls/tlsca.orderer-cert.pem"
```

**This will take some time for the first time.**

> Response
>
> > You should see **_200 OK_** response

## Query transaction from Org2

```bash
kubectl exec -n org2 $ORG2_CLI -- bash -c "peer chaincode query -n mycc -c '{\"Args\":[\"query\",\"a\"]}' -C mychannel"
```

---

### Your Fabric network setup through Kubernetes is successfull

## Removing Network

```bash
kubectl delete -f ./ordererOrg/pv-orderer/
kubectl delete -f ./ordererOrg/ca-orderer/
kubectl delete -f ./ordererOrg/orderers/
kubectl delete -f ./ordererOrg/orderer-namespace.yaml
kubectl delete -f ./org1/pv-org1/
kubectl delete -f ./org1/ca-org1/
kubectl delete -f ./org1/peer0-org1/
kubectl delete -f ./org1/peer1-org1/
kubectl delete -f ./org1/cli-org1/
kubectl delete -f ./org1/org1-namespace.yaml
kubectl delete -f ./org2/peer0-org2/
kubectl delete -f ./org2/peer1-org2/
kubectl delete -f ./org2/cli-org2/
kubectl delete -f ./org2/pv-org2/
kubectl delete -f ./org2/ca-org2/
kubectl delete -f ./org2/org2-namespace.yaml
sudo rm -rf /var/k8s-volume/**
```

> To remove the **kubeadm** cluster refer to home README.md
