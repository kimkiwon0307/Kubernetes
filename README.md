<details>
  <summary>
    <h2>1장 1교시 - 설치 환경과 사전 점검</h2>
  </summary>
  



## 1. Kubernetes 설치 방식

Kubernetes 환경은 크게 다음과 같이 구분할 수 있다.

- **로컬 개발 환경**: Minikube, Docker Desktop
- **직접 구축**: kubeadm
- **Managed Kubernetes**: AWS EKS, GKE, AKS

이번 학습에서는 Kubernetes 내부 구조와 설치 과정을 이해하기 위해 **VMware + Ubuntu + kubeadm** 환경을 사용한다.

---

## 2. 실습 환경

| Node | 역할 | IP | vCPU | RAM | Disk |
|---|---|---|---:|---:|---:|
| control-plane | Control Plane | 192.168.111.150 | 2 | 4GB | 40GB |
| worker1 | Worker | 192.168.111.152 | 2 | 4GB | 30GB |
| worker2 | Worker | 192.168.111.151 | 2 | 4GB | 30GB |

OS:

```text
Ubuntu 24.04 LTS
```

구조:

```text
control-plane
├─ API Server
├─ Scheduler
├─ Controller Manager
└─ etcd

worker1
├─ kubelet
└─ container runtime

worker2
├─ kubelet
└─ container runtime
```

---

## 3. Control Plane과 Worker

### Control Plane

Kubernetes Cluster 전체를 관리한다.

주요 구성요소:

```text
API Server
Scheduler
Controller Manager
etcd
```

### Worker Node

실제 애플리케이션 Pod가 실행되는 서버이다.

주요 구성요소:

```text
kubelet
container runtime
kube-proxy
```

---

## 4. 서버 기본 점검

### Hostname

```bash
hostnamectl
```

Node를 구분하기 위한 서버 이름을 확인한다.

사용할 이름:

```text
control-plane
worker1
worker2
```

---

### OS 확인

```bash
cat /etc/os-release
```

Ubuntu 버전을 확인한다.

---

### CPU 확인

```bash
nproc
```

사용 가능한 CPU 개수를 확인한다.

---

### Memory 확인

```bash
free -h
```

메모리와 Swap 사용량을 확인한다.

---

### Disk 확인

```bash
lsblk
```

Disk, Partition, LVM 구조를 확인한다.

---

### IP 확인

```bash
ip -br addr
```

Network Interface와 IP 주소를 확인한다.

---

### Routing 확인

```bash
ip route
```

기본 Gateway와 Routing Table을 확인한다.

---

### Swap 확인

```bash
swapon --show
```

현재 Swap 활성화 여부를 확인한다.

현재 세 Node 모두 Swap이 활성화되어 있으며 다음 교시에서 Kubernetes 환경에 맞게 설정한다.

---

## 5. 네트워크 점검

Node 간 통신 확인:

```bash
ping -c 3 <다른 Node IP>
```

인터넷 연결 확인:

```bash
ping -c 3 8.8.8.8
```

DNS 확인:

```bash
ping -c 3 google.com
```

판단 예:

```text
8.8.8.8 성공
google.com 성공
→ 네트워크 + DNS 정상
```

```text
8.8.8.8 성공
google.com 실패
→ DNS 문제 가능성
```

---

## 6. 현재 환경 상태

```text
Ubuntu 24.04        ✅
CPU 2 vCPU 이상     ✅
RAM 약 4GB          ✅
Disk 30GB 이상      ✅
Node IP 확인         ✅
Routing 확인         ✅
Swap 활성 상태 확인  ✅
```

추가 확인 사항:

- Node 간 통신
- 인터넷 통신
- LVM 여유 공간

---

## 7. 핵심 정리

```text
Minikube
= 개발/테스트용

kubeadm
= Kubernetes Cluster 직접 구축

EKS / GKE / AKS
= Cloud Managed Kubernetes
```

이번 학습 환경:

```text
VMware
 ↓
Ubuntu 24.04 × 3
 ↓
kubeadm
 ↓
Control Plane 1대
Worker 2대
```

### 다음 교시

```text
Swap 설정
→ Kernel Module
→ sysctl
→ containerd
→ CRI / cgroup 이해
```

</details>
<details>
  <summary>
    <h2> 1장 2교시 - 설치 전 사전 설정과 containerd</h2>
  </summary>


## 1. 학습 목표

Kubernetes Node에서 kubelet과 Container Runtime이 정상 동작하도록 Linux 환경을 준비한다.

핵심 흐름:

```text
Swap 비활성화
→ Kernel Module 준비
→ Network sysctl 설정
→ containerd 설치
→ CRI 확인
→ systemd cgroup 설정
```

---

# 2. Swap 비활성화

현재 kubelet 기본 설정에서는 Swap이 활성화되어 있으면 문제가 발생할 수 있으므로 학습 환경에서는 비활성화한다.

현재 Swap 끄기:

```bash
sudo swapoff -a
```

확인:

```bash
swapon --show
```

재부팅 후 다시 활성화되지 않도록 `/etc/fstab`의 Swap 항목도 비활성화한다.

핵심:

```text
Swap
→ kubelet 실행 및 메모리 관리와 관련
```

---

# 3. Kernel Module 준비

사용한 Module:

```text
overlay
br_netfilter
```

설정:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

즉시 적용:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

확인:

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

역할:

```text
overlay
→ Container Image Layer/FileSystem 처리

br_netfilter
→ Bridge를 지나는 Pod Network Traffic 처리
```

---

# 4. Network Kernel 설정

Kubernetes에서 Node가 Pod Traffic을 전달할 수 있도록 설정한다.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

적용:

```bash
sudo sysctl --system
```

확인:

```bash
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

정상 값:

```text
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
```

핵심:

```text
ip_forward
→ Node가 Packet을 다른 Network로 전달

bridge-nf-call-iptables
→ Bridge Traffic이 iptables/netfilter 처리를 받도록 설정
```

---

# 5. containerd 설치

Kubernetes는 Container를 직접 실행하지 않는다.

구조:

```text
Kubernetes
 ↓
kubelet
 ↓
CRI
 ↓
containerd
 ↓
runc
 ↓
Linux Kernel
```

설치:

```bash
sudo apt update
sudo apt install -y containerd
```

버전 확인:

```bash
containerd --version
```

실습 환경:

```text
containerd 2.2.1
```

Service 확인:

```bash
sudo systemctl status containerd
```

---

# 6. CRI

CRI는 `Container Runtime Interface`의 약자다.

```text
kubelet
 ↕
CRI
 ↕
containerd
```

즉 kubelet과 Container Runtime 사이의 표준 인터페이스이다.

CRI Plugin 확인:

```bash
sudo ctr plugins ls | grep cri
```

---

# 7. cgroup

cgroup은 Linux에서 Process의 CPU, Memory 등의 자원을 관리하는 기능이다.

구조:

```text
Kubernetes
 ↓
kubelet
 ↓
containerd
 ↓
cgroup
 ↓
Linux Kernel
```

나중에 사용하는:

```yaml
resources:
  requests:
  limits:
```

도 결국 Linux cgroup과 연결된다.

---

# 8. systemd cgroup 설정

kubelet과 containerd의 cgroup driver를 맞추기 위해 `systemd` 방식을 사용한다.

containerd 설정 생성:

```bash
sudo mkdir -p /etc/containerd

containerd config default | \
sudo tee /etc/containerd/config.toml > /dev/null
```

확인:

```bash
grep -n "SystemdCgroup" /etc/containerd/config.toml
```

설정:

```text
SystemdCgroup = true
```

변경 후:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

확인:

```bash
sudo systemctl is-active containerd
```

정상:

```text
active
```

---

# 9. containerd 2.x

현재 환경:

```text
containerd 2.2.1
```

containerd 2.x에서는 설정 형식이 1.x와 다를 수 있으므로 인터넷의 오래된 설치 문서를 그대로 따라가지 않고 현재 버전에 맞는 설정을 확인해야 한다.

핵심:

```text
버전 확인
→ 설정 파일 확인
→ SystemdCgroup 확인
→ CRI Plugin 확인
```

---

# 10. 이번 교시 핵심 구조

```text
Swap
→ kubelet 동작 조건

