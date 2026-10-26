# Kubernetes 프로젝트 학습

## 2장 2교시 — kubeadm 기반 Kubernetes 3노드 클러스터 구축

### 1. 클러스터 아키텍처

**구축 환경**
- OS: Ubuntu 24.04 LTS
- 가상화: VMware Workstation
- Kubernetes 설치: kubeadm
- Container Runtime: containerd
- CNI: Flannel
- 구성: Control Plane 1대 + Worker Node 2대

```text
Kubernetes Cluster
│
├── Control Plane
│   ├── API Server
│   ├── etcd
│   ├── Scheduler
│   └── Controller Manager
│
├── Worker1
│   ├── kubelet
│   ├── containerd
│   └── Application Pods
│
└── Worker2
    ├── kubelet
    ├── containerd
    └── Application Pods
```

※ Control Plane이 1대이므로 Control Plane 자체의 고가용성은 보장하지 않는다.

### 2. Kubernetes 주요 도구

| 구성요소 | 역할 |
|---|---|
| kubeadm | 클러스터 초기화 및 Node 참여 |
| kubelet | 각 Node의 Pod 실행 상태 관리 |
| kubectl | API Server를 통한 클러스터 관리 |
| containerd | 실제 컨테이너 실행 |
| Flannel | Node 간 Pod 네트워크 구성 |

**핵심 차이**

- `kubeadm init`: Control Plane 초기화
- `kubeadm join`: Worker Node를 클러스터에 참여
- `kubelet`: Pod 실행 및 상태 관리
- `containerd`: 실제 컨테이너 실행

### 3. Linux 사전 설정

**① Swap 비활성화**

```bash
sudo swapoff -a
swapon --show
```

재부팅 후에도 비활성화하려면 `/etc/fstab`에서 Swap 항목을 주석 처리한다.

**이유:** 이번 실습에서는 kubelet의 기본 Swap 설정과 충돌하지 않도록 비활성화한다.

**② 커널 모듈**

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

- `overlay`: 컨테이너의 계층형 파일시스템 지원
- `br_netfilter`: 브리지 네트워크 트래픽의 Netfilter 처리 지원

**③ 네트워크 설정**

`/etc/sysctl.d/k8s.conf`

```ini
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

적용:

```bash
sudo sysctl --system
```

**핵심:** `ip_forward=1`은 서버가 다른 네트워크로 IPv4 패킷을 전달할 수 있도록 한다.

※ 커널 모듈 자동 로드 설정도 별도로 유지해야 한다.

### 4. containerd 설치 및 설정

```bash
sudo apt update
sudo apt install -y containerd

containerd --version
```

containerd 1.x 기본 설정 생성:

```bash
sudo mkdir -p /etc/containerd
containerd config default > ~/containerd-default.toml
sudo cp ~/containerd-default.toml /etc/containerd/config.toml
```

설정 파일:

```text
/etc/containerd/config.toml
```

containerd 1.x의 해당 옵션:

```toml
SystemdCgroup = true
```

설정 적용:

```bash
sudo systemctl enable --now containerd
sudo systemctl restart containerd
sudo systemctl is-active containerd
```

**이유:** kubelet과 containerd가 systemd 기반 cgroup 관리를 일관되게 사용하도록 설정한다.

※ containerd 2.x는 설정 경로가 다르므로 설치된 버전에 맞게 수정한다.

### 5. Kubernetes 설치

서버 3대에 Kubernetes 공식 1.37 패키지 저장소를 등록한 후 설치한다.

```bash
sudo apt install -y kubeadm kubelet kubectl

sudo apt-mark hold kubelet kubeadm kubectl
```

- `kubeadm`: 클러스터 구축
- `kubelet`: Node 에이전트
- `kubectl`: 관리 CLI
- `apt-mark hold`: 계획하지 않은 자동 패키지 업그레이드 방지

※ 위 명령어 실행 전 Kubernetes APT 저장소 등록이 필요하다.

### 6. Control Plane 초기화

**Control Plane 서버에서만 실행**

```bash
sudo kubeadm init \
  --apiserver-advertise-address=<Control-Plane-IP> \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

| 옵션 | 의미 |
|---|---|
| `--apiserver-advertise-address` | API Server가 알릴 Control Plane 주소 |
| `--pod-network-cidr` | Pod에 사용할 IP 대역 |
| `--cri-socket` | containerd 연결 소켓 |

kubectl 관리자 설정:

```bash
mkdir -p ~/.kube

sudo cp /etc/kubernetes/admin.conf ~/.kube/config

sudo chown $(id -u):$(id -g) ~/.kube/config
```

### 7. Flannel 설치

**Flannel이란?**

