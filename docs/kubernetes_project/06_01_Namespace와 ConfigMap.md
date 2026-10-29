# Kubernetes 프로젝트 학습

## 6장 1교시 — Namespace와 ConfigMap

### 1. Namespace란?

**Namespace는 하나의 Kubernetes 클러스터 안에서 리소스를 논리적으로 구분하는 공간이다.**

**필요한 이유**
- 개발·테스트·운영 환경 구분
- 팀이나 프로젝트별 리소스 관리
- 서로 다른 Namespace에서 동일한 리소스 이름 사용
- 권한 및 리소스 관리 범위 구분

**예시**

```text
Kubernetes Cluster
    │
    ├── marble-dev (개발 환경)
    │     ├── Deployment: marble-app
    │     └── ConfigMap: marble-config
    │
    └── marble-prod (운영 환경)
          ├── Deployment: marble-app
          └── ConfigMap: marble-config
```

서로 다른 Namespace에서는 같은 이름의 Deployment와 ConfigMap을 생성할 수 있다.

**주의 사항**
- Namespace마다 별도의 Kubernetes 클러스터가 필요한 것은 아니다.
- Namespace를 생성해도 네트워크 통신이 자동 차단되지 않는다.
- 하나의 Worker Node에 서로 다른 Namespace의 Pod가 함께 실행될 수 있다.
- Node와 PersistentVolume은 Namespace에 속하지 않는 클러스터 범위 리소스이다.

**★ 반드시 기억: Namespace는 클러스터를 물리적으로 분리하는 것이 아니라 리소스를 논리적으로 구분한다.**

---

### 2. Kubernetes 기본 Namespace

| Namespace | 역할 |
|---|---|
| default | Namespace를 별도로 지정하지 않았을 때 사용하는 기본 공간 |
| kube-system | CoreDNS 등 시스템 구성요소 |
| kube-public | 공개적으로 접근 가능한 일부 클러스터 정보 |
| kube-node-lease | Node 상태 확인용 Lease 정보 |

**주요 명령어**

```bash
# Namespace 목록
kubectl get namespaces

# Namespace 생성
kubectl create namespace marble-dev
kubectl create namespace marble-prod

# 특정 Namespace의 Pod 확인
kubectl get pods -n marble-dev

# 모든 Namespace의 Pod 확인
kubectl get pods -A
```

**핵심 옵션**

- `-n`: 특정 Namespace 지정
- `-A`: 모든 Namespace 조회

---

### 3. ConfigMap이란?

**ConfigMap은 애플리케이션의 일반적인 설정값을 컨테이너 이미지와 분리하여 관리하는 Kubernetes 리소스이다.**

**필요한 이유**
- 개발·운영 환경마다 설정값이 다를 수 있다.
- 설정값 변경을 위해 매번 이미지를 새로 만들 필요가 없다.
- 동일한 이미지에 서로 다른 환경 설정을 적용할 수 있다.
- 설정을 별도 리소스로 관리할 수 있다.

**예시**

```text
동일한 컨테이너 이미지
         │
         ├── 개발 ConfigMap
         │    APP_ENV=development
         │
         └── 운영 ConfigMap
              APP_ENV=production
```

**ConfigMap은 Key-Value 형태로 설정값을 저장한다.**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: marble-config
data:
  APP_ENV: "development"
  APP_MESSAGE: "Marble Development"
```

| 항목 | 의미 |
|---|---|
| kind: ConfigMap | ConfigMap 리소스 생성 |
| metadata.name | ConfigMap 이름 |
| data | 설정값 저장 영역 |
| APP_ENV | 설정 이름(Key) |
| development | 설정값(Value) |

**★ 반드시 기억: ConfigMap은 일반 설정값을 저장하는 리소스이다.**

---

### 4. ConfigMap과 Secret 차이

| 구분 | ConfigMap | Secret |
|---|---|---|
| 목적 | 일반 설정 관리 | 민감정보 관리 |
| 저장 예시 | 실행 환경, 기능 설정 | DB 비밀번호, API 키 |
| 사용 방식 | 환경변수·파일 | 환경변수·파일 등 |
| 학습 | 6장 1교시 | 6장 2교시 |

**주의:** DB 비밀번호와 API 키는 ConfigMap에 저장하지 않는다.

Secret도 안전한 운영을 위해 접근 권한과 저장 시 암호화 등의 보안 설정이 필요하다.

---

### 5. ConfigMap을 Pod에서 사용하는 방법

대표적으로 두 가지 방법이 있다.

**① 환경변수로 전달**

```yaml
envFrom:
  - configMapRef:
      name: marble-config
