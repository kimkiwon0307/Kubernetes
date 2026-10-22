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
    <h2> Kubernetes 1장 2교시 - 설치 전 사전 설정과 containerd</h2>
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
