# K8S_Offline_Install
-----
### install offline K8S in centos Flow

* [Reference](https://docs.genesys.com/Documentation/GCXI/latest/Dep/DockerOffline)

##### Prepare Needed Software at inline env(With internet)

1\. Download Centos Software(Assume you have installed docker)

* `/YourLocalFolder` is your computer's local folder.

```
# setting your repository
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

# download need software into specified local folder
yum install -y --downloadonly --downloaddir=/YourLocalFolder kubeadm-1.21.* kubelet-1.21.* kubectl-1.21.* ebtables
```

2\. Download need K8S Docker's image(By Inline environment)
```
# Pull Docker's image by inline environment(Must For Download and Install Kubernetes Network)
docker pull k8s.gcr.io/kube-apiserver:v1.21.2 
docker pull k8s.gcr.io/kube-controller-manager:v1.21.2 
docker pull k8s.gcr.io/kube-scheduler:v1.21.2
docker pull k8s.gcr.io/kube-proxy:v1.21.2
docker pull k8s.gcr.io/pause:3.4.1
docker pull k8s.gcr.io/etcd:3.4.13-0
docker pull k8s.gcr.io/coredns/coredns:v1.8.0
# For K8S CNI (Container Network Interface) need
docker pull rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.0
docker pull quay.io/coreos/flannel:v0.15.1


# save above image into tar in a specified folder (choose one of below methods)
# method 1 (By Terminal command)
LocalFolder="/YourLocalFolder/"
docker images | grep k8s | awk '{print $1 , $2}' | { 
while read param1 param2;
do
docker save $param1":"$param2 > $LocalFolder$( echo $param1 | cut -d'/' -f2).tar
done;
}
docker save rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.0 > /YourLocalFolder/mirrored-flannelcni-flannel-cni-plugin_v1.0.0.tar
docker save quay.io/coreos/flannel:v0.15.1 > /YourLocalFolder/flannel_v0.15.1

# method 2 (If above method is failed)
docker save k8s.gcr.io/kube-apiserver:v1.21.2 > /YourLocalFolder/kube-apiserver_v1.21.2.tar
docker save k8s.gcr.io/kube-controller-manager:v1.21.2 > /YourLocalFolder/kube-controller-manager_v1.21.2.tar
docker save docker pull k8s.gcr.io/kube-scheduler:v1.21.2 > /YourLocalFolder/kube-scheduler_v1.21.2.tar
docker save k8s.gcr.io/kube-proxy:v1.21.2 > /YourLocalFolder/kube-proxy_v1.21.2.tar
docker save k8s.gcr.io/pause:3.4.1 > /YourLocalFolder/pause_3.4.1.tar
docker save k8s.gcr.io/etcd:3.4.13-0 > /YourLocalFolder/etcd:3.4.13-0.tar
docker save k8s.gcr.io/coredns/coredns:v1.8.0 > /YourLocalFolder/coredns_v1.8.0.tar
docker save rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.0 > /YourLocalFolder/mirrored-flannelcni-flannel-cni-plugin_v1.0.0.tar
docker save quay.io/coreos/flannel:v0.15.1 > /YourLocalFolder/flannel_v0.15.1
```

3\. Download `kube-flannel.yml` from [github](https://github.com/flannel-io/flannel)
```
curl https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml -o /YourLocalFolder/kube-flannel.yml
```
4\. Copy `admin.conf` to specified folder
```
cp /etc/kubernetes/admin.conf $LocalFolder
```

##### Install prepared software in offline env (Without internet)

* `/YourLocalFolder` is above folder which storage needed software.

0\. Setting the folder which storage needed softwares
```
LocalFolder='/YourLocalFolder' 
```

1\. Close swap(Install K8S, it need to close swap)
```
sudo swapoff -a
```

2\. Install Software by specified folder
```
sudo yum install -y --cacheonly --disablerepo=* $( $LocalFolder'*.rpm' )
```

3\. Docker install K8S images from specified folder(For Worker, there is no need)
```
ls $LocalFolder | grep 'tar' | { 
while read param1;
do
docker load -i $LocalFolder$param1
done;
}
```

4\. Ensure that SELinux is in permissive mode 
```
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

5\. Setting ``/etc/sysctl.d/k8s.conf`
```
echo "net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
" | sudo tee /etc/sysctl.d/k8s.conf
sysctl --system
```

6\. To start using your cluster, you need to run the following as a regular user
```
# for worker need to execute : cp $LocalFolder'admin.conf' /etc/kubernetes/admin.conf 
sudo mkdir -p /root/.kube
sudo cp -i /etc/kubernetes/admin.conf /root/.kube/config
sudo chown $(id -u):$(id -g) /root/.kube/config
```

6\. Configure kubectl autocompletion
```
echo "source <(kubectl completion bash)" | sudo tee /root/.bashrc
```

7\. On the PRIMARY(master) machine only, create a cluster, deploy the Flannel network, and schedule pods
* Execute the following command to set up a Kubernetes cluster 
```
sudo kubeadm init --pod-network-cidr=192.168.100.150/16 --kubernetes-version=1.21.2
```
* Execute the following command to verify the node is running:
```
sudo kubectl get nodes
```
>> The Status should be `NotReady`
![](https://i.imgur.com/3CkNUtJ.png)

* Execute the following commands to configure kubectl to manage your cluster
```
sudo grep -q "KUBECONFIG" /root/.bashrc || {
    echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' | sudo tee -a /root/.bashrc
    . /root/.bashrc
}
```

* Deploy the [Flannel](https://github.com/flannel-io/flannel/blob/master/Documentation/kube-flannel.yml) overlay network on the PRIMARY machine
```
sudo kubectl apply -f /YourLocalFolder/kube-flannel.yml
```

* Ensure that kube-dns* pods are working (Each Pod's status should be`Running`)
```
sudo kubectl get pods --all-namespaces
```
![](https://i.imgur.com/EqHzhe5.png)

* Schedule pods:  Executing this command removes the node-role.kubernetes.io/master taint from any nodes that have it, so that the scheduler can schedule pods everywhere:
```
sudo kubectl taint nodes --all node-role.kubernetes.io/master-
```
   
8\. On the SECONDARY(Worker) machines only, join the SECONDARY to the cluster:

```
systemctl enable kubelet.service
```

* Copy master's file:`admin.conf` to worker kubernetes's folder
```
cp $( $LocalFolder'admin.conf' ) /etc/kubernetes/admin.conf 
```

* check master's token
* * Token only lives within 24 hours (That means the token will be changed after 24 hours)
```
token=$( sudo kubeadm token list | grep system | awk '{print $1}' )
```
* check sha256's hash
```
hash=$(sudo openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //')
```

* Worker joins the cluster
```
kubeadm join --token $token <primary-ip>:<primary-port> --discovery-token-ca-cert-hash sha256:$hash
```


sudo kubectl get nodes --all-namespaces


Token only live 24 hours
```
kubeadm join 10.0.3.15:6443 --token ju9vqy.4o0r5kdida74okqs \
        --discovery-token-ca-cert-hash sha256:e621858e9494a28b6a1b90d8ff263ad0601c3a6331fb5785ec94824142c28336
        
        
kubeadm join 192.168.100.150:6443 --token ju9vqy.4o0r5kdida74okqs \
        --discovery-token-ca-cert-hash sha256:e621858e9494a28b6a1b90d8ff263ad0601c3a6331fb5785ec94824142c28336        
```

##### Error Action
* [ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml](https://tree.rocks/kubernetes-with-multi-server-node-setup-on-ubuntu-server-280066e6b106)
```
sudo kubeadm reset --cri-socket=/var/run/crio/crio.sock

sudo swapoff -a

sudo kubeadm init --pod-network-cidr=192.168.100.150/16 --kubernetes-version=1.21.2
```

* [Unable to connect to the server: x509: certificate signed by unknown authority](https://blog.csdn.net/woay2008/article/details/93250137)
* 删除集群然后重新创建也算是一个常规的操作，如果你在执行 kubeadm reset命令后没有删除创建的 $HOME/.kube目录，重新创建集群就会出现这个问题！
```
sudo rm -rf /root/.kube
sudo mkdir -p /root/.kube
sudo cp -i /etc/kubernetes/admin.conf /root/.kube/config
sudo chown $(id -u):$(id -g) /root/.kube/config
```

##### Check K8S Status
```
sudo kubectl get nodes
sudo kubectl get pods --all-namespaces
```

[![hackmd-github-sync-badge](https://hackmd.io/IY5xpzk2Tfa6t42OujNhIQ/badge)](https://hackmd.io/IY5xpzk2Tfa6t42OujNhIQ)