overlay
→ Container Filesystem

br_netfilter
→ Bridge Network 처리

ip_forward
→ Packet Forwarding

containerd
→ Container Runtime

CRI
→ kubelet과 Runtime 사이 인터페이스

cgroup
→ CPU / Memory 자원 관리

systemd cgroup
→ kubelet과 containerd의 자원 관리 방식 통일
```

---

# 11. 검증 명령어

```bash
swapon --show
```

→ 출력 없음

```bash
sysctl net.ipv4.ip_forward
```

→ `1`

```bash
sysctl net.bridge.bridge-nf-call-iptables
```

→ `1`

```bash
containerd --version
```

→ containerd 버전 확인

```bash
grep -n "SystemdCgroup" /etc/containerd/config.toml
```

→ `true`

```bash
sudo systemctl is-active containerd
```

→ `active`

---

# 2교시 핵심 한 줄

```text
Kubernetes를 설치하기 전에
Linux Network / Container Runtime / cgroup 환경을 먼저 준비한다.
```

### 다음 교시

```text
kubeadm
kubelet
kubectl
설치
→ Control Plane 초기화
```
</details>
<details>
  <summary>
    <h2>1장 3교시 - kubeadm 설치와 Control Plane 초기화</h2>
    
  </summary>

# Kubernetes 

## 1. 핵심 도구

```text
kubeadm
= Kubernetes Cluster 초기 구축 도구

kubelet
= 각 Node에서 Pod 실행을 관리하는 Agent

kubectl
= 사용자가 Kubernetes API Server에 명령을 보내는 CLI
```

구조:

```text
사용자
 ↓
kubectl
 ↓
API Server

Node
 ↓
kubelet
 ↓
CRI
 ↓
containerd
 ↓
Container
```

---

# 2. Kubernetes 패키지 설치

세 Node 모두:

```bash
sudo apt-get update

sudo apt-get install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gpg
```

Kubernetes Repository 등록 후:

```bash
sudo apt-get install -y kubelet kubeadm kubectl
```

버전 고정:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

목적:

```text
일반 apt upgrade로
Kubernetes 버전이 임의 변경되는 것을 방지
```

---

# 3. Control Plane 초기화

Control Plane에서만 실행:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.111.150 \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

핵심 옵션:

```text
--apiserver-advertise-address
= API Server가 사용할 Node IP

--pod-network-cidr
= Pod가 사용할 Network 대역

--cri-socket
= 사용할 Container Runtime의 CRI Socket
```

---

# 4. kubeadm init이 하는 일

```text
Preflight Check
 ↓
인증서 생성
 ↓
kubeconfig 생성
 ↓
etcd 구성
 ↓
API Server 구성
 ↓
Controller Manager 구성
 ↓
Scheduler 구성
 ↓
kubelet 설정
 ↓
Worker Join용 Token 생성
```

---

# 5. Static Pod

Control Plane 핵심 Component들은 Static Pod 형태로 실행된다.

Manifest 위치:

```bash
sudo ls /etc/kubernetes/manifests/
```

확인한 파일:

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

동작 구조:

```text
/etc/kubernetes/manifests
 ↓
kubelet이 감시
 ↓
containerd에 실행 요청
 ↓
Control Plane Component 실행
```

핵심:

```text
kubeadm
= 처음 구성

kubelet
= 계속 감시하고 실행 상태 유지
```

---

# 6. kubectl 설정

관리자 kubeconfig:

```text
/etc/kubernetes/admin.conf
```

일반 사용자에서 kubectl을 사용하기 위해:

```bash
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf \
  $HOME/.kube/config

