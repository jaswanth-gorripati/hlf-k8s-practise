# Starting Orderer Org

```bash
kubectl apply -f orderer-namespace.yaml
kubectl apply -f pv-orderer/
kubectl apply -f pv-orderer/
```

Inside ca-orderer pod

```bash
export FABRIC_CA_CLIENT_TLS_CERTFILES=/certs/ca-orderer/ca-cert.pem

export FABRIC_CA_CLIENT_HOME=/etc/hyperledger/orderer-tls/admin

fabric-ca-client enroll -u https://ca-orderer-admin:ca-orderer-adminpw@ca-orderer:7054
gencerts.sh orderer:orderer:orderer.svc.cluster.local:3
```

now start ca-of org1

```bash
kubectl apply -f ./org1/org1-namespace.yaml
kubectl apply -f ./org1/pv-org1/
kubectl apply -f ./org1/ca-org1/
```

Inside ca-orderer pod

```bash
export FABRIC_CA_CLIENT_TLS_CERTFILES=/certs/ca-org1/ca-cert.pem

export FABRIC_CA_CLIENT_HOME=/etc/hyperledger/org1-tls/admin

fabric-ca-client enroll -u https://ca-org1-admin:ca-org1-adminpw@ca-org1.org1.svc.cluster.local:7054
gencerts.sh peer:org1:org1.svc.cluster.local:1
```

Now start ca-of org2

```bash
kubectl apply -f ./org2/org2-namespace.yaml
kubectl apply -f ./org2/pv-org2/
kubectl apply -f ./org2/ca-org2/
```

Inside ca-orderer pod

```bash
export FABRIC_CA_CLIENT_TLS_CERTFILES=/certs/ca-org2/ca-cert.pem

export FABRIC_CA_CLIENT_HOME=/etc/hyperledger/org2-tls/admin

fabric-ca-client enroll -u https://ca-org2-admin:ca-org2-adminpw@ca-org2.org2.svc.cluster.local:7054
gencerts.sh peer:org2:org2.svc.cluster.local:1
```
