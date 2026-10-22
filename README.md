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
