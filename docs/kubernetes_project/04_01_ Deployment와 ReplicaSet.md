# Kubernetes 프로젝트 학습
## 4장 1교시 — Deployment와 ReplicaSet

### 1. Deployment란?

Deployment는 Kubernetes에서 **애플리케이션의 배포, 복제본 수, 업데이트를 관리하는 리소스**이다.

**필요한 이유**
- 단독 Pod는 삭제하면 자동으로 재생성되지 않는다.
- 실제 서비스에서는 Pod가 삭제되더라도 원하는 개수를 유지해야 한다.
- Deployment를 사용하면 Pod 복제, 자동 복구, 배포 변경 등을 관리할 수 있다.

### 2. Deployment, ReplicaSet, Pod의 관계

```text
Deployment
    │
    │ 배포 및 ReplicaSet 관리
    ▼
ReplicaSet
    │
    │ 원하는 Pod 개수 유지
    ▼
Pod 1    Pod 2    Pod 3
    │       │       │
    └── 애플리케이션 컨테이너 실행
```

| 구성요소 | 역할 |
|---|---|
| Deployment | ReplicaSet 관리, 배포 및 변경 제어 |
| ReplicaSet | 지정된 개수의 Pod 유지 |
| Pod | 실제 애플리케이션 컨테이너 실행 단위 |

**핵심:** Deployment가 ReplicaSet을 관리하고, ReplicaSet이 Pod 수를 유지한다.

### 3. Desired State와 Self-Healing

- **Desired State:** 사용자가 원하는 상태
- **Actual State:** 현재 실제 상태
- **Reconciliation:** 실제 상태를 원하는 상태에 맞추는 과정
- **Self-Healing:** 장애나 리소스 소실 시 원하는 상태로 복구하는 동작

**Pod 자동 복구 원리**

```text
원하는 Pod 수: 3개
        ↓
Pod 1개 삭제
        ↓
현재 Pod 수: 2개
        ↓
ReplicaSet이 차이 감지
        ↓
새 Pod 1개 생성
        ↓
Scheduler가 Node 선택
        ↓
kubelet과 containerd가 컨테이너 실행
        ↓
다시 Pod 3개 유지
```

**중요:** kubelet의 컨테이너 재시작과 ReplicaSet의 Pod 재생성은 서로 다른 동작이다.

### 4. Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-study
  template:
    metadata:
      labels:
        app: nginx-study
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
```

| 설정 | 의미 |
|---|---|
| `apiVersion: apps/v1` | Deployment API 버전 |
| `kind: Deployment` | Deployment 리소스 생성 |
| `metadata.name` | Deployment 이름 |
| `replicas: 2` | 유지할 Pod 개수 |
| `selector.matchLabels` | 관리 대상 Pod의 라벨 선택 조건 |
| `template` | Pod 생성에 사용할 설계도 |
| `template.metadata.labels` | 생성할 Pod에 부여할 라벨 |
| `image` | 컨테이너 이미지 |

**중요:** `selector.matchLabels`와 `template.metadata.labels`가 일치해야 한다.

### 5. 핵심 실습 명령어

**① Deployment 생성**

```bash
kubectl apply -f nginx-deployment.yaml
```

**② Deployment, ReplicaSet, Pod 확인**

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
```

**③ 상세 정보 확인**

```bash
kubectl describe deployment nginx-deploy
```

**④ Pod 삭제 후 자동 복구 확인**

```bash
# 관리 대상 Pod 확인
kubectl get pods -l app=nginx-study

# 조회한 실제 Pod 중 하나 삭제
kubectl delete pod <Pod이름>

# 새 Pod 생성 확인
kubectl get pods -l app=nginx-study -o wide
```

Pod를 삭제하면 ReplicaSet이 새로운 Pod를 생성하여 원하는 개수를 유지한다.

**⑤ Pod 개수 증가 (Scale Out)**

```bash
kubectl scale deployment nginx-deploy --replicas=4
```

**⑥ Pod 개수 감소 (Scale In)**

`nginx-deployment.yaml`의 `replicas`를 3으로 수정한 후:

```bash
kubectl apply -f nginx-deployment.yaml
```