sudo chown $(id -u):$(id -g) \
  $HOME/.kube/config
```

`kubeconfig`는 kubectl이 Cluster에 접속하기 위한 정보 파일이다.

```text
Cluster 주소
인증 정보
Context
사용자 정보
```

등이 들어 있다.

---

# 7. 현재 Cluster 상태

확인:

```bash
kubectl get nodes
```

결과:

```text
control-plane   NotReady
```

이 상태는 현재 정상이다.

이유:

```text
CNI 미설치
 ↓
Pod Network 없음
 ↓
Node NotReady
```

---

# 8. kube-system 확인

```bash
kubectl get pods -n kube-system -o wide
```

정상 실행:

```text
etcd
kube-apiserver
kube-controller-manager
kube-scheduler
kube-proxy
```

현재 Pending:

```text
CoreDNS
```

이유:

```text
CNI 없음
 ↓
Pod Network 없음
 ↓
CoreDNS Pending
```

---

# 9. 현재 상태 요약

```text
Control Plane 구성      ✅
API Server              ✅
etcd                    ✅
Scheduler               ✅
Controller Manager      ✅
kubelet                 ✅
containerd              ✅
kubectl 연결            ✅

CNI                     ❌
Worker Join             ❌
```

현재:

```text
Control Plane
   ↓
NotReady
```

다음 교시에서:

```text
Flannel CNI 설치
 ↓
Pod Network 생성
 ↓
CoreDNS Running
 ↓
Node Ready
 ↓
Worker Join
```

으로 이어진다.

---

# 10. 핵심 명령어

```bash
kubeadm version
kubelet --version
kubectl version --client
```

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods -n kube-system -o wide
```

```bash
sudo ls /etc/kubernetes/manifests/
```

---

# 3교시 핵심 한 줄

```text
kubeadm init으로 Control Plane을 구성하고,
kubelet이 Static Pod 형태의 핵심 Component를 실행·유지한다.
```

현재 `NotReady`와 `CoreDNS Pending`은 **CNI가 아직 없기 때문에 정상적인 중간 상태**이다.

### 다음 교시

```text
Flannel CNI 설치
→ CoreDNS Running
→ Node Ready
→ worker1 / worker2 Join
→ 3-Node Cluster 완성
```
  
</details>




<details>

  <summary>
    <h2>
      4교시 - CNI 설치와 Worker Join  
    </h2>
    
  </summary>
## 1. 학습 목표

이번 교시에서는 Kubernetes Cluster에 Pod Network를 구성하고 Worker Node를 Cluster에 Join했다.

최종 목표:

```text
control-plane   Ready
worker1         Ready
worker2         Ready
```

---

# 2. CNI란?

CNI는 `Container Network Interface`의 약자다.

Kubernetes Pod들이 서로 통신할 수 있도록 Pod Network를 구성한다.

```text
Pod A
 ↓
CNI
 ↓
Pod Network
 ↓
Pod B
```

이번 실습에서는 `Flannel`을 사용했다.

---

# 3. Flannel 설치

Control Plane에서:

```bash
kubectl apply -f \
https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

확인:

```bash
kubectl get pods -n kube-flannel -o wide
```

Flannel은 DaemonSet 방식으로 각 Node에 하나씩 실행된다.

```text
control-plane → Flannel Pod
worker1       → Flannel Pod
worker2       → Flannel Pod
```

---

# 4. CNI 설치 전/후 변화

CNI 설치 전:

```text
Node      → NotReady
CoreDNS   → Pending
```

Flannel 설치 후:

```text
Pod Network 생성
 ↓
CoreDNS Running
 ↓
Node Ready
```

실제 CoreDNS Pod IP도 `10.244.x.x` 대역을 사용했다.

---

# 5. Worker Join

Control Plane에서 Join 명령 확인:

```bash
kubeadm token create --print-join-command
```

Worker에서 실행:

```bash
sudo kubeadm join 192.168.111.150:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

