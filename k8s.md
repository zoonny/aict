# k8s 클러스터 구축

## 참고
> https://velog.io/@lavi15/%EA%B0%9C%EB%B0%9C%EC%9D%BC%EC%A7%80Ubuntu-%EC%84%9C%EB%B2%84%EC%97%90-K8S-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0

## 구성 환경

- Azure VM 3대 (Ubuntu 24.04LTS)
- Kubernetes v1.30.7 (containerd | calico v3.28.1)


|VM|사양|Private IP|Public IP|
|-|-|-|-|
|sidev-k8s-master|2vCore,2GB|52.231.104.37|10.18.0.69|
|sidev-k8s-worker1|1vCore,1GB|52.231.109.105|10.18.0.70|
|sidev-k8s-worker2|1vCore,1GB|52.231.109.179|10.18.0.71|

- Network CIDR: 10.18.0.0/24

## 사전 준비

```shell
sudo apt-get update
sudo apt-get upgrade -y
sudo apt install net-tools -y 

sudo su -

# 코어확인
nproc
#메모리 확인
free -h
```

## containerd 설치

```shell
# swapoff 진행
swapoff -a

# 커널 모듈 및 네트워크 설정
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

# containerd 설치
apt install -y containerd

mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml

# SystemdCgroup을 false에서 true로 변경
# sandbox_image를 "registry.k8s.io/pause:3.9"로 변경
vi /etc/containerd/config.toml

systemctl restart containerd.service
```

## kubeadm 설치

```shell
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet

# control-plane에서만 진행 (control-plane과 worker node 의 ip에 따라 ip 변경)
kubeadm init --apiserver-advertise-address 10.18.0.69 --pod-network-cidr=10.18.0.0/24

# control-plane에서만 진행
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# worker node 에서만 진행 (control-plane에서 init 성공시 나오는 명령어 사용)
kubeadm join 10.18.0.69:6443 --token 45jzsc.y10w4dw6imbl4dqc \
        --discovery-token-ca-cert-hash sha256:20ad27cb0faec91f4e910251c04a43a4874c0186c843c0181c6cbff8b5fc635e
```



### kubeadm int 실행 결과

```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.18.0.69:6443 --token 45jzsc.y10w4dw6imbl4dqc \
        --discovery-token-ca-cert-hash sha256:20ad27cb0faec91f4e910251c04a43a4874c0186c843c0181c6cbff8b5fc635e
```

### cni(calico) 설치
- Master Node에서만 실행

```shell
mkdir cni
cd cni

wget https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/custom-resources.yaml

# cidr 대역을 worker node 대역으로 변경: 10.18.0.0/24
vi custom-resources.yaml

kubectl create -f tigera-operator.yaml
kubectl create -f custom-resources.yaml
```

### 설치 후 명령어 실행 결과

```shell
root@sidev-k8s-master:~# kubectl get no
NAME                STATUS   ROLES           AGE   VERSION
sidev-k8s-master    Ready    control-plane   68m   v1.30.7
sidev-k8s-worker1   Ready    <none>          26m   v1.30.7
sidev-k8s-worker2   Ready    <none>          21m   v1.30.7

root@sidev-k8s-master:~/cni# kubectl get pods -A -n kube-system
NAMESPACE          NAME                                       READY   STATUS    RESTARTS   AGE
calico-apiserver   calico-apiserver-77dbd7fff6-d62gl          1/1     Running   0          22s
calico-apiserver   calico-apiserver-77dbd7fff6-z4rf6          1/1     Running   0          22s
calico-system      calico-kube-controllers-f664f9864-dj9mp    1/1     Running   0          87s
calico-system      calico-node-8q4hk                          1/1     Running   0          87s
calico-system      calico-typha-745c79d6b8-ddgz8              1/1     Running   0          87s
calico-system      csi-node-driver-g89cd                      2/2     Running   0          87s
kube-system        coredns-55cb58b774-b65rr                   1/1     Running   0          32m
kube-system        coredns-55cb58b774-jdjfn                   1/1     Running   0          32m
kube-system        etcd-sidev-k8s-master                      1/1     Running   0          32m
kube-system        kube-apiserver-sidev-k8s-master            1/1     Running   0          32m
kube-system        kube-controller-manager-sidev-k8s-master   1/1     Running   0          32m
kube-system        kube-proxy-t8tnf                           1/1     Running   0          32m
kube-system        kube-scheduler-sidev-k8s-master            1/1     Running   0          32m
tigera-operator    tigera-operator-77f994b5bb-tpjvq           1/1     Running   0          100s
```

## Azure Disk 연결

```shell
root@sidev-k8s-master:~# sudo lsblk
sda       8:0    0   30G  0 disk
├─sda1    8:1    0   29G  0 part /var/lib/kubelet/pods/beb1d5c4-3a9c-4213-ae8b-ffcd5f22ad64/volume-subpaths/tigera-ca-bundle/calico-node/1
│                                /
├─sda14   8:14   0    4M  0 part
├─sda15   8:15   0  106M  0 part /boot/efi
└─sda16 259:0    0  913M  0 part /boot
sdb       8:16   0    4G  0 disk
└─sdb1    8:17   0    4G  0 part /mnt
sdc       8:32   0  128G  0 disk
sr0      11:0    1  628K  0 rom
root@sidev-k8s-master:~# sudo parted /dev/sdc --script mklabel gpt mkpart xfspart xfs 0% 100%
root@sidev-k8s-master:~# sudo mkfs.xfs /dev/sdc1
meta-data=/dev/sdc1              isize=512    agcount=4, agsize=8388480 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=33553920, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.
root@sidev-k8s-master:~# sudo partprobe /dev/sdc1
root@sidev-k8s-master:~# sudo mkdir /data
root@sidev-k8s-master:~# sudo mount /dev/sdc1 /data
root@sidev-k8s-master:~# sudo blkid
/dev/sdb1: UUID="0fe8ba69-047f-406e-9433-14d99947f589" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="d7981506-01"
/dev/sda16: LABEL="BOOT" UUID="b852c0dd-fb6f-43c9-8de0-aac13b367377" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="8d53968d-f0a5-42e7-96ce-d5d3b2cfec93"
/dev/sda15: LABEL_FATBOOT="UEFI" LABEL="UEFI" UUID="9C78-5597" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="1e19f3ab-07d6-441b-a982-69ad274b5235"
/dev/sda1: LABEL="cloudimg-rootfs" UUID="90dbb72d-9869-4713-b6f0-9618fa7b481b" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="eb8d468b-b4cb-4ca0-9311-12cbb618369a"
/dev/sdc1: UUID="c17d3a7d-a166-48c4-b0bb-3721a2294bfd" BLOCK_SIZE="4096" TYPE="xfs" PARTLABEL="xfspart" PARTUUID="e2d433f2-ce3c-4edb-86ec-9aee241400c2"
/dev/sda14: PARTUUID="165b7d7e-2c98-4e73-a274-116a4189955a"
root@sidev-k8s-master:~# sudo vi /etc/fstab
UUID=c17d3a7d-a166-48c4-b0bb-3721a2294bfd       /data   xfs     defaults,nofail 1 2
root@sidev-k8s-master:~# lsblk -o NAME,HCTL,SIZE,MOUNTPOINT | grep -i "sd"
sda     0:0:0:0      30G
├─sda1               29G /
├─sda14               4M
├─sda15             106M /boot/efi
└─sda16             913M /boot
sdb     0:0:0:1       8G
└─sdb1                8G /mnt
sdc     1:0:0:0     128G
└─sdc1              128G /data
```