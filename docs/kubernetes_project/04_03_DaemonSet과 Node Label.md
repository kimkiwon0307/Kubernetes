# Kubernetes 프로젝트 학습

## 4장 3교시 — DaemonSet과 Node Label

### 1. DaemonSet이란?

**DaemonSet은 조건에 맞는 각 Node에 Pod를 하나씩 실행하고 유지하는 Kubernetes 리소스이다.**

**필요한 이유**
- 서버마다 로그 수집 에이전트를 실행해야 하는 경우
- 서버마다 CPU·메모리 모니터링 에이전트가 필요한 경우
- 각 Node에서 네트워크 관련 구성요소를 실행해야 하는 경우

**대표적인 활용**
- Filebeat: 로그 수집
- Prometheus Node Exporter: 서버 메트릭 수집
- Flannel: Kubernetes Pod 네트워크 구성

### 2. Deployment와 DaemonSet의 차이

| 구분 | Deployment | DaemonSet |
|---|---|---|
| 목적 | 애플리케이션 배포 및 복제 | Node별 Pod 실행 |
| Pod 개수 | replicas로 지정 | 대상 Node 수에 따라 결정 |
| Node 추가 | Pod 개수가 자동으로 증가하지 않음 | 조건 충족 시 Pod 자동 추가 |
| 사용 예 | Spring Boot, Nginx | Filebeat, Node Exporter |

**핵심**

- Deployment: Pod를 총 몇 개 유지할 것인가?
- DaemonSet: 어떤 Node마다 Pod를 실행할 것인가?

### 3. Node Label과 nodeSelector

**Label**

Kubernetes 리소스를 구분하기 위해 붙이는 이름표이다.

```bash
kubectl label node worker1 lab.example.com/agent=enabled
kubectl label node worker2 lab.example.com/agent=enabled
```

**nodeSelector**

특정 라벨이 있는 Node에만 Pod를 배치하도록 제한하는 설정이다.

```yaml
nodeSelector:
  lab.example.com/agent: "enabled"
```

**selector.matchLabels와 nodeSelector 차이**

| 항목 | 선택 대상 |
|---|---|
| selector.matchLabels | 컨트롤러가 관리할 Pod |
| nodeSelector | Pod를 실행할 Node |

**주의:** DaemonSet은 반드시 Node Label을 설정해야만 동작하는 것은 아니다. 별도의 Node 선택 조건이 없다면 배치 조건을 충족하는 모든 Node가 대상이 될 수 있다.

### 4. DaemonSet 동작 원리

```text
DaemonSet 생성
      ↓
DaemonSet Controller가 대상 Node 확인
      ↓
Worker1 → Pod 생성
Worker2 → Pod 생성
      ↓
Scheduler가 각 Pod 배치
      ↓
kubelet이 실행 관리
      ↓
containerd가 컨테이너 실행
```

**Node 변화에 따른 동작**

- 대상 Node가 추가되면 해당 Node에 Pod 자동 생성
- Node가 대상에서 제외되면 해당 DaemonSet Pod 정리
- 해당 DaemonSet Pod가 삭제되면 대상 Node에 새 Pod 생성

### 5. 핵심 YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: study-node-agent
spec:
  selector:
    matchLabels:
      app: study-node-agent

  template:
    metadata:
      labels:
        app: study-node-agent
    spec:
      nodeSelector:
        lab.example.com/agent: "enabled"

      containers:
        - name: agent
          image: busybox:1.36
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                echo "Agent running"
                sleep 15
              done
```

**핵심 설정**

- `kind: DaemonSet`: DaemonSet 생성
- `selector.matchLabels`: 관리 대상 Pod 선택
- `template`: Pod 생성에 사용할 설계도
- `nodeSelector`: 배포 대상 Node 선택
- `containers`: 실행할 컨테이너 정의

### 6. 핵심 실습 명령어

**① Node 라벨 추가 및 확인**

```bash
kubectl label node worker1 lab.example.com/agent=enabled
kubectl label node worker2 lab.example.com/agent=enabled

kubectl get nodes -L lab.example.com/agent
```

**② DaemonSet 생성 및 검증**

```bash
kubectl apply -f node-agent.yaml

