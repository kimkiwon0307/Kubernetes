# Kubernetes 보충 미니 프로젝트

## 1교시 — Namespace · ConfigMap · Deployment · Service 통합 실습

### 1. 프로젝트 목표

**Kubernetes에서 Nginx 웹서비스를 구축하면서 리소스 간 연결 관계를 이해한다.**

이번 미니 프로젝트에서는 실제 Marble Spring Boot 대신 가벼운 Nginx 이미지를 사용한다.

**구현 목표**
- Namespace로 실습 환경 분리
- ConfigMap으로 웹페이지 내용 관리
- Deployment로 Nginx Pod 2개 실행
- Service로 Nginx Pod에 요청 전달
- YAML 참조 관계 이해
- Service selector 오류 재현 및 복구

---

## 2. 전체 아키텍처

```text
Kubernetes Cluster
│
└── Namespace: marble-mini
      │
      ├── ConfigMap
      │     name: marble-mini-config
      │     └── index.html 저장
      │
      ├── Deployment
      │     name: marble-mini-web
      │     │
      │     ├── Nginx Pod 1
      │     │     └── app=marble-mini-web
      │     │
      │     └── Nginx Pod 2
      │           └── app=marble-mini-web
      │
      └── Service
            name: marble-mini-svc
            selector: app=marble-mini-web
                  │
                  └── Nginx Pod에 요청 전달
```

**요청 흐름**

```text
클러스터 내부 HTTP 요청
        ↓
Service: marble-mini-svc
        ↓
Pod labels와 Service selector 비교
        ↓
Nginx Pod
        ↓
ConfigMap에서 제공된 index.html 응답
```

### ★ 반드시 기억

**Service는 Deployment의 이름이 아니라 Pod의 labels를 기준으로 연결할 대상을 선택한다.**

---

## 3. 필요한 리소스와 선택 이유

| 리소스 | 역할 | 필요한 이유 |
|---|---|---|
| Namespace | 리소스 논리적 분리 | 기존 실습과 충돌 방지 |
| ConfigMap | 일반 설정·파일 관리 | 웹페이지와 컨테이너 이미지 분리 |
| Deployment | Pod 관리 | Nginx Pod 2개 유지 |
| Service | 안정적인 네트워크 주소 제공 | Pod IP 변경에도 접근 가능 |

**이번 교시에서 사용하지 않은 리소스**
- Secret: 민감정보가 없으므로 불필요
- PVC/PV: 영구적으로 저장할 데이터가 없으므로 불필요
- StatefulSet: Pod별 고정 식별자와 저장소가 필요하지 않음

**★ 반드시 기억: 리소스는 무조건 많이 만드는 것이 아니라 필요한 기능에 맞춰 선택한다.**

---

## 4. YAML 파일 구성

```text
mini-project/
│
├── 01-configmap.yaml
├── 02-deployment.yaml
└── 03-service.yaml
```

| YAML 파일 | 생성 리소스 | 역할 |
|---|---|---|
| 01-configmap.yaml | ConfigMap | 웹페이지 내용 저장 |
| 02-deployment.yaml | Deployment | Nginx Pod 생성 및 ConfigMap 연결 |
| 03-service.yaml | Service | Nginx Pod에 네트워크 요청 전달 |

**중요:** YAML 파일명은 Kubernetes 리소스 간 참조 기준이 아니다. YAML 내부의 리소스 이름과 selector, labels 등이 중요하다.

---

## 5. ConfigMap 생성

파일: `01-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: marble-mini-config
data:
  index.html: |
    <!doctype html>
    <html lang="ko">
    <head>
      <meta charset="utf-8">
      <title>Marble Mini</title>
    </head>
    <body>
      <h1>Marble Mini Project</h1>
      <p>Kubernetes Resource Connection Test</p>
    </body>
    </html>
```

**핵심 설정**

| 항목 | 의미 |
|---|---|
| kind: ConfigMap | ConfigMap 리소스 생성 |
| metadata.name | 다른 리소스에서 참조할 이름 |
| data.index.html | 웹페이지 파일 내용 |

ConfigMap의 이름은 `marble-mini-config`이다.

Deployment에서 이 이름을 참조해야 한다.

### ★ 반드시 기억

**ConfigMap은 일반 설정값 또는 파일 내용을 컨테이너 이미지와 분리하여 관리한다.**

---

## 6. Deployment 생성