**⑦ 배포 상태 검증**

```bash
kubectl rollout status deployment/nginx-deploy
kubectl get deployments
kubectl get pods -o wide
```

### 6. Scaling이란?

Scaling은 애플리케이션의 처리 능력을 조절하기 위해 복제본 수나 자원 크기를 변경하는 것이다.

| 구분 | 의미 |
|---|---|
| Scale Out | Pod 개수 증가 |
| Scale In | Pod 개수 감소 |
| Scale Up | 개별 Pod 등의 CPU·메모리 자원 증가 |
| Scale Down | 개별 Pod 등의 CPU·메모리 자원 감소 |

**주의**
- Pod 개수를 4개로 늘려도 Worker Node에 균등하게 배치된다는 보장은 없다.
- `kubectl scale`로 변경한 값과 YAML의 `replicas`가 다르면 이후 YAML 재적용 시 값이 다시 변경될 수 있다.
- Pod가 자동 복구되어도 애플리케이션 오류까지 자동 해결되는 것은 아니다.

### 7. Deployment 장애 대응

**공통 공식: 상태 → 상세 → 로그 → 원인 → 검증**

**① 상태 확인**

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
```

**② 상세 정보 및 Events 확인**

```bash
kubectl describe deployment nginx-deploy
kubectl describe pod <Pod이름>
```

**③ 컨테이너 로그 확인**

```bash
kubectl logs <Pod이름>
```

**④ 원인 분석**

| 장애 | 점검 항목 |
|---|---|
| Pending | Node 자원, 스케줄링 조건 |
| ImagePullBackOff | 이미지 이름, 태그, 인증, 네트워크 |
| CrashLoopBackOff | 애플리케이션 로그, 종료 코드 |
| NotReady | Node, kubelet, containerd |
| Deployment 생성 오류 | YAML, selector와 labels |

**⑤ 정상 복구 검증**

```bash
kubectl get deployments
kubectl get pods -o wide
```

원하는 Pod 개수와 실제 준비된 Pod 개수가 일치하는지 확인한다.

### 8. 면접 핵심 질문

**Q1. Deployment란 무엇인가?**

Deployment는 ReplicaSet을 관리하여 애플리케이션을 배포하고, 원하는 복제본 수와 업데이트를 관리하는 Kubernetes 리소스이다.

**Q2. ReplicaSet의 역할은?**

사용자가 설정한 개수의 Pod가 유지되도록 관리한다.

**Q3. Pod를 삭제하면 왜 자동 복구되는가?**

ReplicaSet이 원하는 Pod 개수와 실제 개수를 비교하여 부족한 Pod를 새로 생성하기 때문이다.

**Q4. Deployment와 ReplicaSet의 차이는?**

Deployment는 배포와 업데이트를 관리하며, ReplicaSet은 Pod 복제본 수를 직접 유지한다.

**Q5. Pod 개수가 부족하면 어떻게 점검하는가?**

Deployment, ReplicaSet, Pod의 상태를 확인하고, 상세 정보와 이벤트, 필요한 로그를 분석한다. 원인을 수정한 후 원하는 복제본 수가 정상적으로 준비되었는지 검증한다.

---

### 9. 최종 핵심 암기

1. **Deployment:** 애플리케이션 배포 및 ReplicaSet 관리
2. **ReplicaSet:** 원하는 Pod 개수 유지
3. **replicas:** 복제본 개수
4. **selector:** 관리할 Pod 선택 조건
5. **labels:** Pod 식별을 위한 라벨
6. **template:** Pod 생성 설계도
7. **Self-Healing:** 원하는 상태로 자동 조정
8. **Scale Out/In:** Pod 개수 증가·감소
9. **자동 복구:** Pod 삭제 시 ReplicaSet이 새 Pod 생성
10. **장애 대응:** 상태 → 상세 → 로그 → 원인 → 검증

**핵심 한 문장**

> Deployment는 ReplicaSet을 관리하고, ReplicaSet은 실제 Pod 수를 원하는 개수만큼 유지하여 애플리케이션의 복제와 자동 복구를 지원한다.

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/6d1bfb63-a522-444b-b633-c8fd169cc78f" />
