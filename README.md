# K8s-Settings

### network settings
  
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF

### 파라미터 설정

    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf 
    net.bridge.bridge-nf-call-iptables = 1 
    net.bridge.bridge-nf-call-ip6tables = 1 
    net.ipv4.ip_forward = 1 
    EOF
    
    sudo sysctl --system


### swap off 및 방화벽


    sudo sed -i '/swap/d' /etc/fstab

    sudo swapoff -a

    systemctl stop firewalld

    systemctl disable firewalld


### Installing kubeadm

#### 참고 : [kubernetes v1.29](https://v1-29.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

- Update the apt package index and install packages
-     sudo apt-get update
-     # apt-transport-https may be a dummy package; if so, you can skip that package
      sudo apt-get install -y apt-transport-https ca-certificates curl gpg

- Download the public signing key for the Kubernetes package repositories
-     # If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
      # sudo mkdir -p -m 755 /etc/apt/keyrings
      curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

- Add the appropriate Kubernetes apt repository
-     # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
      echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee    /etc/apt/sources.list.d/kubernetes.list

### CNI 설치(Master node)

    sudo apt-get install -y containerd
    sudo mkdir -p /etc/containerd/

    containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
    sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

    sudo systemctl daemon-reload
    sudo systemctl start containerd
    sudo systemctl enable containerd


###  kubelet kubeadm kubectl 설치 (Master, worker node)

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl

### kubernetes cluster 초기화(Master node)

cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 to the file /boot/firmware/cmdline.txt. 

- kubeadm init
-     sudo kubeadm token create --print-join-command
      sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<master_IP> --cri-socket unix:///var/run/containerd/containerd.sock

- kubeconfig 설정
-     mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

- flannel
-     sudo swapoff -a
      sudo systemctl restart kubelet
      kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

- 이후 Join (worker node)
    
### token 재생성

    # 현재 토큰 확인
    sudo kubeadm token list

    # 토큰 재생성 및 join 명령어 생성
    sudo kubeadm token  create --print-join-command