파일: `02-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: marble-mini-web

spec:
  replicas: 2

  selector:
    matchLabels:
      app: marble-mini-web

  template:
    metadata:
      labels:
        app: marble-mini-web

    spec:
      containers:
        - name: nginx
          image: nginx:stable-alpine

          ports:
            - containerPort: 80

          volumeMounts:
            - name: web-page
              mountPath: /usr/share/nginx/html
              readOnly: true

      volumes:
        - name: web-page
          configMap:
            name: marble-mini-config
            items:
              - key: index.html
                path: index.html
```

### 핵심 설정

| 설정 | 의미 |
|---|---|
| replicas: 2 | Pod 2개 유지 |
| selector.matchLabels | Deployment가 관리할 Pod 조건 |
| template.metadata.labels | 생성할 Pod의 라벨 |
| containers.image | Nginx 컨테이너 이미지 |
| volumeMounts | 컨테이너에 볼륨 연결 |
| volumes.configMap | 사용할 ConfigMap 참조 |

### 연결 관계 ① Deployment ↔ Pod

```yaml
selector:
  matchLabels:
    app: marble-mini-web
```

```yaml
template:
  metadata:
    labels:
      app: marble-mini-web
```

**두 값이 일치해야 한다.**

Deployment의 selector는 어떤 Pod를 관리할지 지정하고, template의 labels는 생성되는 Pod에 붙일 라벨이다.

### 연결 관계 ② Deployment ↔ ConfigMap

ConfigMap:

```yaml
metadata:
  name: marble-mini-config
```

Deployment:

```yaml
volumes:
  - name: web-page
    configMap:
      name: marble-mini-config
```

**ConfigMap의 metadata.name과 Deployment의 configMap.name이 일치해야 한다.**

### 연결 관계 ③ volumes ↔ volumeMounts

```yaml
volumes:
  - name: web-page
```

```yaml
volumeMounts:
  - name: web-page
    mountPath: /usr/share/nginx/html
```

두 `name`이 같아야 컨테이너에 볼륨을 연결할 수 있다.

`mountPath`는 컨테이너 내부에서 파일이 보이는 경로이다.

### ★ 반드시 기억

- **Deployment selector ↔ Pod labels**
- **ConfigMap 이름 ↔ configMap.name**
- **volumes.name ↔ volumeMounts.name**

---

## 7. Service 생성

파일: `03-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: marble-mini-svc

spec:
  type: ClusterIP

  selector:
    app: marble-mini-web

  ports:
    - name: http
      port: 80
      targetPort: 80
```

### 핵심 설정

| 설정 | 의미 |
|---|---|
| type: ClusterIP | 클러스터 내부 접근용 Service |
| selector | 연결할 Pod의 라벨 조건 |
| port: 80 | Service가 요청받는 포트 |
| targetPort: 80 | 요청을 전달할 Pod의 포트 |

### 연결 관계 ④ Service ↔ Pod

Service:

```yaml
selector:
  app: marble-mini-web
```

Pod:

```yaml
labels:
  app: marble-mini-web
```

**두 값이 일치해야 Service가 해당 Pod를 선택한다.**

```text
Service
selector: app=marble-mini-web
          │
          ▼
Pod 1
labels: app=marble-mini-web
          │
          ▼
연결 대상에 포함
```

**★ 반드시 기억: Service는 selector와 Pod labels를 비교하여 백엔드 Pod를 선택한다.**

---

## 8. 전체 YAML 연결 관계 — 가장 중요

| 시작 리소스 | 참조 설정 | 연결 대상 |
|---|---|---|
| Deployment | selector.matchLabels | Pod labels |
| Deployment | volumes.configMap.name | ConfigMap metadata.name |
| Pod 컨테이너 | volumeMounts.name | volumes.name |
| Service | spec.selector | Pod labels |
| Service | targetPort | Pod가 실제 요청을 받는 포트 |

### 반드시 이해할 두 가지 연결 방식

**① 이름으로 연결**

```text
ConfigMap
name: marble-mini-config
        │
        ▼
Deployment
configMap.name: marble-mini-config
```

**② Label과 Selector로 연결**

```text
Service
selector: app=marble-mini-web
        │
        ▼
Pod
labels: app=marble-mini-web
```

**★ 반드시 기억: 모든 리소스 이름을 동일하게 만드는 것이 아니라 서로 참조하는 설정값을 맞춰야 한다.**

---

## 9. 실습 명령어 — 실행 순서

실습 위치: Control Plane

### ① Namespace와 폴더 생성