kubectl get daemonsets
kubectl get pods -o wide
```

정상 목표:

```text
Worker1 → DaemonSet Pod 1개
Worker2 → DaemonSet Pod 1개
```

**③ Worker2를 배포 대상에서 제외**

```bash
kubectl label node worker2 lab.example.com/agent-

kubectl get daemonsets
kubectl get pods -o wide
```

Worker2의 해당 DaemonSet Pod가 정리되는지 확인한다.

**④ Worker2 다시 추가**

```bash
kubectl label node worker2 lab.example.com/agent=enabled

kubectl get daemonsets
kubectl get pods -o wide
```

Worker2에 Pod가 자동 생성되는지 확인한다.

**⑤ DaemonSet 로그 확인**

```bash
kubectl logs -l app=study-node-agent \
  -c agent --tail=20 --prefix=true
```

**⑥ 실습 정리**

```bash
kubectl delete -f node-agent.yaml

kubectl label node worker1 lab.example.com/agent-
kubectl label node worker2 lab.example.com/agent-
```

### 7. DaemonSet 장애 대응

**공통 공식: 상태 → 상세 → 로그 → 원인 → 검증**

**① 상태 확인**

```bash
kubectl get daemonsets
kubectl get pods -o wide
```

**② 상세 정보 확인**

```bash
kubectl describe daemonset study-node-agent
```

**③ 로그 확인**

```bash
kubectl logs <Pod이름>
```

**④ 원인 파악**

- Node Label이 일치하는가?
- Node가 Ready 상태인가?
- 이미지 다운로드에 실패했는가?
- CPU·메모리 자원이 부족한가?
- Node에 taint가 설정되어 있는가?

**⑤ 정상 검증**

```bash
kubectl get daemonsets
kubectl get pods -o wide
```

Pod가 아예 생성되지 않았다면 로그가 없을 수 있으므로 DaemonSet의 상세 정보와 Node 상태를 먼저 확인한다.

### 8. 4장 3교시 최종 핵심 암기

1. **DaemonSet:** 대상 Node마다 Pod 하나씩 유지
2. **Deployment:** 클러스터 전체의 원하는 Pod 개수 관리
3. **Label:** Kubernetes 리소스에 붙이는 이름표
4. **nodeSelector:** 특정 라벨을 가진 Node 선택
5. **selector.matchLabels:** 관리 대상 Pod 선택
6. **Node 추가:** 조건을 만족하면 DaemonSet Pod 자동 생성
7. **실무 활용:** 로그 수집·모니터링·네트워크 에이전트
8. **장애 대응:** 상태 → 상세 → 로그 → 원인 → 검증

**핵심 한 문장**

> DaemonSet은 조건에 맞는 Node마다 Pod를 하나씩 유지하여 로그 수집이나 모니터링처럼 모든 대상 서버에서 실행해야 하는 기능을 관리한다.

---

# 1~4장 핵심 개념 총정리

**현재 단계에서 반드시 알고 있어야 할 개념**

## 1장 — Kubernetes 기초

| 개념 | 기억할 내용 |
|---|---|
| Kubernetes | 컨테이너를 자동으로 배포하고 관리하는 플랫폼 |
| Container | 애플리케이션과 실행 환경을 격리한 단위 |
| Docker | 컨테이너 이미지 생성과 실행 등에 사용하는 플랫폼 |
| 오케스트레이션 | 여러 컨테이너의 배포·운영·복구를 자동으로 관리하는 것 |
| 선언형 관리 | 원하는 상태를 정의하면 시스템이 해당 상태를 유지하도록 조정 |

**핵심:** Kubernetes는 여러 컨테이너를 효율적으로 운영하기 위해 사용한다.

## 2장 — Kubernetes 아키텍처

| 개념 | 기억할 내용 |
|---|---|
| Cluster | 여러 Node를 묶어 운영하는 Kubernetes 환경 |
| Control Plane | Kubernetes 클러스터를 관리하는 영역 |
| Worker Node | 애플리케이션 Pod가 실행되는 서버 |
| API Server | Kubernetes 관리 요청을 처리하는 진입점 |
| etcd | 클러스터 상태 정보를 저장하는 저장소 |
| Scheduler | Pod를 실행할 Node를 선택 |
| Controller Manager | 원하는 상태를 유지하도록 여러 컨트롤러를 실행 |
| kubelet | 각 Node에서 Pod 실행 상태 관리 |
| containerd | 실제 컨테이너 실행 |
| kubeadm | 클러스터 초기화 및 Node 참여 |
| kubectl | Kubernetes를 관리하는 명령어 도구 |
| Flannel | Pod 간 네트워크 연결을 지원하는 CNI 플러그인 |

**핵심:** Control Plane이 클러스터를 관리하고, Worker Node에서 실제 컨테이너가 실행된다.

## 3장 — Pod와 YAML

| 개념 | 기억할 내용 |
|---|---|
| Pod | 하나 이상의 컨테이너를 포함하는 Kubernetes 최소 배포 단위 |
| YAML | Kubernetes 리소스의 원하는 구성을 작성하는 형식 |
| apiVersion | Kubernetes API 버전 |
| kind | 생성할 리소스 종류 |
| metadata | 리소스 이름·라벨 등 정보 |
| spec | 원하는 리소스 구성 |
| restartPolicy | 컨테이너 종료 후 재시작 여부 설정 |
| Always | 정상·비정상 종료 모두 재시작 |
| OnFailure | 비정상 종료 시에만 재시작 |
| Never | 컨테이너 재시작 안 함 |
| emptyDir | Pod 수명 동안 사용할 임시 볼륨 |
| 멀티 컨테이너 Pod | 하나의 Pod에서 여러 컨테이너가 함께 동작하는 구조 |
| ImagePullBackOff | 컨테이너 이미지 다운로드 실패 후 재시도 대기 |
| CrashLoopBackOff | 반복적으로 종료되는 컨테이너의 재시작 지연 |

**핵심:** Kubernetes는 Pod 단위로 컨테이너를 관리하며, YAML로 원하는 구성을 정의한다.

## 4장 — Deployment, ReplicaSet, DaemonSet

| 개념 | 기억할 내용 |
|---|---|
| Deployment | 애플리케이션 배포·복제·업데이트 관리 |
| ReplicaSet | 원하는 개수의 Pod 유지 |
| replicas | 유지하려는 Pod 개수 |
| Self-Healing | 원하는 상태에 맞게 자동으로 복구·조정하는 동작 |
| Scaling | Pod 개수나 자원 크기를 조절하는 것 |
| Rolling Update | 기존 Pod를 새 버전으로 점진적으로 교체 |
| maxSurge | 배포 중 추가 생성할 수 있는 Pod 수 |
| maxUnavailable | 배포 중 허용되는 Available Pod 감소 수 |
| Revision | Deployment Pod 템플릿의 변경 이력 |
| Rollback | 이전 배포 구성으로 복구 |
| DaemonSet | 대상 Node마다 Pod 하나씩 유지 |
| Label | 리소스 분류용 이름표 |
| nodeSelector | 특정 라벨의 Node 선택 |

**핵심:** Deployment는 애플리케이션의 복제본을 관리하고, DaemonSet은 대상 Node마다 필요한 Pod를 실행한다.

---

## 1~4장 최종 암기 — 핵심 10문장

1. **Kubernetes:** 컨테이너를 자동으로 배포하고 관리하는 플랫폼이다.
2. **Control Plane:** 클러스터를 관리하는 영역이다.
3. **Worker Node:** 실제 애플리케이션 Pod가 실행되는 서버다.
4. **API Server:** Kubernetes 관리 요청을 처리한다.
5. **Scheduler:** Pod를 실행할 Node를 선택한다.
6. **kubelet / containerd:** kubelet은 Pod 실행을 관리하고 containerd는 컨테이너를 실제로 실행한다.
7. **Pod:** 하나 이상의 컨테이너를 포함하는 최소 배포 단위다.
8. **Deployment:** 애플리케이션의 배포와 복제본을 관리한다.
9. **ReplicaSet:** 원하는 Pod 개수를 유지한다.
10. **DaemonSet:** 조건에 맞는 Node마다 Pod 하나씩 유지한다.

**현재 단계의 학습 기준**

위 개념들을 보고 각각 무엇을 하는지 이해하고, Deployment와 DaemonSet처럼 비슷한 기술을 구분할 수 있으면 충분하다.

명령어와 YAML은 반복 실습하면서 익히고, 설계와 장애 대응은 후속 통합 프로젝트에서 더 깊게 학습한다.

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/b9ca57e0-584a-4626-98df-bd86edd0c461" />

