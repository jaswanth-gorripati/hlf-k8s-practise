# Pre-requirements

1. install kubeadm
2. install kubectl
3. install Docker

## Start the cluster in KUBEADM

> To start the clulster in kubeadm we need to use flannel cri

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
```

To configure **kubectl** to connect cluster

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

then run below command to start flannel

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
```

then run below command for ingress addon

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
```

## Remove cluster

```bash
kubectl drain k8-virtualbox --delete-local-data --force --ignore-daemonsets
kubectl delete node k8-virtualbox
kubectl config delete-cluster
rm -rf $HOME/.kube/*
kubeadm reset
```

## Error 1: Detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd"

**Posible fix** : _Change docker cgroup driver to systemd_

```bash
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

systemctl daemon-reload

systemctl restart docker
```

And also add below line in **_'/etc/systemd/system/kubelet.service.d/10-kubeadm.conf'_** file

```bash
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"
```

then

```bash
systemctl daemon-reload

systemctl restart kubelet
```

## Error 2: Running with swap on is not supported. Please disable swap

**possible fix**: _Turn swap off_

```bash
 sudo swapoff -a
```

## In order to able to deploy the pods in master node

**Problem :**

> Master node will have a taint **node-role.kubernetes.io/master:NoSchedule** ( to get taints of node use **kubectl describe node k8s-virtualbox | grep Taints** )

we need to remove the taint of the master node using below command :

```bash
kubectl taint node k8s-virtualbox node-role.kubernetes.io/master:NoSchedule-
```

## To deploy the dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

Then create a default user ( admin ) using the files already written in _~/k8s-dashboard_ folder

to get the token run

```bash
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

Then go to **http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/** for UI