```bash
mkdir -p ~/k8s-study/mini-project
cd ~/k8s-study/mini-project

kubectl get namespace marble-mini --ignore-not-found
```

Namespace가 없으면:

```bash
kubectl create namespace marble-mini
```

### ② ConfigMap 생성

```bash
nano 01-configmap.yaml

kubectl apply -n marble-mini -f 01-configmap.yaml

kubectl get configmaps -n marble-mini
```

### ③ Deployment 생성

```bash
nano 02-deployment.yaml

kubectl apply -n marble-mini -f 02-deployment.yaml

kubectl rollout status deployment/marble-mini-web \
  -n marble-mini --timeout=180s

kubectl get deployments,pods -n marble-mini -o wide
```

**확인:** Pod 2개가 Running, Deployment READY가 2/2인지 확인한다.

### ④ Service 생성

```bash
nano 03-service.yaml

kubectl apply -n marble-mini -f 03-service.yaml

kubectl get svc -n marble-mini

kubectl describe svc marble-mini-svc -n marble-mini
```

### ⑤ Service와 Pod 연결 확인

```bash
kubectl get pods -n marble-mini --show-labels

kubectl get endpointslices -n marble-mini \
  -l kubernetes.io/service-name=marble-mini-svc
```

EndpointSlice에서 Service가 선택한 Pod의 주소를 확인한다.

### ⑥ 웹페이지 접속 검증

```bash
kubectl exec -n marble-mini \
  deployment/marble-mini-web -- \
  wget -qO- http://marble-mini-svc
```

정상 응답 예시:

```html
<h1>Marble Mini Project</h1>
<p>Kubernetes Resource Connection Test</p>
```

이 결과가 나오면 Service를 통한 Nginx 웹페이지 접근에 성공한 것이다.

---

## 10. 장애 재현 — Service selector 오류

이번 실습의 핵심은 **YAML의 연결 설정이 틀리면 어떤 일이 발생하는지 확인하는 것**이다.

### ① 정상 설정

Service:

```yaml
selector:
  app: marble-mini-web
```

Pod:

```yaml
labels:
  app: marble-mini-web
```

두 값이 일치하므로 Service가 Pod를 선택한다.

### ② 일부러 잘못 설정

Service YAML을 다음처럼 수정한다.

```yaml
selector:
  app: wrong-web
```

변경 내용 적용:

```bash
kubectl apply -n marble-mini -f 03-service.yaml
```

### ③ 장애 확인

```bash
kubectl describe svc marble-mini-svc -n marble-mini

kubectl get pods -n marble-mini --show-labels

kubectl get endpointslices -n marble-mini \
  -l kubernetes.io/service-name=marble-mini-svc
```

**예상 증상**
- Nginx Pod는 정상 실행 중
- Service가 선택할 Pod가 없음
- EndpointSlice에 사용 가능한 백엔드 주소가 없음
- Service를 통한 접속 실패

### ④ 장애 복구

Service selector를 원래대로 수정한다.

```yaml
selector:
  app: marble-mini-web
```

재적용:

```bash
kubectl apply -n marble-mini -f 03-service.yaml
```

다시 확인:

```bash
kubectl describe svc marble-mini-svc -n marble-mini

kubectl exec -n marble-mini \
  deployment/marble-mini-web -- \
  wget -qO- http://marble-mini-svc
```

웹페이지가 다시 표시되면 복구된 것이다.

### ★ 반드시 기억

**Pod가 Running이어도 Service의 selector와 Pod의 labels가 일치하지 않으면 서비스 연결에 실패할 수 있다.**

---

## 11. 장애 대응 핵심 정리

공통 공식:

**상태 → 상세 → 로그/연결 관계 → 원인 → 검증**

| 증상 | 확인 항목 |
|---|---|
| Pod가 생성되지 않음 | Deployment 상태, Pod Events |
| Pod CrashLoopBackOff | 컨테이너 로그, 실행 명령 |
| ConfigMap 연결 실패 | ConfigMap 이름·키·Namespace |
| Service 접속 실패 | Service selector와 Pod labels |
| EndpointSlice에 주소 없음 | Service와 Pod의 선택 관계, Pod Ready |
| 웹페이지 내용이 다름 | ConfigMap 내용, 볼륨 마운트 경로 |
| 포트 연결 오류 | Service port, targetPort, 컨테이너 수신 포트 |

### 핵심 명령어

