# Kubernetes 프로젝트식 학습 — 10장 1교시

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/82403ece-7204-4655-a7af-23a0873a53a3" />







## 주제: Nginx + API 웹서비스 Deployment 구축

### 1. 학습 목표

- Kubernetes의 Pod, ReplicaSet, Deployment 관계 이해
- 매니페스트(Manifest)의 개념과 YAML 구조 이해
- Namespace를 이용한 리소스 구분
- Nginx Frontend와 API 서버 배포
- kubectl을 활용한 배포 상태 및 장애 확인

---

## 2. 핵심 개념

### ① Pod

Kubernetes에서 컨테이너를 실행하는 기본 단위.

- 하나 이상의 컨테이너를 포함
- Pod마다 클러스터 내부 IP를 할당받음
- Pod가 삭제되면 단독 Pod 자체를 복구하는 상위 컨트롤러는 없음

### ② ReplicaSet

지정한 개수의 Pod를 유지하는 컨트롤러.

예를 들어 `replicas: 3`이면 정상적인 Pod가 3개 유지되도록 조정한다.

### ③ Deployment

애플리케이션의 배포와 업데이트를 관리하는 리소스.

주요 기능:
- ReplicaSet 관리
- Pod 개수 유지
- Rolling Update
- Rollback
- 선언한 상태를 유지하도록 조정

**리소스 관계**

```text
Deployment
    ↓ 관리
ReplicaSet
    ↓ 생성 및 개수 유지
Pod
    ↓ 실행
Container
```

★ 핵심: 일반적인 무상태 웹 애플리케이션은 단독 Pod보다 Deployment를 이용해 관리하는 것이 적합하다.

### ④ Namespace

클러스터 내부의 리소스를 논리적으로 구분하는 공간.

이번 프로젝트의 Namespace: `k8s-mini`

### ⑤ Manifest (매니페스트)

Kubernetes에 생성하거나 관리할 리소스의 원하는 상태를 선언하는 설정.

- YAML: 설정을 표현하는 파일 형식
- Manifest: Kubernetes 리소스를 정의하는 설정
- `kubectl apply -f`: 매니페스트를 Kubernetes API에 적용

★ 핵심: 매니페스트를 통째로 암기하는 것보다 각 필드의 역할을 이해하는 것이 중요하다.

---

## 3. Deployment 매니페스트 핵심 구조

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: mini-frontend
  namespace: k8s-mini

spec:
  replicas: 1

  selector:
    matchLabels:
      app: mini-frontend

  template:
    metadata:
      labels:
        app: mini-frontend

    spec:
      containers:
        - name: nginx
          image: nginx:stable-alpine
          ports:
            - containerPort: 80
```

| 설정 | 의미 |
|---|---|
| `apiVersion` | 사용할 Kubernetes API 버전 |
| `kind` | 생성할 리소스 종류 |
| `metadata.name` | 리소스 이름 |
| `metadata.namespace` | 리소스를 생성할 Namespace |
| `spec.replicas` | 유지할 Pod 개수 |
| `selector.matchLabels` | 관리할 Pod를 선택하는 조건 |
| `template.metadata.labels` | 생성할 Pod에 부여하는 라벨 |
| `containers.name` | 컨테이너 이름 |
| `containers.image` | 실행할 컨테이너 이미지 |
| `containerPort` | 컨테이너가 사용하는 포트 선언 |

### 반드시 이해해야 하는 관계

```yaml
selector:
  matchLabels:
    app: mini-frontend

template:
  metadata:
    labels:
      app: mini-frontend
