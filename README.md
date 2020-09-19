# Raspberry Pi 4b Kubernetes Cluster

## Overview

Kubernetes is a very powerful platform to scale your applications, and the Raspberry Pi is a low-cost computer with excellent power efficiency you can use to run tasks without breaking the bank. Canonical recently released Ubuntu 20.04, with full support for the Raspberry Pi. In this video, we take a look at how to create a Pi-powered Kubernetes cluster based on Ubuntu.

| Relevant Links |
| ---
| [Original Video](https://www.youtube.com/watch?v=qv3_gLvjITk&feature=youtu.be)
| [Ubuntu 20.04 Download (Pi Version)](https://ubuntu.com/download/raspberry-pi)
| [Original Source](https://wiki.learnlinux.tv/index.php/Setting_up_a_Raspberry_Pi_Kubernetes_Cluster_with_Ubuntu_20.04)

## What You'll Need

    At least two Raspberry Pi 4 boards
    A Raspberry-certified power supply for each
    An SD card with decent speed
    SD card flashed with Ubuntu 20.04 for each


## Set-up Process (do the following on each Raspberry Pi)

### Edit the host name

Edit /etc/hosts and /etc/hostname on the SD card to the actual name of the instance

For example:

```
k8s-master
```

```
k8s-worker-01
```

(Or whatever naming scheme you wish)

### Configure boot options

Edit /boot/firmware/cmdline.txt and add:

```bash
sudo nano /boot/firmware/cmdline.txt
```

```
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1
```

Note: Add that to the end of the first line, do not create a new line.

### Install all updates

```bash
sudo apt update && sudo apt dist-upgrade
```

### Reboot

Reboot each Pi:

```bash
sudo reboot
```

### Create a user for yourself

```bash
sudo adduser john
usermod -aG sudo john
```

### Install Docker

```bash
curl -sSL get.docker.com | sh
sudo usermod -aG docker john
```

### Set Docker daemon options

Edit the daemon.json file (this file most likely won't exist yet)

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```

### Enable routing

Find the following line in the file:

/etc/sysctl.conf

```bash
 #net.ipv4.ip_forward=1
```
Uncomment that line.

### Reboot again

```bash
sudo reboot
```

### Test that docker is working properly

Check docker daemon:

```bash
systemctl status docker
```

### Run the hello-world container:

```bash
docker run hello-world
```

### Add Kubernetes repository

```bash
sudo nano /etc/apt/sources.list.d/kubernetes.list
```

Add:

```
deb http://apt.kubernetes.io/ kubernetes-xenial main
```

Add the GPG key to the Pi:

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

### Install required Kubernetes packages

```bash
sudo apt update
sudo apt install kubeadm kubectl kubelet
```

Note: If you get errors with the first command, wait a few minutes and try again.

## Master-only

### Initialize Kubernetes

Run:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Once this runs, you will get some output that will include the join command, but don't join nodes yet. Copy this somewhere for later.

### Set up config directory

The previous command will give you three additional commands to run, most likely these:

```bash
mkdir -p ~.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Go ahead and run those, but if it recommends different commands, run those instead.

### Install flannel network driver

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Note: The lack of sudo is intentional
Make sure all the pods come up

```bash
kubectl get pods --all-namespaces
```

### Join worker nodes to the cluster

Once all of the pods have come up, run the join command on each worker node. This command was provided in an earlier step.

### Check status of nodes

See if the nodes have joined successfully, run the following command a few times until everything is ready:

```bash
kubectl get nodes
```
pod.yml

```yml
 apiVersion: v1
 kind: Pod
 metadata:
   name: nginx-example
   labels:
     app: nginx
 spec:
   containers:
     - name: nginx
       image: linuxserver/nginx
       ports:
         - containerPort: 80
           name: "nginx-http"
```

service-nodeport.yml

```yml
 apiVersion: v1
 kind: Service
 metadata:
   name: nginx-example
 spec:
   type: NodePort
   ports:
     - name: http
       port: 80
       nodePort: 30080
       targetPort: nginx-http
   selector:
     app: nginx
```

Apply the pod yaml file

```bash
kubectl apply -f pod.yml
```
Check the status with:

```bash
kubectl get pods
```
Check the status with more info:

```bash
kubectl get pods -o wide
```

Apply the service yaml file

```bash
kubectl apply -f service-nodeport.yml
```

Check the status with:

```bash
kubectl get service
```