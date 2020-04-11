# 1Org-1Peer-1Ca-SoloOrderer Network

### Instructions to run the network

> NOTE:
>> COnfigtxgen tool is not in the repository. you can download the binary tools by below code
Run commands form home directory
```bash
mkdir binaryTools && cd binaryTools
curl -sSL http://bit.ly/2ysbOFE | bash -s -- 1.4.6
```

```bash
cd <this-folder>

kubectl apply -f org1-namespace.yaml

kubectl apply -f .

kubectl apply -f ./ca-org1/

```
Now we need to generarte the Crypto files and onfiguration files for orderers and peers

```bash
CA_CLIENT=$(kubectl get pods -n org1 -o=name | grep ca-client-deployment)

kubectl exec -n org1 $CA_CLIENT /scripts/./genAllCerts.sh
```
Now copy the binary tool inside the container

```bash
kubectl get pods -n org1
```
Copy ca-client-deployment pod name from above output

```bash

kubectl cp ~/binaryTools/fabric-samples/bin/configtxgen org1/<paste-pod-name>:/config/configtxgen

kubectl exec -n org1 $CA_CLIENT /scripts/./genconfig.sh

kubectl apply -f ./orderer0/

kubectl apply -f ./peer0-org1/

ORG1_CLI=$(kubectl get pods -n org1 -o=name | grep cli-peer0-org1)

ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/orderer/tls/tlsca.default.svc.cluster.local-cert.pem

kubectl exec -n org1 $ORG1_CLI peer channel create -o orderer0:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls --cafile $ORDERER_CA

kubectl exec -n org1 $ORG1_CLI peer channel join -b mychannel.block

kubectl exec -n org1 $ORG1_CLI peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/

kubectl exec -n org1 $ORG1_CLI peer chaincode instantiate -n mycc -v 1.0 -o orderer0:7050 -C mychannel -c '{"Args":["init","a","100","b","200"]}' --tls --cafile $ORDERER_CA

kubectl exec -n org1 $ORG1_CLI peer chaincode list --instantiated -C mychannel

kubectl exec -n org1 $ORG1_CLI peer chaincode invoke -n mycc -c '{"Args":["invoke","a","b","10"]}' -C mychannel -o orderer0:7050 --tls --cafile $ORDERER_CA

kubectl exec -n org1 $ORG1_CLI peer chaincode query -n mycc -c '{"Args":["query","a"]}' -C mychannel
```


