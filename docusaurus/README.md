# Kubernetes Cluster Setup Guide

## Table of Contents
- [Preparing the Control Plane](#preparing-the-control-plane)
- [Setting Up Containerd](#setting-up-containerd)
- [Initializing the Kubernetes Control Plane](#initializing-the-kubernetes-control-plane)
- [Preparing the Worker Node](#preparing-the-worker-node)
- [Installing Cilium CNI](#installing-cilium-cni)
- [Joining Worker Nodes to the Cluster](#joining-worker-nodes-to-the-cluster)
- [Setting Up External DNS with Cloudflare](#setting-up-external-dns-with-cloudflare)
- [Deploying NGINX Ingress Controller](#deploying-nginx-ingress-controller)

## Preparing the Control Plane

1. **Disable swap space**:
    ```sh
    sudo swapoff -a
    sudo vi /etc/fstab # Comment out the swap entry
    ```

2. **Configure iptables to receive bridged traffic**:
    ```sh
    sudo vi /etc/ufw/sysctl.conf
    # Add the following lines
    net/bridge/bridge-nf-call-ip6tables = 1
    net/bridge/bridge-nf-call-iptables = 1
    net/bridge/bridge-nf-call-arptables = 1
    ```

3. **Install ebtables and ethtool**:
    ```sh
    sudo apt-get install ebtables ethtool -y
    ```

4. **Enable required kernel modules**:
    ```sh
    sudo vi /etc/modules-load.d/k8s.conf
    # Add the following lines
    overlay
    br_netfilter

    sudo modprobe overlay
    sudo modprobe br_netfilter
    ```

5. **Add bridge rules and IPv4 forwarder rules**:
    ```sh
    sudo vi /etc/sysctl.d/k8s.conf
    # Add the following lines
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1

    sudo sysctl --system
    sudo reboot
    ```

6. **Add Kubernetes repository and install packages**:
    ```sh
    sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

## Setting Up Containerd

1. **Enable required kernel modules**:
    ```sh
    sudo vi /etc/modules-load.d/containerd.conf
    # Add the following lines
    overlay
    br_netfilter
    ```

2. **Reload sysctl configurations**:
    ```sh
    sudo sysctl --system
    ```

3. **Install containerd dependencies**:
    ```sh
    sudo apt install curl gnupg2 software-properties-common apt-transport-https ca-certificates -y
    ```

4. **Add containerd repository and install containerd**:
    ```sh
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update
    sudo apt-get install containerd.io -y
    ```

5. **Configure containerd**:
    ```sh
    sudo mkdir -p /etc/containerd
    sudo containerd config default | sudo tee /etc/containerd/config.toml

    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

    sudo systemctl restart containerd
    sudo systemctl enable containerd
    ```

## Initializing the Kubernetes Control Plane

1. **Initialize the control plane**:
    ```sh
    sudo kubeadm init --pod-network-cidr=10.1.1.0/24 --apiserver-advertise-address=<IP_ADDRESS_OF_NODE> --apiserver-cert-extra-sans=<EXTERNAL_IP_ADDRESS_OF_CONTROL_PLANE>
    ```

2. **Configure kubectl for the root user**:
    ```sh
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

## Preparing the Worker Node

1. **Repeat the steps from the control plane setup up to the point of Kubernetes and containerd installation**.

## Installing Cilium CNI

1. **Download and install Cilium**:
    ```sh
    curl -LO https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
    sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
    rm cilium-linux-amd64.tar.gz
    sudo cilium install
    ```

## Joining Worker Nodes to the Cluster

1. **Join the worker node to the cluster using the command provided by `kubeadm init`**:
    ```sh
    sudo kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
    ```

## Setting Up External DNS with Cloudflare

1. **Install External DNS using Helm**:
    ```sh
    helm install external-dns-app \
      --set provider=cloudflare \
      --set cloudflare.secretName=cloudflare-secret \
      oci://registry-1.docker.io/bitnamicharts/external-dns
    ```

## Deploying NGINX Ingress Controller

1. **Apply the NGINX Ingress Controller manifest**:
    ```sh
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/baremetal/deploy.yaml
    ```