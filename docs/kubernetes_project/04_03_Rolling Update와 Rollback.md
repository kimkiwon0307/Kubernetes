# Kubernetes 프로젝트 학습
## 4장 2교시 — Rolling Update와 Rollback

### 1. Rolling Update란?

**Rolling Update는 기존 Pod를 한꺼번에 종료하지 않고, 새로운 버전의 Pod로 점진적으로 교체하는 배포 방식이다.**

**필요한 이유**
- 기존 Pod를 모두 삭제하면 서비스가 중단될 수 있다.
- 기존 Pod를 유지하면서 새 버전의 Pod를 생성할 수 있다.
- 새 Pod가 준비되면 기존 Pod를 점진적으로 종료한다.
- 배포 중 서비스 중단 가능성을 줄일 수 있다.

**동작 원리**

```text
기존 버전: v1 Pod 3개
        ↓
새 버전 v2 Pod 생성
        ↓
Readiness Probe로 준비 상태 확인
        ↓
기존 v1 Pod 일부 종료
        ↓
새 v2 Pod 추가 생성
        ↓
반복하여 전체 Pod 교체
        ↓
최종: v2 Pod 3개
```

**주의:** Rolling Update만 사용한다고 완전한 무중단 배포가 보장되지는 않는다. Readiness Probe, 가용 Pod 수, 네트워크, 애플리케이션 호환성 등을 함께 고려해야 한다.

### 2. RollingUpdate와 Recreate 비교

| 구분 | RollingUpdate | Recreate |
|---|---|---|
| 배포 방식 | 기존 Pod를 점진적으로 교체 | 기존 Pod 전체 종료 후 새 Pod 생성 |
| 서비스 중단 | 최소화 가능 | 발생 가능성 높음 |
| 기존·신규 버전 공존 | 가능 | 일반적으로 없음 |
| 사용 예 | 웹 서버, API 서버 | 버전 공존이 어려운 환경 |

### 3. maxSurge와 maxUnavailable

**maxSurge**
- 배포 중 추가로 생성할 수 있는 Pod의 최대 개수
- `replicas: 3`, `maxSurge: 1`이면 최대 4개까지 생성 가능

**maxUnavailable**
- 배포 과정에서 사용할 수 없어도 허용되는 Pod의 최대 개수
- `replicas: 3`, `maxUnavailable: 0`이면 최소 3개의 Available Pod를 유지하도록 배포를 진행

```yaml
replicas: 3

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

**핵심 암기**

- `maxSurge`: 얼마나 더 만들 수 있는가?
- `maxUnavailable`: 얼마나 줄어도 되는가?

※ Available Pod는 Ready 상태 등 가용성 조건을 충족한 Pod를 의미한다. 위 설정은 Rolling Update에 대한 제약이며 실제 서버 장애까지 막는 것은 아니다.

### 4. Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  revisionHistoryLimit: 10

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

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
          image: nginx:1.30.6-alpine
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
```

| 설정 | 의미 |
|---|---|
| `replicas: 3` | Pod 3개 유지 |
| `revisionHistoryLimit` | 이전 ReplicaSet 이력 보관 개수 |
| `RollingUpdate` | 점진적인 버전 교체 |
| `maxSurge: 1` | 최대 1개 추가 Pod 허용 |
| `maxUnavailable: 0` | 배포 중 Available Pod 감소 허용 안 함 |
| `readinessProbe` | 컨테이너가 요청을 처리할 준비가 되었는지 검사 |
| `periodSeconds: 5` | 5초마다 검사 |

### 5. 핵심 실습 명령어

**① Deployment 생성 및 확인**

```bash
kubectl apply -f nginx-deployment.yaml

kubectl rollout status deployment/nginx-deploy

kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
```

**② Rolling Update — 이미지 버전 변경**

```bash
kubectl set image deployment/nginx-deploy \
  nginx=nginx:1.31.8-alpine
```

- `set image`: 컨테이너 이미지 변경
- `deployment/nginx-deploy`: 대상 Deployment
- `nginx=`: 컨테이너 이름
- `nginx:1.31.8-alpine`: 새 이미지

**③ 배포 완료 확인**

```bash
kubectl rollout status deployment/nginx-deploy

kubectl get replicasets
kubectl get pods -o wide
```

**④ Revision 이력 확인**

```bash
kubectl rollout history deployment/nginx-deploy
```

특정 Revision 조회:

```bash
kubectl rollout history deployment/nginx-deploy \
  --revision=2
```

※ 실제 Revision 번호는 배포 이력을 확인하여 지정한다.

**⑤ 잘못된 이미지 배포로 장애 재현**

```bash
kubectl set image deployment/nginx-deploy \
  nginx=nginx:9999-does-not-exist-lab
```

장애 확인:

```bash
kubectl rollout status deployment/nginx-deploy \
  --timeout=45s

kubectl get pods -o wide
kubectl describe deployment nginx-deploy
```

문제 Pod에서는 `ImagePullBackOff`가 나타날 수 있다.

**⑥ Rollback으로 정상 복구**

```bash
kubectl rollout history deployment/nginx-deploy

kubectl rollout undo deployment/nginx-deploy
```

복구 검증:

```bash
kubectl rollout status deployment/nginx-deploy

kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
```