```

ConfigMap의 여러 값을 컨테이너 환경변수로 전달한다.

**② 설정 파일로 전달**

ConfigMap을 Volume으로 마운트하여 컨테이너 내부에서 파일 형태로 사용할 수 있다.

| 방식 | 특징 |
|---|---|
| envFrom | ConfigMap의 여러 값을 환경변수로 전달 |
| configMapKeyRef | ConfigMap의 특정 값만 환경변수로 전달 |
| Volume Mount | ConfigMap 값을 파일 형태로 제공 |

**★ 반드시 기억: ConfigMap은 환경변수 또는 파일 형태로 Pod에 전달할 수 있다.**

---

### 6. Namespace별 ConfigMap 실습

**① 개발용 ConfigMap**

파일: `configmap-dev.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: marble-config
data:
  APP_ENV: "development"
  APP_MESSAGE: "Marble Development"
```

**② 운영용 ConfigMap**

파일: `configmap-prod.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: marble-config
data:
  APP_ENV: "production"
  APP_MESSAGE: "Marble Production"
```

**③ ConfigMap 적용**

```bash
kubectl apply -n marble-dev -f configmap-dev.yaml

kubectl apply -n marble-prod -f configmap-prod.yaml
```

**④ ConfigMap 확인**

```bash
kubectl get configmaps -n marble-dev
kubectl get configmaps -n marble-prod

kubectl describe configmap marble-config -n marble-dev
kubectl describe configmap marble-config -n marble-prod
```

**실습 목표**

| Namespace | ConfigMap | APP_ENV |
|---|---|---|
| marble-dev | marble-config | development |
| marble-prod | marble-config | production |

같은 ConfigMap 이름이라도 Namespace별로 다른 값을 저장할 수 있다.

---

### 7. Deployment에서 ConfigMap 사용

파일: `marble-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: marble-app
spec:
  replicas: 1

  selector:
    matchLabels:
      app: marble-app

  template:
    metadata:
      labels:
        app: marble-app

    spec:
      containers:
        - name: marble
          image: busybox:1.36
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                echo "APP_ENV=$APP_ENV"
                echo "APP_MESSAGE=$APP_MESSAGE"
                sleep 15
              done

          envFrom:
            - configMapRef:
                name: marble-config
```

이번 실습에서는 실제 Spring Boot 애플리케이션 대신 BusyBox를 사용하여 설정값 전달을 확인한다.

**Deployment 배포**

```bash
kubectl apply -n marble-dev -f marble-deployment.yaml

kubectl apply -n marble-prod -f marble-deployment.yaml
```

**상태 확인**

```bash
kubectl get pods -n marble-dev
kubectl get pods -n marble-prod
```

**로그 확인**

```bash
kubectl logs -n marble-dev deployment/marble-app --tail=4

kubectl logs -n marble-prod deployment/marble-app --tail=4
```

**기대 결과**

```text
[marble-dev]
APP_ENV=development
APP_MESSAGE=Marble Development

[marble-prod]
APP_ENV=production
APP_MESSAGE=Marble Production
```

**핵심 원리**

동일한 Deployment YAML을 사용해도 Namespace별 ConfigMap을 참조하므로 서로 다른 환경 설정을 사용할 수 있다.

---

### 8. ConfigMap 변경 시 주의 사항

**ConfigMap을 환경변수로 전달한 경우, 값을 수정해도 기존 컨테이너의 환경변수가 자동으로 변경되지 않는다.**

**실습 예시**

기존 개발 환경 설정:

```yaml
APP_MESSAGE: "Marble Development"
```

변경 후:

```yaml
APP_MESSAGE: "Marble Development V2"
```

**① 변경된 ConfigMap 적용**

```bash
kubectl apply -n marble-dev -f configmap-dev.yaml
```

**② 현재 Pod 환경변수 확인**

```bash
kubectl exec -n marble-dev deployment/marble-app -- \
  sh -c 'echo "$APP_MESSAGE"'
```

이미 실행 중인 컨테이너는 기존 설정값을 유지한다.

**③ Deployment 재시작 배포**

```bash
kubectl rollout restart -n marble-dev deployment/marble-app

kubectl rollout status -n marble-dev deployment/marble-app
```

**④ 변경된 값 확인**

```bash
kubectl exec -n marble-dev deployment/marble-app -- \
  sh -c 'echo "$APP_MESSAGE"'
