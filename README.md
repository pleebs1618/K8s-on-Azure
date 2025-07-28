# K8s on Azure
Notes created while following Kelsey Hightower's 'K8s The Hard Way'

## Kubernetes the Hard Way on Azure

This guide documents how to manually set up a Kubernetes cluster on Microsoft Azure using Windows as the management machine.
It follows the concepts of Kelsey Hightower’s “Kubernetes the Hard Way”, adapted for Azure and Windows PowerShell.

## Prerequisites

Install Azure CLI and Create a Resource Group

```
az login
az group create -n kubernetes -l eastus2
```

## Install Client Tools

also verifies installation

```
Invoke-WebRequest -Uri https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssl_1.6.3_windows_amd64.exe -OutFile cfssl.exe
Invoke-WebRequest -Uri https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssljson_1.6.3_windows_amd64.exe -OutFile cfssljson.exe
.\cfssl.exe version
```

## Install kubectl via Chocolatey

```
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
choco install kubernetes-cli
kubectl version -oyaml --client
```

## Kubernetes Networking Overview
Kubernetes networking addresses four key problems:

1. Container-to-Container Communication – Handled by Pods (containers share a network namespace).
2. Pod-to-Pod Communication – Solved with CNI plugins like Calico, Flannel, or Cilium.
3. Pod-to-Service Communication – Managed with Kubernetes Services (ClusterIP, DNS).
4. External-to-Service Communication – Exposed via NodePort, LoadBalancer, or Ingress.

### Example of a Service Manifest

```
kind: Service
metadata:
  name: sas-compute-service
spec:
  selector:
    app: sas
    tier: compute
```

### Ingress Manifest

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sas-ingress
spec:
  rules:
  - host: sas.example.com
    http:
      paths:
      - path: /SASStudio
        pathType: Prefix
        backend:
          service:
            name: sas-studio-service
            port:
              number: 80
```

## Provision Azure Networking

### Create Virtual Network and Subnet

```
az network vnet create -g kubernetes `
  -n kubernetes-vnet `
  --address-prefix 10.240.0.0/24 `
  --subnet-name kubernetes-subnet
```

### Add Rules for SSH and Kubernetes API

```
az network nsg rule create -g kubernetes `
  -n kubernetes-allow-ssh `
  --access allow `
  --destination-port-range 22 `
  --direction inbound `
  --nsg-name kubernetes-nsg `
  --protocol tcp `
  --priority 1000

az network nsg rule create -g kubernetes `
  -n kubernetes-allow-api-server `
  --access allow `
  --destination-port-range 6443 `
  --direction inbound `
  --nsg-name kubernetes-nsg `
  --protocol tcp `
  --priority 1001

```

### Create Controller Nodes

```
$UBUNTULTS = "Canonical:0001-com-ubuntu-server-focal:20_04-lts-gen2:latest"
$VMSize = "Standard_B1s"

az vm availability-set create -g kubernetes -n controller-as

for ($i = 0; $i -lt 2; $i++) {
    az network public-ip create --sku Standard -n "controller-$i-pip" -g "kubernetes"
    az network nic create -g "kubernetes" -n "controller-$i-nic" `
        --private-ip-address "10.240.0.1$i" `
        --public-ip-address "controller-$i-pip" `
        --vnet "kubernetes-vnet" `
        --subnet "kubernetes-subnet" `
        --ip-forwarding
    az vm create -g "kubernetes" -n "controller-$i" `
        --image $UBUNTULTS --size $VMSize `
        --nics "controller-$i-nic" `
        --availability-set "controller-as" `
        --generate-ssh-keys `
        --admin-username "kuberoot"
}
```

Creation of CA, ca-csr.json, ca-config.json, ca.pem, ca-key.pem...

### Bootstrapping etcd

```
sudo mv etcd.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
etcdctl member list
```

### Control Plane Setup

```
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
kubectl get componentstatuses --kubeconfig admin.kubeconfig
```

### Worker Nodes Bootstrap

Install and configure:

- containerd
- runc or runsc
- CNI plugins
- kubelet
- kube-proxy
