#Ubuntu OS
#Please execute below commands on all nodes used for kubernetes cluster.
containerd:

cat > /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF


    modprobe overlay
    modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

    sysctl --system

#Install containerd package on all nodes
#Here, i will suggest to follow the offical doc link because offical doc is holding all update.
#Doc link: "https://docs.docker.com/engine/install/ubuntu/"
# Note: Follow till steps "#Install Container Engine" which is mention below.

    sudo apt-get update
    sudo apt-get install ca-certificates curl gnupg
    for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

#Set up the repository
#Update the apt package index and install packages to allow apt to use a repository over HTTPS:
    sudo apt-get update
    sudo apt-get install ca-certificates curl gnupg

#Add Docker’s official GPG key:

    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

#Use the following command to set up the repository:

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

#Install Container Engine:
    sudo apt-get update
    sudo apt-get install containerd.io

# Configure containerd
    mkdir -p /etc/containerd
    containerd config default > /etc/containerd/config.toml
    sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Restart containerd
    systemctl restart containerd

# To execute crictl CLI commands, ensure we create a configuration file as mentioned below
    cat /etc/crictl.yaml
Output: No files found than,

    vi /etc/crictl.yaml
        runtime-endpoint: unix:///run/containerd/containerd.sock
        image-endpoint: unix:///run/containerd/containerd.sock
        timeout: 2
    :wq


#Install kubernetes packages on all nodes.
#Recommanded: Offical web doc link "https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/"


#Update the apt package index and install packages needed to use the Kubernetes apt repository:

    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl

#Download the Google Cloud public signing key:

    curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

#Add the Kubernetes apt repository:

    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

#Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl

#On control nodes (k8s master node):
#We are bootstrapping control plane node.
#"--pod-network-cidr" waswill be change here on-permisses  or cloud setup. 
#So instead of applying the calico YAML files, first download them and update the CIDR of pod to be used cidr which ever "For example: 10.244.0.0/16 private class".

    kubeadm init --apiserver-advertise-address=192.168.1.90 --cri-socket=/run/containerd/containerd.sock --pod-network-cidr=10.244.0.0/16

# --apiserver-advertise-address: Controller IP (Private/public)

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

#calico network for kubernetes CNI
    kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
    wget https://docs.projectcalico.org/manifests/custom-resources.yaml


or

Please refer the official document before proceeding further
https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart

Note:
- Since I am using subnet for POD as 10.244.0.0/16 instead of default 192.168.0.0/16 which will conflict with my nodes ip address. First I downloaded custom-resources.yaml file and updates below parameter.
- cidr: 10.244.0.0/16


# After update, apply the YAML file
kubectl apply -f custom-resources.yaml 

#On worker nodes to join the cluster:
#See the output of kubeadm init command after successfull. It will print kubeadm join command output to be executed on worker nodes:
#For example:

    kubeadm join 10.128.0.3:6443 --token wt98bb.4gsf0qqicl5vg0gm \
            --discovery-token-ca-cert-hash sha256:1c4bc8d73e4d45e78569ff3644e67b75946289cb41adecb26cedafb0608df64f 

#On contol node:
#Now check all the pods deployed using kubeadm are running, before checking the node status:
    kubectl get pods --all-namespaces

Note: By default control node will not be able to launch the pod, so to enable execute below command:

    kubectl taint nodes --all node-role.kubernetes.io/master-
    cli tool to manage CRI like containerd, cri-o etc.,
    root@containerdmanager:~/pods# cat /etc/crictl.yaml
    runtime-endpoint: unix:///run/containerd/containerd.sock
    image-endpoint: unix:///run/containerd/containerd.sock
    timeout: 10
    debug: true
    root@containerdmanager:~/pods# 


root@containerdmanager:~/pods# crictl images
IMAGE                                 TAG                 IMAGE ID            SIZE
docker.io/calico/cni                  v3.15.1             2858353c1d25f       79.3MB
docker.io/calico/kube-controllers     v3.15.1             8ed9dbffe3501       22MB
docker.io/calico/node                 v3.15.1             1470783b14749       90.8MB
docker.io/calico/pod2daemon-flexvol   v3.15.1             a696ebcb2ac78       37.5MB
docker.io/calico/typha                v3.15.1             38d40645f58bd       21.5MB
k8s.gcr.io/coredns                    1.6.7               67da37a9a360e       13.6MB
k8s.gcr.io/etcd                       3.4.3-0             303ce5db0e90d       101MB
k8s.gcr.io/kube-apiserver             v1.18.6             56acd67ea15a3       51.1MB
k8s.gcr.io/kube-controller-manager    v1.18.6             ffce5e64d9156       49.1MB
k8s.gcr.io/kube-proxy                 v1.18.6             c3d62d6fe4120       49.3MB
k8s.gcr.io/kube-scheduler             v1.18.6             0e0972b2b5d1a       34.1MB
k8s.gcr.io/pause                      3.1                 da86e6ba6ca19       317kB
k8s.gcr.io/pause                      3.2                 80d28bedfe5de       300kB
quay.io/tigera/operator               v1.7.1              8b48553703849       46.3MB
root@containerdmanager:~/pods# crictl pods
POD ID              CREATED             STATE               NAME                                        NAMESPACE           ATTEMPT
98db6794c4a27       2 hours ago         Ready               calico-kube-controllers-5687f44fd5-scbzs    calico-system       0
e4f75a2569350       2 hours ago         Ready               coredns-66bff467f8-5kjrx                    kube-system         0
d7d9a090bf94d       2 hours ago         Ready               coredns-66bff467f8-5l5c5                    kube-system         0
985b4a4cfd63f       2 hours ago         Ready               calico-node-6kwh9                           calico-system       0
b47a7c9983129       2 hours ago         Ready               calico-typha-7cf64cf698-5rtvc               calico-system       0
83d3551062e2e       2 hours ago         Ready               tigera-operator-6659cdcd96-gsxhf            tigera-operator     0
88a68133f59a9       2 hours ago         Ready               kube-proxy-fjxzv                            kube-system         0
1a1821d80cf25       2 hours ago         Ready               kube-apiserver-containerdmanager            kube-system         0
03bb53a905231       2 hours ago         Ready               kube-controller-manager-containerdmanager   kube-system         0
eed86b358faa2       2 hours ago         Ready               kube-scheduler-containerdmanager            kube-system         0
ae5737f24c28b       2 hours ago         Ready               etcd-containerdmanager                      kube-system         0
root@containerdmanager:~/pods#
If you want to create pod with crictl client, please follow the steps below:
$ cat pod-config.json
{
    "metadata": {
        "name": "nginx-sandbox",
        "namespace": "default",
        "attempt": 1,
        "uid": "hdishd83djaidwnduwk28bcsb"
    },
    "log_directory": "/tmp",
    "linux": {
    }
}

$ crictl runp pod-config.json
f84dd361f8dc51518ed291fbadd6db537b0496536c1d2d6c05ff943ce8c9a54f

$ crictl pods
