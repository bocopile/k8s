# Baremetal Server Setup for Kubernetes

## System Spec

OS : Ubuntu 22.04.4
CPU Arch: x86_64 (arm64)
CPU: 20 Core / Unknown
RAM: 64GB

## Caution

Do not try this on Cloud Infrastructure.
If your subnet CIDR set to 10.0.0.0/16, internal ip will duplicated and network will not work.

```bash
# -------------------
# Section for Installing Docker
sudo apt-get update
# ca-certificates : 인증기관 CA에서 발급한 인증서를 포함한 패키지 => Https통신 목적
# curl : http, https를 이용하여 데이터를 전송하는 도구
# gnupg : GnuPG(GNU Privacy Guard) : 데이터를 암호화와 서명에 사용되는 도구
# lsb-release : 현재 시스템의 배포판 정보를 확인하는 명령어
sudo apt-get install ca-certificates curl gnupg lsb-release

# Used for Debugging Networks
sudo apt-get install net-tools

# Docker 공식 GPG 키를 다운로들하고 gpg 독구를 사용하여 지정된 위치에 저장되는 수행 
sudo mkdir -p /etc/apt/keyrings
# -f : http 요청 실패시 오류 메세지를 미출력
# -s : 진행상황이나 오류 메세지를 출력하지 않고 조용히 실행
# -S : d오류 발생시 오류 메세지를 출력
# -L : 요청한 URL이 리다이렉션이 되는 경우 새로운 위치를 따라가서 요청
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Docker의 APT 저장소를 Ubuntu 시스템의 패키지 소스 목록에 추가하는 명령어
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

# Ubuntu 시스템에 특정 버전의 docker 패키지를 세팅
sudo apt-get install docker-ce=5:20.10.22~3-0~ubuntu-jammy docker-ce-cli=5:20.10.22~3-0~ubuntu-jammy docker-compose-plugin=2.14.1~ubuntu-jammy
# Docker 버전이 자동으로 업데이트 되지 않도록 세팅
sudo apt-mark hold docker-ce docker-ce-cli docker-compose-plugin containerd.io
# -------------------

# -------------------
# Section for Installing containerd and runc
# containerd : 컨테이너를 실행하고 노드에서 이미지를 관리하는데 필요한 최소한의 기능 세티를 제공하는 OCI 호환 코어 컨테이너 런타임중 하나
wget https://github.com/containerd/containerd/releases/download/v1.6.14/containerd-1.6.14-linux-arm64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.6.14-linux-arm64.tar.gz

wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mv containerd.service /usr/lib/systemd/system/

sudo systemctl daemon-reload
sudo systemctl enable --now containerd

sudo systemctl status containerd

# runc : Open Container Initiative (OCI)에서 정의한 컨테이너 실행 스펙에 따라 컨테이너를 생성하고 실행하는데 사용됩니다. runc는 컨테이너의 생명 주기 동안 PID 1로 작동하며, 컨테이너의 프로세스를 생성하고 격리된 환경에서 실행하는 역할
wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.arm64
sudo install -m 755 runc.arm64 /usr/local/sbin/runc

# containerd 설정 파일을 생성하고 수정하여 Systemd cgroup 드라이버를 사용하도록 설정하는 것입니다.
sudo mkdir -p /etc/containerd/
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
# -------------------

# -------------------
# Section for Installing CNI
sudo mkdir -p /opt/cni/bin/
sudo wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-arm64-v1.1.1.tgz
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-arm64-v1.1.1.tgz
# -------------------

# -------------------
# Section for Initializing Docker
sudo usermod -aG docker <user_name>
id <user_name>

sudo service docker start

sudo mkdir -p /container
sudo chown -R $(id -u):$(id -g) /container

sudo vi /etc/docker/daemon.json
{
    "data-root": "/container/docker",
    "exec-opts": ["native.cgroupdriver=systemd"]
}

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker

sudo ls -al /container
sudo ls -al /container/docker
# -------------------

# -------------------
# Section for Initializing System
sudo hostnamectl set-hostname <new_hostname>
hostname

192.168.0.151 k8s-master
192.168.0.152 k8s-worker01
192.168.0.153 k8s-worker02
192.168.0.154 k8s-worker03

sudo vi /etc/hosts
<public_ip> <host_name> <domain>

# Linux 방화벽 해제
sudo ufw disable
sudo ufw status

# docker 에서는 swap 메모리를 해제 한다.
# https://velog.io/@yange/kubernetes-swap-off%EC%97%90-%EB%8C%80%ED%95%98%EC%97%AC
sudo swapoff -a
# swap storage 관련하여 주석 처리
sudo vi /etc/fstab
# 주석 처리하면서 기존 swapfile의 위치 확인
sudo rm -f <swapfile_direction>

# swap 메모리가 0인지 확인
free
# -------------------

# -------------------
# Section for Initializing Network
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# kernel component
# overlay : 컨테이너를 효율적으로 관리하는데 사용
# br_netfilter 브릿지 네트워크를 통해 필터링을 가능하도록 한다.
sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl 설정 파일 생성
# net.bridge.bridge-nf-call-iptables : 브릿지된 네트워크를 통해 전달 되는 트래픽에 대해 iptables 필터링을 활성화
# net.bridge.bridge-nf-call-ip6tables : 브릿지된 네트워크를 통해 전달되는 IPv6 트래픽에 대해 iptables 필터링을 활성화
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1 
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# sysctl 설정 적용
sudo sysctl --system
# -------------------

# -------------------
# Section for Installing Kubernetes Binary
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
# kubectl : k8s 클러스터를 관리하기 위한 커멘드라인 인터페이스 (CLI) 도구 => 클러스터 내의 리소스를 생성, 수정,삭제 등의 관리
# kubeadm : k8s 클러스터를 부트스트랩(시작) 하는데 사용
# kubelet : k8s 노드에서 실행되는 에이전트 => 관리자가 정의한 Pod 및 컨테이너의 상태를 유지하고 관리
sudo apt-get install kubectl kubeadm kubelet

sudo apt-mark hold kubectl kubeadm kubelet kubernetes-cni

cat <<EOF | sudo tee /etc/default/kubelet
KUBELET_EXTRA_ARGS=--root-dir="/container/k8s"
EOF

sudo systemctl daemon-reload
sudo systemctl enable kubelet && sudo systemctl start kubelet
# -------------------

# 사용하지 않는 plugin 설정 제거
sudo vi /etc/containerd/config.toml
##### comment disabled plugin #####

# -------------------
# Section for Initializing Kubernetes Cluster
CERTKEY=$(kubeadm certs certificate-key)
sudo kubeadm init --pod-network-cidr=10.32.0.0/12 --control-plane-endpoint=192.168.64.2 --apiserver-cert-extra-sans=192.168.64.2,192.168.35.77 --upload-certs --certificate-key=$CERTKEY

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Only for Single Node Setup
kubectl get node
kubectl taint node <node_name> node-role.kubernetes.io/control-plane:NoSchedule-
# -------------------

# -------------------
# (Only if error) Reset Cluster
sudo kubeadm reset
sudo rm -rf /etc/kubernetes
rm -rf ~/.kube

sudo rm -rf /var/lib/etcd
sudo rm -rf /var/lib/cni
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
# -------------------

# -------------------
# Section for Setup Cluster Network and Test
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=arm64

curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Your Kernel Version Should be >= 4.9.17
uname -r

sudo apt update & sudo apt install resolvconf

sudo service resolvconf restart

cilium install

cilium status --wait

# 네트워크 테스트 여부 확인
cilium connectivity test
# -------------------

# -------------------
# Section for Post Setup
# 1. You need to install ingress controller unless you setup reverse proxy by yourself
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
sudo mv /usr/local/bin/helm /usr/bin

helm repo add stable https://charts.helm.sh/stable
helm repo list

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm repo list

helm show values ingress-nginx/ingress-nginx > ingress-nginx.yaml
chmod 777 ingress-nginx.yaml

# -- update ingress-nginx.ymal
vi ingress-nginx.yaml
hostNetwork: false
hostPort:
  enabled: true
kind: DaemonSet
externalIPS:
  - <public_ip>
loadBalancerSourceRanges:
  - <public_ip>/32

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --create-namespace --namespace ingress-nginx -f ingress-nginx.yaml
```