```bash
kubectl get pods -n marble-mini -o wide

kubectl describe deployment marble-mini-web \
  -n marble-mini

kubectl describe svc marble-mini-svc \
  -n marble-mini

kubectl get pods -n marble-mini --show-labels

kubectl logs -n marble-mini \
  deployment/marble-mini-web
```

**★ 반드시 기억: 서비스 연결 오류에서는 Service의 selector와 Pod의 labels부터 비교한다.**

---

## 12. 면접 핵심 질문

**Q1. Deployment와 Service를 함께 사용하는 이유는 무엇인가요?**

Deployment는 Pod를 생성하고 개수를 유지하며, Service는 해당 Pod에 안정적으로 접근할 수 있는 네트워크 주소를 제공합니다.

**Q2. Service는 Deployment를 직접 참조하나요?**

일반적인 selector 기반 Service는 Deployment 이름이 아니라 Pod의 labels를 기준으로 백엔드를 선택합니다.

**Q3. ConfigMap을 Deployment에 연결하려면 어떻게 하나요?**

ConfigMap을 환경변수 또는 볼륨으로 참조할 수 있습니다. 이번 실습에서는 volumes.configMap.name으로 ConfigMap을 지정하고 volumeMounts로 컨테이너 내부에 연결했습니다.

**Q4. Service가 Pod와 연결되지 않는다면 무엇을 확인하나요?**

Service selector와 Pod labels가 일치하는지 확인하고, EndpointSlice와 Pod Ready 상태를 점검합니다.

**Q5. Deployment의 selector.matchLabels와 template.metadata.labels가 다른 경우 어떻게 되나요?**

Deployment의 selector가 Pod 템플릿의 labels와 일치해야 합니다. 일치하지 않으면 Deployment 생성 또는 설정 적용 과정에서 검증 오류가 발생할 수 있습니다.

**Q6. YAML 파일명은 리소스 연결에 영향을 주나요?**

아닙니다. 리소스 이름, 참조 필드, selector 및 labels 등 YAML 내부의 설정으로 연결합니다.

---

## 13. 최종 핵심 암기

| 개념 | 한 줄 암기 |
|---|---|
| Namespace | 리소스를 논리적으로 구분 |
| ConfigMap | 일반 설정과 파일 내용 관리 |
| Deployment | Pod 생성 및 개수 유지 |
| Service | Pod에 안정적인 네트워크 접근 제공 |
| labels | Pod에 부착하는 식별 정보 |
| selector | 조건에 맞는 Pod를 선택 |
| volumes | Pod에서 사용할 볼륨 정의 |
| volumeMounts | 컨테이너에 볼륨 연결 |
| EndpointSlice | Service의 백엔드 엔드포인트 정보 관리 |

### ★ 반드시 기억할 5문장

1. **Deployment는 Pod를 생성하고 원하는 개수를 유지한다.**
2. **Deployment의 selector.matchLabels는 Pod 템플릿의 labels와 일치해야 한다.**
3. **Service는 selector와 Pod labels를 비교하여 연결할 Pod를 선택한다.**
4. **ConfigMap은 이름으로 참조하고 volumes와 volumeMounts를 통해 컨테이너에 연결할 수 있다.**
5. **YAML 파일명보다 리소스 간 참조 이름과 labels·selector의 관계가 중요하다.**

### 이번 미니 프로젝트 핵심 한 문장

> Kubernetes에서는 Deployment가 Pod를 관리하고, ConfigMap이 설정이나 파일을 제공하며, Service가 Pod의 labels와 selector를 기준으로 네트워크 요청을 전달하도록 각 리소스를 연결해 하나의 애플리케이션을 구성한다.

---

## 14. 학습 결과 및 다음 단계

- 학습 범위: 미니 프로젝트 1교시 — Namespace · ConfigMap · Deployment · Service 연결
- 객관식 확인문제: **8문항 중 8문항 정답 (100점)**
- 실제 배포·장애 복구 성공 여부: 별도 확인 필요
- 미니 프로젝트: **1/2교시 강의 진행**
- 기존 Kubernetes 본 과정: **17/26교시 유지**
- 다음 실습: **미니 프로젝트 2교시 — Secret · StorageClass · PV · PVC · StatefulSet · MySQL 연결**

**주의:** 다음 교시에서 기존 리소스를 활용하므로 `marble-mini` Namespace는 아직 삭제하지 않는다.

<img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/25f38a2e-33db-4473-b8fa-80cfb6545de1" />