**⑦ 현재 이미지 확인**

```bash
kubectl get deployment nginx-deploy \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

정상 복구 목표:

```text
nginx:1.31.8-alpine
```

Rollback 후에는 `nginx-deployment.yaml`의 이미지도 현재 정상 버전으로 수정하여 설정을 일치시킨다.

### 6. Revision과 Rollback

**Revision**
- Deployment의 Pod 템플릿 변경 이력
- 이미지 버전 등이 변경되면 새 Revision이 생성된다.
- 단순 `replicas` 변경만으로는 새 Revision이 생성되지 않는다.

**Rollback**
- 배포 오류가 발생했을 때 이전 Pod 템플릿 구성으로 되돌리는 작업
- `kubectl rollout undo`를 이용한다.
- 이전 버전의 ReplicaSet 이력이 남아 있어야 한다.

**주의**
- 배포 실패 시 Kubernetes가 자동으로 Rollback하는 것은 아니다.
- Rollback으로 DB 데이터까지 복원되는 것은 아니다.
- Rollback 이후 실제 애플리케이션 상태를 반드시 검증해야 한다.

### 7. 핵심 명령어 비교

| 명령어 | 목적 |
|---|---|
| `kubectl set image` | 이미지 버전 변경 |
| `kubectl rollout status` | 배포 진행 상태 확인 |
| `kubectl rollout history` | 배포 이력 조회 |
| `kubectl rollout undo` | 이전 배포 구성으로 복구 |
| `kubectl rollout restart` | 현재 설정으로 Pod 재시작 배포 |
| `kubectl get replicasets` | 기존·신규 ReplicaSet 확인 |

**중요한 차이**

- `rollout status` = 배포 확인
- `rollout history` = 이력 확인
- `rollout undo` = 이전 버전 복구
- `rollout restart` = 현재 설정으로 재시작 배포

### 8. 장애 대응 방법

**공통 공식: 상태 → 상세 → 로그 → 원인 → 검증**

① 상태 확인

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
```

② 상세 정보 및 이벤트 확인

```bash
kubectl describe deployment nginx-deploy
kubectl describe pod <문제-Pod-이름>
```

③ 컨테이너 로그 확인

```bash
kubectl logs <Pod-이름>
```

④ 원인 분석

| 증상 | 우선 점검 |
|---|---|
| ImagePullBackOff | 이미지 이름·태그·레지스트리·네트워크 |
| CrashLoopBackOff | 애플리케이션 오류·이전 로그 |
| Pending | Node 자원·스케줄링 |
| 새 Pod가 Ready가 안 됨 | Readiness Probe·애플리케이션 상태 |
| Rollout 진행 안 됨 | Deployment·Pod 이벤트 및 이미지 |

⑤ 정상 복구 검증

```bash
kubectl rollout status deployment/nginx-deploy
kubectl get deployments
kubectl get pods -o wide
```

**중요:** 잘못된 이미지 때문에 컨테이너가 실행되지 않았다면 `kubectl logs`보다 `kubectl describe pod`의 Events를 우선 확인한다.

### 9. 면접 핵심 질문

**Q1. Rolling Update란 무엇인가?**

기존 Pod를 한꺼번에 종료하지 않고 새로운 버전의 Pod로 점진적으로 교체하여 서비스 중단 가능성을 최소화하는 배포 방식이다.

**Q2. maxSurge와 maxUnavailable의 차이는?**

maxSurge는 배포 중 추가 생성 가능한 Pod 수이며, maxUnavailable은 배포 중 사용할 수 없어도 허용되는 Pod 수이다.

**Q3. Revision이란 무엇인가?**

Deployment의 Pod 템플릿 변경 이력이며, 이미지 변경 등 새로운 배포를 추적하는 데 사용된다.

**Q4. Rollback은 어떻게 하는가?**

`kubectl rollout history`로 이력을 확인하고, `kubectl rollout undo`로 이전 정상 배포 구성으로 복구한다.

**Q5. 배포 실패 시 장애 대응 방법은?**

Deployment와 Pod 상태를 확인하고 Events와 로그를 분석한다. 원인을 확인하여 수정하거나 이전 정상 버전으로 Rollback한 뒤 최종 상태를 검증한다.

---

### 10. 최종 핵심 암기

1. **Rolling Update:** 기존 Pod를 새 버전으로 점진적 교체
2. **Recreate:** 기존 Pod 전체 종료 후 새 버전 생성
3. **maxSurge:** 배포 중 추가 가능한 Pod 수
4. **maxUnavailable:** 배포 중 허용되는 Available Pod 감소 수
5. **Readiness Probe:** Pod의 요청 처리 준비 상태 확인
6. **Revision:** Pod 템플릿 변경 이력
7. **Rollback:** 이전 정상 배포 구성으로 복구
8. **set image:** 이미지 버전 변경
9. **rollout history:** 배포 이력 확인
10. **rollout undo:** 이전 버전 복구

**이번 교시 핵심 한 문장**

> Kubernetes Deployment는 Rolling Update로 기존 Pod를 새 버전으로 점진적으로 교체하며, 배포에 문제가 발생하면 Rollback으로 이전 정상 배포 구성으로 되돌릴 수 있다.

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/524b7be0-bdc3-45a9-8b7e-58c31152aa4b" />