```

예상 결과:

```text
Marble Development V2
```

**ConfigMap 변경 시 동작**

| 사용 방식 | 변경된 설정 반영 |
|---|---|
| 환경변수 | Pod 재생성 필요 |
| 일반 Volume 마운트 | 일정 시간 후 파일 갱신 가능 |
| subPath 마운트 | 일반적인 자동 갱신이 적용되지 않음 |

Volume으로 전달한 설정 파일이 갱신되더라도 애플리케이션이 변경된 파일을 다시 읽는지는 별도로 확인해야 한다.

**★ 반드시 기억: 환경변수로 사용 중인 ConfigMap을 수정했다면 Pod 재생성이 필요하다.**

---

### 9. Namespace와 ConfigMap 장애 대응

**공통 공식: 상태 → 상세 → 로그 → 원인 → 검증**

**① 상태 확인**

```bash
kubectl get pods -n marble-dev -o wide

kubectl get configmaps -n marble-dev
```

**② 상세 정보 확인**

```bash
kubectl describe deployment marble-app -n marble-dev

kubectl describe pod <Pod이름> -n marble-dev
```

**③ 로그 확인**

```bash
kubectl logs -n marble-dev deployment/marble-app
```

**④ 원인 분석**

| 증상 | 확인할 내용 |
|---|---|
| Pod가 시작되지 않음 | Pod Events, ConfigMap 참조 |
| CreateContainerConfigError | ConfigMap 이름·키·Namespace |
| 설정값이 비어 있음 | ConfigMap 내용과 환경변수 이름 |
| 변경된 설정이 적용되지 않음 | Pod 재생성 여부 |
| 리소스가 보이지 않음 | Namespace 지정 여부 |

Pod가 시작되지 않았다면 로그가 없을 수 있으므로 `kubectl describe pod`의 Events를 먼저 확인한다.

**⑤ 수정 후 검증**

```bash
kubectl rollout status -n marble-dev deployment/marble-app

kubectl logs -n marble-dev deployment/marble-app --tail=4
```

---

### 10. 면접 핵심 질문

**Q1. Namespace란 무엇인가요?**

Kubernetes 클러스터 안에서 리소스를 논리적으로 구분하는 공간입니다.

**Q2. Namespace를 분리하면 Node도 분리되나요?**

아닙니다. Node는 클러스터 범위 리소스이므로 하나의 Node에 여러 Namespace의 Pod가 함께 실행될 수 있습니다.

**Q3. ConfigMap이란 무엇인가요?**

애플리케이션의 일반적인 설정값을 컨테이너 이미지와 분리하여 관리하는 Kubernetes 리소스입니다.

**Q4. ConfigMap과 Secret의 차이는 무엇인가요?**

ConfigMap은 일반 설정값을, Secret은 비밀번호나 토큰 같은 민감정보를 관리하는 데 사용합니다.

**Q5. ConfigMap을 수정했는데 Pod 설정이 변경되지 않는 이유는 무엇인가요?**

환경변수로 전달한 ConfigMap 값은 컨테이너 시작 시 설정되므로 기존 컨테이너에 자동 반영되지 않습니다. 새 값을 사용하려면 Pod를 재생성해야 합니다.

---

## 11. 6장 1교시 최종 핵심 암기

| 개념 | 한 줄 암기 |
|---|---|
| Namespace | Kubernetes 리소스를 논리적으로 구분 |
| Node | Namespace에 속하지 않는 클러스터 범위 리소스 |
| ConfigMap | 일반적인 애플리케이션 설정값 저장 |
| Key-Value | 설정 이름과 값의 구조 |
| envFrom | 설정값 여러 개를 환경변수로 전달 |
| configMapRef | 사용할 ConfigMap 지정 |
| Secret | 비밀번호·토큰 등 민감정보 관리 |
| rollout restart | Deployment의 Pod를 다시 배포 |

### ★ 반드시 기억할 5문장

1. **Namespace는 하나의 Kubernetes 클러스터에서 리소스를 논리적으로 구분하는 공간이다.**
2. **Node는 Namespace에 속하지 않으며, 하나의 Node에서 여러 Namespace의 Pod가 실행될 수 있다.**
3. **ConfigMap은 애플리케이션의 일반 설정값을 이미지와 분리하여 관리하는 리소스이다.**
4. **ConfigMap은 환경변수 또는 파일 형태로 Pod에 전달할 수 있다.**
5. **환경변수로 전달한 ConfigMap 값은 변경해도 기존 컨테이너에 자동 반영되지 않으므로 Pod 재생성이 필요하다.**

### 이번 교시 핵심 한 문장

> Namespace는 Kubernetes 리소스를 논리적으로 구분하고, ConfigMap은 애플리케이션 설정값을 컨테이너 이미지와 분리하여 환경별로 관리할 수 있도록 하는 리소스이다.
