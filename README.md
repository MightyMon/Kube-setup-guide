# Kubernetes Cluster Setup 

---
## Message for Everyone 

Use it properly, you got almost everything you would need to install kubernetes. Google(specifically searchand not gemini) is your best friend, and dont dare using gpt generated command snippets , if you dont know what is being done.Also please pardon me for any spelling or grammar mistake.
*<p align="right"><i>Goutham(mightymon)</i></p>*

---
## Table of Contents

1. [Prerequisites](#prerequisites)  
2. [Linux Servers Setup](#linux-servers-setup)  
3. [Local DNS Configuration](#local-dns-configuration)  
4. [RKE2 Installation](#rke2-installation)  
   - [RKE2 Server Installation](#rke2-server-installation)  
   - [RKE2 Agent Installation](#rke2-agent-installation)  
5. [Rancher Installation](#rancher-installation)  
6. [Longhorn Installation](#longhorn-installation)  
7. [NeuVector Installation](#neuvector-installation)  
8. [Automation](#automation)  
9. [If a GPU cluster](#if-a-gpu-cluster)
---

## Prerequisites

- **Linux machines** (Ubuntu 22.04 x64 recommended) with internet access.  
- **Minimum specs per node:**  
  - 4 CPU cores  
  - 8GB RAM  
  - 60GB disk space  
- **SSH client** for remote access.  
- **Local DNS server** (`dnsmasq` or equivalent) for internal hostname resolution.  
- **Firewall disabled** on all nodes.

---

## Linux Servers Setup

Example server configuration:

| Name        | CPU | Memory | IP Address      | Disk (GB) | OS               |
|-------------|-----|--------|-----------------|-----------|------------------|
| rancher-01  | 4   | 8Gi    | 192.168.1.12   | 60        | Ubuntu 22.04 x64 |
| rancher-02  | 4   | 8Gi    | 192.168.1.74   | 60        | Ubuntu 22.04 x64 |
| rancher-03  | 4   | 8Gi    | 192.168.1.247  | 60        | Ubuntu 22.04 x64 |

### Disable Firewall and Install Dependencies

Run on all nodes:

```bash
sudo systemctl disable --now ufw
sudo apt update && sudo apt upgrade -y
sudo apt install -y nfs-common curl
sudo apt autoremove -y
```
#### Note: if you have gpu in any node: Install nvidia and cuda drivers properly and check if you get the output of nvidia-smi correctly
---
### setup proxy for system and for rke2-server service(IMP if you dont have internet)

## Local DNS Configuration



Setup and Verify DNS Resolution:

```bash
dig rancher.local #not in dns server, check in some other system with the dns config
```

---

## RKE2 Installation

### RKE2 Server Installation (rancher-01)

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh -

sudo mkdir -p /etc/rancher/rke2/
echo "token: JustYetAnotherNodeForUs" | sudo tee /etc/rancher/rke2/config.yaml

sudo systemctl enable --now rke2-server.service
```

Validate Installation:

```bash
systemctl status rke2-server.service
```

Configure `kubectl`:

```bash
sudo ln -s $(find /var/lib/rancher/rke2/data/ -name kubectl) /usr/local/bin/kubectl
echo "export KUBECONFIG=/etc/rancher/rke2/rke2.yaml" >> ~/.bashrc
source ~/.bashrc
kubectl get nodes
```

### RKE2 Agent Installation (rancher-02 and rancher-03)

```bash
export RANCHER1_IP=192.168.1.12 #replace with correct ip

curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=agent sh -

sudo mkdir -p /etc/rancher/rke2/
cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
server: https://$RANCHER1_IP:9345
token: JustYetAnotherNodeForUs
EOF

sudo systemctl enable --now rke2-agent.service
```

Validate Nodes:

```bash
kubectl get nodes -o wide # wait till everything comes up , it might take some time , patience is key 
```

---

## Rancher Installation
<span style="color:red">Don't install rancher if you need to import the cluster to the existing rancher UI </span>


### Install Helm

```bash
curl -#L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest --force-update
helm repo add jetstack https://charts.jetstack.io --force-update
```

### Install Cert Manager

```bash
helm upgrade -i cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace --set crds.enabled=true
```

### Install Rancher

```bash
helm upgrade -i rancher rancher-latest/rancher \
  --create-namespace \
  --namespace cattle-system \
  --set hostname=rancher.local \
  --set bootstrapPassword=thisisnotacommonpassword \
  --set replicas=1
```
Wait till all the pods are up and running

Access Rancher UI: [https://rancher.local](https://rancher.local)

---

## Longhorn Installation

```bash
apt install open-iscsi # in all nodes 
```
```bash
helm repo add longhorn https://charts.longhorn.io --force-update
helm upgrade -i longhorn longhorn/longhorn \
  --namespace longhorn-system --create-namespace
```



---

## NeuVector Installation

```bash
Install it via GUI in rancher
```
## Grafana & promethus Installation

```bash
Install it via GUI in rancher ,look for monitoring in apps in rancher
```


---
## Automation
We will think about that later :)

## If a GPU cluster
Turn on MIG in mig enabled GPU-s if needed
```bash
helm install --wait --generate-name \
nvidia/gpu-operator \
--set operator.defaultRuntime=containerd \
--set toolkit.env[0].name=CONTAINERD_CONFIG \
--set toolkit.env[0].value=/etc/containerd/config.toml \
--set toolkit.env[1].name=CONTAINERD_SOCKET \
--set toolkit.env[1].value=/run/containerd/containerd.sock \
--set toolkit.env[2].name=CONTAINERD_RUNTIME_CLASS \
--set toolkit.env[2].value=nvidia \
--set toolkit.env[3].name=CONTAINERD_SET_AS_DEFAULT \
--set-string toolkit.env[3].value=true \
--set mig.strategy=mixed
```