서로 다른 Worker Node의 Pod들이 통신할 수 있도록 네트워크를 구성하는 CNI 플러그인이다.

```bash
curl -fL \
  https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml \
  -o kube-flannel.yml

kubectl apply -f kube-flannel.yml
```

상태 확인:

```bash
kubectl get pods -n kube-flannel
```

**네트워크 구분**

- `192.168.111.0/24`: VMware 서버 네트워크
- `10.244.0.0/16`: Kubernetes Pod 네트워크

### 8. Worker Node 참여

**Control Plane에서 Join 명령 생성**

```bash
sudo kubeadm token create --print-join-command
```

**Worker1·Worker2에서 실행**

출력된 `kubeadm join` 명령어를 각각 `sudo`로 실행한다.

Join 이후 Control Plane에서 확인:

```bash
kubectl get nodes -o wide
```

정상 목표:

```text
NAME            STATUS
control-plane   Ready
worker1         Ready
worker2         Ready
```

### 9. Pod 배포 및 검증

```bash
# Nginx Deployment 생성
kubectl create deployment demo-nginx \
  --image=nginx:alpine \
  --replicas=3

# 배포 상태 확인
kubectl rollout status deployment/demo-nginx

# Pod가 배치된 Node 확인
kubectl get pods -o wide

# Pod 상태 및 이벤트 확인
kubectl describe pod <pod-name>

# 실습 Deployment 정리
kubectl delete deployment demo-nginx
```

**Pod 생성 원리**

```text
kubectl
   ↓
API Server
   ↓
etcd (상태 저장)
   ↓
Controller (원하는 Pod 수 관리)
   ↓
Scheduler (Node 선택)
   ↓
kubelet (Pod 실행 관리)
   ↓
containerd (컨테이너 실행)
```

※ 실제 구성요소는 API Server를 통해 상태를 확인하고 변경한다.

### 10. Worker Node NotReady 장애 대응

**장애 점검 순서**

1. Node 상태 확인
2. Node의 Conditions와 Events 확인
3. 서버 및 네트워크 통신 확인
4. kubelet과 containerd 상태 확인
5. CNI, CPU, 메모리, 디스크 점검
6. 원인 해결 후 Ready 상태 재검증

**Control Plane에서 실행**

```bash
kubectl get nodes -o wide
kubectl describe node worker1
kubectl get pods -A -o wide
```

**장애 Worker에서 실행**

```bash
sudo systemctl status kubelet
sudo systemctl status containerd

sudo journalctl -u kubelet -n 50 --no-pager
sudo journalctl -u containerd -n 50 --no-pager

free -h
df -h
ip route
```

**중요한 구분**

- Node 장애 → `kubectl get nodes`, `kubectl describe node`
- Pod 장애 → `kubectl get pods`, `kubectl describe pod`
- 애플리케이션 로그 → `kubectl logs <pod-name>`

### 11. 핵심 면접 질문

**Q1. kubeadm과 kubelet의 차이는?**

kubeadm은 Kubernetes 클러스터를 초기화하고 Node를 참여시키는 도구이다. kubelet은 각 Node에서 Pod의 실행 상태를 지속적으로 관리하는 에이전트이다.

**Q2. containerd는 무엇인가?**

Kubernetes에서 kubelet의 요청을 받아 컨테이너를 실제로 생성하고 실행하는 컨테이너 런타임이다.

**Q3. Flannel은 왜 필요한가?**

서로 다른 Node에 배치된 Pod들이 네트워크를 통해 통신할 수 있도록 하기 위해서이다.

**Q4. Worker Node가 NotReady라면?**

먼저 Node 상태와 상세 이벤트를 확인하고, 서버 네트워크 및 kubelet·containerd 상태를 점검한다. 이후 CNI와 시스템 자원 문제를 확인하고 복구 여부를 검증한다.

**Q5. kubeadm init과 join의 차이는?**

`kubeadm init`은 Control Plane을 초기화하고, `kubeadm join`은 새로운 Node를 기존 클러스터에 연결한다.

---

### 최종 핵심 요약

- **kubeadm:** 클러스터 구축·참여
- **kubelet:** Pod 상태 관리
- **containerd:** 컨테이너 실행
- **kubectl:** Kubernetes 관리
- **Flannel:** Pod 네트워크
- **Control Plane:** 클러스터 관리
- **Worker Node:** 애플리케이션 실행
- **Ready:** Node가 Kubernetes에서 정상적으로 사용 가능한 상태

**2장 2교시 완료 기준:** Control Plane 1대와 Worker Node 2대가 모두 Ready이고, 테스트 Pod 배포 및 노드 배치가 정상적으로 검증된 상태.