개념:

```text
kubeadm init
= Cluster 생성

kubeadm join
= 기존 Cluster에 Node 가입
```

---

# 6. Join 과정

```text
Worker
 ↓
API Server :6443 접속
 ↓
Bootstrap Token 인증
 ↓
Control Plane 확인
 ↓
kubelet 설정
 ↓
Node 등록
```

---

# 7. Node 상태 확인

Control Plane에서:

```bash
kubectl get nodes -o wide
```

최종 결과:

```text
control-plane   Ready
worker1         Ready
worker2         Ready
```

환경:

```text
control-plane   192.168.111.150
worker1         192.168.111.152
worker2         192.168.111.151
```

모든 Node에서:

```text
Kubernetes   v1.37.1
containerd   2.2.1
```

을 사용했다.

---

# 8. kube-system 확인

```bash
kubectl get pods -n kube-system -o wide
```

주요 Component:

```text
CoreDNS
etcd
kube-apiserver
kube-controller-manager
kube-scheduler
kube-proxy
```

모두 정상적으로 `Running` 상태임을 확인했다.

`kube-proxy`는 각 Node마다 실행된다.

```text
control-plane → kube-proxy
worker1       → kube-proxy
worker2       → kube-proxy
```

---

# 9. Flannel 확인

```bash
kubectl get pods -n kube-flannel -o wide
```

각 Node마다 Flannel Pod가 하나씩 실행된다.

```text
Node 3대
 ↓
Flannel Pod 3개
```

이 구조는 DaemonSet의 대표적인 사용 예이다.

---

# 10. Node Network와 Pod Network

## Node Network

VM 자체 IP:

```text
control-plane   192.168.111.150
worker1         192.168.111.152
worker2         192.168.111.151
```

## Pod Network

```text
10.244.0.0/16
```

구분:

```text
192.168.111.x
= Node 간 통신

10.244.x.x
= Pod 간 통신
```

---

# 11. Worker에서 kubectl 오류

Worker에서 다음 명령을 실행했을 때:

```bash
kubectl get pods
```

다음 오류가 발생했다.

```text
http://localhost:8080
connect: connection refused
```

원인:

```text
Worker 일반 사용자에게
kubeconfig가 없음
 ↓
kubectl이 API Server 위치를 모름
```

Cluster 장애가 아니라 `kubectl` 접속정보 문제였다.

현재 실습에서는 관리 명령을 Control Plane에서 실행한다.

---

# 12. 장애 확인 순서

Node가 `NotReady`인 경우:

```text
kubectl get nodes
 ↓
kubectl describe node <node>
 ↓
CNI Pod 확인
 ↓
Events / Logs 확인
```

Flannel 확인:

```bash
kubectl get pods -n kube-flannel -o wide
```

Worker Join 문제 시:

```text
Control Plane IP 확인
API Server 6443 통신 확인
Token 확인
containerd 확인
kubelet 확인
Swap 확인
```

---

# 13. 최종 Cluster 구조

```text
                 control-plane
                192.168.111.150
                       │
                  API Server
                       │
          ┌────────────┴────────────┐
          │                         │
       worker1                   worker2
  192.168.111.152          192.168.111.151
          │                         │
      Flannel CNI               Flannel CNI
          │                         │
        Pod                       Pod
```

---

# 14. 1장 전체 흐름

```text
1교시
VM / OS / CPU / RAM / Network 점검

2교시
Swap / Kernel / sysctl / containerd / CRI / cgroup

3교시
kubeadm / kubelet / kubectl
→ Control Plane 초기화

4교시
Flannel CNI
→ Worker Join
→ 3-Node Cluster 완성
```

---

# 4교시 핵심 한 줄

```text
CNI로 Pod Network를 구성하고,
Worker Node를 Join하여 3-Node Kubernetes Cluster를 완성했다.
```

# 1장 완료

```text
control-plane   Ready
worker1         Ready
worker2         Ready
```

  
</details>