```

Deployment의 Selector와 Pod Template의 Labels가 일치해야 한다.

**★ Selector는 관리할 Pod를 찾는 조건이고, Labels는 Pod에 부착된 식별 정보다.**

또한 `containerPort: 80`만 설정한다고 외부 접속이 자동으로 허용되는 것은 아니다. 외부 접속은 Service 등을 통해 별도로 구성한다.

---

## 4. 실습 프로젝트 구성

환경:
- VMware Workstation
- Ubuntu 24.04 LTS
- Kubernetes v1.37.1
- containerd 2.2.1
- Control Plane 1대, Worker Node 2대

### 클러스터 구성

| 노드 | IP | 상태 |
|---|---|---|
| control-plane | 192.168.111.150 | Ready |
| worker1 | 192.168.111.152 | Ready |
| worker2 | 192.168.111.151 | Ready |

### 배포한 애플리케이션

| Deployment | 이미지 | 포트 | 용도 |
|---|---|---|---|
| mini-frontend | nginx:stable-alpine | 80 | 웹서버 |
| mini-api | hashicorp/http-echo:1.0.0 | 5678 | HTTP 응답 서버 |

API 서버는 다음 문자열을 반환하도록 구성했다.

```text
Hello from Kubernetes Mini API
```

---

## 5. 핵심 명령어

### 클러스터 확인

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

### 매니페스트 적용

```bash
kubectl apply -f mini-web.yaml
```

### Deployment, ReplicaSet, Pod 확인

```bash
kubectl get deployments -n k8s-mini
kubectl get replicasets -n k8s-mini
kubectl get pods -n k8s-mini -o wide
```

### 배포 완료 여부 확인

```bash
kubectl rollout status deployment/mini-frontend -n k8s-mini
kubectl rollout status deployment/mini-api -n k8s-mini
```

### HTTP 접속 테스트

Nginx:

```bash
kubectl -n k8s-mini port-forward deployment/mini-frontend 18080:80
```

별도 Control Plane 터미널:

```bash
curl -I http://127.0.0.1:18080
```

정상 기대 결과: `HTTP/1.1 200 OK`

API:

```bash
kubectl -n k8s-mini port-forward deployment/mini-api 18081:5678
```

별도 Control Plane 터미널:

```bash
curl http://127.0.0.1:18081/
```

정상 기대 결과: `Hello from Kubernetes Mini API`

**주의:** 위 HTTP 응답은 실습 시 확인해야 하는 기대 결과이며, 실제 성공 출력은 아직 검증하지 않았다.

`port-forward`는 기본적으로 명령을 실행한 Control Plane의 localhost에서 접근한다. Windows PC 외부 접속은 2교시에서 NodePort로 진행한다.

---

## 6. 실제 구축 및 검증 결과

직접 실행한 `kubectl get deploy,rs,pods -n k8s-mini -o wide` 결과:

| 항목 | 실제 상태 |
|---|---|
| mini-frontend Deployment | 1/1 Ready |
| mini-api Deployment | 1/1 Ready |
| mini-frontend ReplicaSet | 1개 유지 |
| mini-api ReplicaSet | 1개 유지 |
| Nginx Pod | worker1에서 Running |
| API Pod | worker2에서 Running |
| 두 Pod 재시작 횟수 | 각각 0회 |

실제 Pod IP:
- Nginx: `10.244.1.55`
- API: `10.244.2.50`

**검증 결과:** Kubernetes 클러스터의 노드 3대가 모두 Ready이며, Deployment → ReplicaSet → Pod 관계가 정상적으로 구성되었음을 확인했다.

HTTP 통신 검증과 1교시 확인문제 채점은 별도로 마무리해야 한다.

---

## 7. 기본 장애 대응 명령어

```bash
# Pod 상태 확인
kubectl get pods -n k8s-mini -o wide

# 이벤트 확인
kubectl get events -n k8s-mini --sort-by=.metadata.creationTimestamp

# Deployment 상세 확인
kubectl describe deployment mini-api -n k8s-mini

# 애플리케이션 로그 확인
kubectl logs deployment/mini-api -n k8s-mini --tail=30
```

주요 장애 상태:

| 상태 | 의미 |
|---|---|
| Pending | Pod가 실행 가능한 노드에 배치되지 못하는 등의 상태 |
| ImagePullBackOff | 컨테이너 이미지 다운로드 실패 후 재시도 대기 |
| CrashLoopBackOff | 컨테이너가 반복 종료되어 재시작 대기 |
| Running | Pod가 노드에 배치되고 컨테이너가 실행 중인 상태 |

★ `Running`만으로 HTTP 서비스 정상 동작이 보장되지는 않는다. 애플리케이션 응답과 준비 상태까지 별도로 확인해야 한다.

---

## 8. 실무 관점 핵심

실무에서는 매니페스트 전체를 암기하기보다 다음 능력이 중요하다.

1. 서비스 요구사항을 파악한다.
2. 적합한 Kubernetes 리소스를 선택한다.
3. 기존 매니페스트 또는 공식 문서를 참고한다.
4. 필요한 설정을 수정하고 검증한다.
5. 실제 배포 상태와 서비스 동작을 확인한다.
6. 문제가 발생하면 상태·이벤트·로그를 분석한다.

유용한 명령어:

```bash
# 리소스 필드 설명 확인
kubectl explain deployment.spec

# Deployment YAML 초안 생성
kubectl create deployment test-nginx \
  --image=nginx:stable-alpine \
  --dry-run=client -o yaml

# 매니페스트 서버 측 검증
kubectl apply --dry-run=server -f mini-web.yaml
```

---

## 9. 1교시 최종 핵심 암기

**반드시 기억할 7가지**

1. Pod = 컨테이너 실행 기본 단위
2. ReplicaSet = Pod 개수 유지
3. Deployment = 배포·업데이트·ReplicaSet 관리
4. Namespace = 리소스의 논리적 구분
5. Manifest = 원하는 Kubernetes 리소스 상태를 선언하는 설정
6. Selector ↔ Labels = 관리 대상 Pod를 연결하는 핵심 관계
7. kubectl = Kubernetes API를 통해 리소스를 관리하는 명령어 도구

**1교시 핵심 한 문장**

> Deployment 매니페스트로 원하는 애플리케이션의 상태를 선언하면 Kubernetes가 ReplicaSet과 Pod를 생성·관리하며, 엔지니어는 kubectl을 사용해 실제 상태를 검증하고 장애를 분석한다.

---

**다음 학습: 10장 2교시 — Service, NodePort, 내부 DNS, Pod 확장 및 Nginx–API 연결**
