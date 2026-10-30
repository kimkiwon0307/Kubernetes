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


# Kubernetes 보충 미니 프로젝트 — 2교시

## 1. 학습 목표

**StatefulSet을 이용해 MySQL을 실행하고, Pod가 삭제되더라도 기존 데이터를 유지하는 구조를 구축한다.**

이번 교시에서는 다음 내용을 학습했다.

- MySQL 구축에 필요한 Kubernetes 리소스 구분
- StatefulSet과 Headless Service 연결
- Secret을 이용한 MySQL 비밀번호 관리
- StorageClass · PV · PVC 연결
- `volumeClaimTemplates`를 이용한 PVC 자동 생성
- MySQL 데이터 저장 및 Pod 재생성 후 데이터 유지 검증 방법
- YAML 참조 관계와 장애 대응

---

## 2. 필요한 리소스와 역할

| 리소스 | 역할 | 필요한 이유 |
|---|---|---|
| Secret | 비밀번호 관리 | MySQL 인증정보 전달 |
| StorageClass | 저장소 제공 방식 정의 | Local PV 사용 방식 지정 |
| PV | 실제 저장 공간 등록 | Worker1 디스크 사용 |
| PVC | 저장 공간 요청 | MySQL Pod에서 PV 사용 |
| Headless Service | Pod별 DNS 식별 | StatefulSet의 안정적인 네트워크 이름 |
| StatefulSet | MySQL Pod 관리 | 안정적인 Pod 이름과 저장소 사용 |
| ClusterIP Service | MySQL 접속 주소 | 다른 Pod에서 DB에 접근 |

### ★ 반드시 기억

- **StatefulSet:** MySQL Pod와 저장소를 안정적으로 관리
- **Headless Service:** StatefulSet Pod별 DNS 식별
- **StorageClass:** 저장소 제공 방식
- **PV:** 실제 저장 공간
- **PVC:** 저장 공간 요청
- **Secret:** 인증정보 관리

---

## 3. 전체 아키텍처

```text
Namespace: marble-mini
│
├── Nginx Deployment (1교시)
│     └── Nginx Pod 2개
│
├── MySQL Service
│     └── mini-mysql-svc
│
├── Headless Service
│     └── mini-mysql-hs
│
├── Secret
│     └── mini-mysql-secret
│
└── StatefulSet: mini-mysql
      │
      └── Pod: mini-mysql-0
            │
            ├── Secret 사용
            │
            ├── MySQL 실행
            │
            └── PVC: data-mini-mysql-0
                  │
                  ▼
             PV: mini-mysql-pv
                  │
                  ▼
             Worker1 실제 저장소
             /mnt/k8s-mini-mysql
```

**중요:** Nginx는 아직 MySQL을 사용하는 Spring Boot 애플리케이션이 아니다. 이번 실습에서는 MySQL을 독립적으로 구축하고 DB 접속과 데이터 보존을 검증한다.

---

## 4. YAML 파일 구성

```text
mini-project/
│
├── 01-configmap.yaml        # 1교시
├── 02-deployment.yaml       # 1교시
├── 03-service.yaml          # 1교시
│
├── mysql-secret.env         # GitHub 업로드 금지
├── .gitignore
├── 04-storageclass.yaml
├── 05-pv.yaml
├── 06-mysql-headless.yaml
├── 07-mysql-svc.yaml
└── 08-mysql-sts.yaml
```

이번 교시의 핵심 Kubernetes 리소스 6개 중 Secret은 로컬 환경변수 파일을 이용해 생성한다.

PVC는 별도 YAML을 작성하지 않고 StatefulSet의 `volumeClaimTemplates`로 생성한다.

---

## 5. 가장 중요한 YAML 연결 관계

| 시작 리소스 | YAML 설정 | 연결 대상 |
|---|---|---|
| PV | `storageClassName` | StorageClass 이름 |
| PVC 템플릿 | `storageClassName` | StorageClass 이름 |
| StatefulSet | `serviceName` | Headless Service 이름 |
| StatefulSet | `secretKeyRef.name` | Secret 이름 |
| StatefulSet | `secretKeyRef.key` | Secret 내부 키 |
| StatefulSet | `selector.matchLabels` | Pod의 `labels` |
| Service | `selector` | Pod의 `labels` |
| MySQL 컨테이너 | `volumeMounts.name` | `volumeClaimTemplates.metadata.name` |
| PV | `nodeAffinity` | Worker1의 Node 라벨 |

### ★ 반드시 기억

**Kubernetes 리소스는 YAML 파일명이 아니라 이름을 참조하거나 labels와 selector를 비교해서 연결한다.**

모든 리소스 이름을 똑같이 만드는 것이 아니라, 서로 참조하는 설정값을 일치시켜야 한다.

---

## 6. StorageClass · PV · PVC 연결

### ① StorageClass

파일: `04-storageclass.yaml`

```yaml
kind: StorageClass
metadata:
  name: mini-local-sc
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

위 코드는 핵심 설정만 발췌한 것이다.

- `mini-local-sc`: StorageClass 이름
- `no-provisioner`: PV를 자동 생성하지 않음
- `WaitForFirstConsumer`: Pod 스케줄링을 고려해 PV/PVC 바인딩 결정

따라서 이번 실습에서는 **PV를 직접 생성해야 한다.**

### ② PV

파일: `05-pv.yaml`

```yaml
metadata:
  name: mini-mysql-pv

spec:
  storageClassName: mini-local-sc

  local:
    path: /mnt/k8s-mini-mysql
```

핵심 설정 발췌.

- `storageClassName`: StorageClass와 연결
- `local.path`: Worker1의 실제 저장소 경로
- `nodeAffinity`: 저장소가 존재하는 Worker1을 지정하는 조건
- `Retain`: PVC 삭제 후 저장소 데이터를 자동 삭제하지 않도록 설정
- `ReadWriteOnce`: 한 Node에서 읽기·쓰기 마운트 허용

### ③ PVC

StatefulSet 안에서 생성한다.

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: mini-local-sc
      resources:
        requests:
          storage: 2Gi
```

StatefulSet `mini-mysql`이 첫 Pod를 생성하면 다음 PVC가 생성된다.

```text
Pod: mini-mysql-0
PVC: data-mini-mysql-0
```

### 전체 연결 관계

```text
StorageClass
mini-local-sc
      │
      ├── PV에서 storageClassName으로 참조
      │
      └── PVC 템플릿에서 storageClassName으로 참조
                 │
                 ▼
          PVC: data-mini-mysql-0
                 │
                 │ 조건에 맞는 PV에 바인딩
                 ▼
          PV: mini-mysql-pv
                 │
                 ▼
          Worker1 실제 디스크
```

**★ 반드시 기억: StorageClass는 저장소 제공 방식, PVC는 요청, PV는 실제 저장 공간이다.**

---

## 7. StatefulSet과 Headless Service 연결

### ① Headless Service

파일: `06-mysql-headless.yaml`

```yaml
metadata:
  name: mini-mysql-hs

spec:
  clusterIP: None
  selector:
    app: mini-mysql
```

핵심 설정 발췌.

- `clusterIP: None`: Headless Service
- `selector`: 대상 MySQL Pod 선택

### ② StatefulSet

파일: `08-mysql-sts.yaml`

```yaml
metadata:
  name: mini-mysql

spec:
  serviceName: mini-mysql-hs
  replicas: 1

  selector:
    matchLabels:
      app: mini-mysql

  template:
    metadata:
      labels:
        app: mini-mysql
```

핵심 설정 발췌.

### 연결 관계

```text
Headless Service
metadata.name: mini-mysql-hs
           │
           ▼
StatefulSet
serviceName: mini-mysql-hs
           │
           ▼
Pod: mini-mysql-0
```

Pod별 DNS 예시:

```text
mini-mysql-0.mini-mysql-hs.marble-mini.svc.cluster.local
```

Pod가 Ready이고 Service의 selector가 일치하는 등 DNS 등록 조건이 충족되어야 조회할 수 있다.

### ★ 반드시 기억

**StatefulSet의 `serviceName`은 Headless Service의 이름과 일치해야 한다.**

---

## 8. Secret과 StatefulSet 연결

Secret 이름:

```text
mini-mysql-secret
```

Secret 내부 키:

```text
MYSQL_ROOT_PASSWORD
MYSQL_PASSWORD
```

StatefulSet에서 환경변수로 참조한다.

```yaml
env:
  - name: MYSQL_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mini-mysql-secret
        key: MYSQL_PASSWORD
```

### 연결 관계

```text
Secret
name: mini-mysql-secret
key: MYSQL_PASSWORD
          │
          ▼
StatefulSet
secretKeyRef.name
secretKeyRef.key
          │
          ▼
MySQL 컨테이너 환경변수
MYSQL_PASSWORD
```

**★ 반드시 기억**

- `secretKeyRef.name` = Secret 이름
- `secretKeyRef.key` = Secret 내부 데이터 키

비밀번호가 들어 있는 `mysql-secret.env` 파일은 GitHub에 업로드하지 않는다.

---

## 9. StatefulSet과 PVC 연결

StatefulSet의 PVC 템플릿:

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
```

MySQL 컨테이너의 볼륨 마운트:

```yaml
volumeMounts:
  - name: data
    mountPath: /var/lib/mysql
```

**`data`라는 이름이 일치해야 한다.**

```text
volumeClaimTemplates
name: data
     │
     ▼
PVC: data-mini-mysql-0
     │
     ▼
volumeMounts
name: data
     │
     ▼
MySQL 데이터 경로
/var/lib/mysql
```

### ★ 반드시 기억

`volumeClaimTemplates`는 Pod마다 PVC를 생성하고, `volumeMounts`는 해당 저장소를 컨테이너 내부 경로에 연결한다.

---

## 10. MySQL Service 연결

파일: `07-mysql-svc.yaml`

```yaml
metadata:
  name: mini-mysql-svc

spec:
  type: ClusterIP
  selector:
    app: mini-mysql
  ports:
    - port: 3306
      targetPort: 3306
```

핵심 설정 발췌.

StatefulSet의 Pod 라벨:

```yaml
labels:
  app: mini-mysql
```

두 값이 일치해야 Service가 MySQL Pod를 선택한다.

같은 Namespace의 다른 Pod에서는 다음 주소를 사용할 수 있다.

```text
mini-mysql-svc:3306
```

**Headless Service는 Pod별 DNS 식별, 일반 ClusterIP Service는 MySQL 접근용 안정적인 주소를 제공한다.**

---

## 11. 전체 구축 순서

```text
① Namespace 확인
       ↓
② Worker1 저장 디렉터리 준비
       ↓
③ Worker1 Node 라벨 지정
       ↓
④ Secret 생성
       ↓
⑤ StorageClass 생성
       ↓
⑥ PV 생성
       ↓
⑦ Headless Service 생성
       ↓
⑧ MySQL ClusterIP Service 생성
       ↓
⑨ StatefulSet 생성
       ↓
⑩ Pod와 PVC/PV 상태 확인
       ↓
⑪ MySQL에 데이터 저장
       ↓
⑫ Pod 삭제 및 재생성
       ↓
⑬ 기존 데이터 유지 확인
```

이 순서는 이번 실습의 이해를 돕기 위한 구축 순서이며 모든 Kubernetes 리소스에 동일하게 강제되는 순서는 아니다.

### 주요 적용 명령어

```bash
# StorageClass, PV는 클러스터 범위 리소스
kubectl apply -f 04-storageclass.yaml
kubectl apply -f 05-pv.yaml

# Service와 StatefulSet은 marble-mini Namespace에 생성
kubectl apply -n marble-mini -f 06-mysql-headless.yaml
kubectl apply -n marble-mini -f 07-mysql-svc.yaml
kubectl apply -n marble-mini -f 08-mysql-sts.yaml
```

Secret은 사전에 별도로 생성한다.

```bash
kubectl create secret generic mini-mysql-secret \
  -n marble-mini \
  --from-env-file=mysql-secret.env
```

Secret이 이미 존재한다면 중복 생성하지 않는다.

---

## 12. 정상 동작 검증

### ① StatefulSet 상태

```bash
kubectl get statefulsets -n marble-mini
```

목표:

```text
mini-mysql   1/1
```

### ② Pod 상태

```bash
kubectl get pods -n marble-mini -o wide
```

목표:

```text
mini-mysql-0   Running
```

MySQL이 정상적인지 로그도 확인한다.

### ③ PVC/PV 상태

```bash
kubectl get pvc -n marble-mini
kubectl get pv mini-mysql-pv
```

목표:

```text
PVC: data-mini-mysql-0   Bound
PV:  mini-mysql-pv        Bound
```

### ④ MySQL 데이터 저장

```bash
kubectl exec -it -n marble-mini mini-mysql-0 -- \
  mysql -u marble -p marble
```

```sql
CREATE TABLE IF NOT EXISTS mini_note (
    id INT PRIMARY KEY,
    message VARCHAR(100)
);

INSERT IGNORE INTO mini_note (id, message)
VALUES (1, 'StatefulSet data test');

SELECT * FROM mini_note;
```

### ⑤ Pod 삭제

```bash
kubectl delete pod mini-mysql-0 -n marble-mini
```

StatefulSet은 원하는 Pod 수를 유지하도록 Pod를 재생성한다.

```bash
kubectl rollout status statefulset/mini-mysql \
  -n marble-mini --timeout=300s
```

### ⑥ 데이터 유지 확인

MySQL에 재접속하여 다음 SQL을 실행한다.

```sql
SELECT * FROM mini_note;
```

기존 데이터가 남아 있다면 데이터 유지 검증에 성공한 것이다.

### ★ 반드시 기억

**데이터가 유지되는 것은 StatefulSet이 데이터를 복제했기 때문이 아니라, 기존 PVC/PV와 실제 저장소를 다시 사용하기 때문이다.**

---

## 13. 장애 대응

| 장애 증상 | 확인 대상 | 원인 예시 |
|---|---|---|
| PVC Pending | StorageClass · PV | 이름, 용량, 접근 모드 불일치 |
| Pod Pending | Node · PV | nodeAffinity, Node 자원 문제 |
| MySQL 실행 실패 | Secret · 로그 | 비밀번호, 초기화, 파일 권한 오류 |
| Service 접속 실패 | Service · Pod | selector와 labels 불일치 |
| DNS 조회 실패 | Headless Service | Service 이름, Pod Ready 상태 |
| 데이터 유지 실패 | PVC · PV | 기존 저장소 연결 여부 |

### 장애 분석 명령어

```bash
kubectl get pods,pvc,svc -n marble-mini

kubectl get pv,storageclass

kubectl describe pod mini-mysql-0 -n marble-mini

kubectl describe pvc data-mini-mysql-0 -n marble-mini

kubectl logs mini-mysql-0 -n marble-mini --tail=100

kubectl get events -n marble-mini \
  --sort-by=.lastTimestamp
```

### StatefulSet의 한계

StatefulSet을 생성한다고 다음 기능들이 자동 구현되지는 않는다.

- MySQL Primary/Replica 복제
- 자동 Failover
- DB 백업
- Node 장애 시 데이터 복구
- 완전한 고가용성 DB 운영

특히 이번 Local PV 방식은 Worker1에 저장소가 종속되므로 Worker1이 고장 나면 기존 데이터에 접근하지 못할 수 있다.

---

## 14. 면접 핵심 질문

**Q1. StatefulSet은 왜 사용하나요?**

상태 저장 애플리케이션의 안정적인 Pod 이름과 저장소를 관리하기 위해 사용합니다.

**Q2. StatefulSet과 Deployment의 차이는 무엇인가요?**

Deployment는 주로 무상태 애플리케이션을 관리하고, StatefulSet은 안정적인 Pod 식별자와 Pod별 저장소가 필요한 애플리케이션에 적합합니다.

**Q3. Headless Service는 왜 필요한가요?**

StatefulSet의 Pod별 DNS 이름을 제공하기 위해 사용합니다.

**Q4. volumeClaimTemplates는 무엇인가요?**

StatefulSet의 각 Pod에서 사용할 PVC를 생성하는 템플릿입니다.

**Q5. StorageClass와 PV/PVC는 어떻게 연결되나요?**

PV와 PVC가 StorageClass 이름 및 용량·접근 모드 등의 조건을 통해 바인딩됩니다.

**Q6. Pod를 삭제해도 MySQL 데이터가 유지되는 이유는 무엇인가요?**

재생성된 Pod가 보존된 기존 PVC와 PV를 통해 같은 저장소를 다시 사용하기 때문입니다.

**Q7. StatefulSet을 사용하면 MySQL 고가용성이 자동 구현되나요?**

아닙니다. DB 복제, 자동 장애조치, 백업 및 장애 복구는 별도로 구성해야 합니다.

---

## 15. 최종 핵심 암기

| 리소스 | 핵심 키워드 |
|---|---|
| Secret | `secretKeyRef` |
| StorageClass | `storageClassName` |
| PV | `local.path`, `nodeAffinity` |
| PVC | 저장 공간 요청, `Bound` |
| StatefulSet | `replicas`, `serviceName` |
| Headless Service | `clusterIP: None` |
| volumeClaimTemplates | Pod별 PVC |
| volumeMounts | 컨테이너 저장 경로 |
| Service | `selector` ↔ `labels` |

### ★ 반드시 기억할 5문장

1. **StatefulSet은 상태 저장 Pod의 안정적인 이름과 저장소 사용을 관리한다.**
2. **StatefulSet의 serviceName은 Headless Service의 이름을 참조한다.**
3. **volumeClaimTemplates로 Pod마다 PVC를 생성할 수 있다.**
4. **PVC와 PV는 StorageClass 및 바인딩 조건에 따라 연결된다.**
5. **StatefulSet만으로 DB 복제, 백업, 자동 Failover가 구현되지는 않는다.**

### 이번 교시 핵심 한 문장

> MySQL StatefulSet을 구축할 때는 Headless Service로 Pod별 DNS를 구성하고, Secret으로 비밀번호를 전달하며, volumeClaimTemplates로 생성된 PVC를 PV와 연결하여 Pod가 재생성되어도 기존 데이터를 사용할 수 있도록 한다.

---

## 16. 학습 결과

- 학습 범위: Kubernetes 보충 미니 프로젝트 2교시
- 확인문제: **8문항 중 8문항 정답 (100점)**
- 미니 프로젝트 1~2교시 확인문제: **16/16 정답**
- 이론 및 확인문제: 완료
- 실제 MySQL 구축·데이터 유지 성공 여부: 별도 검증 필요
- 본 과정 진도: 17/26교시
- 다음 학습: **8장 1교시 — Probe · Health Check**


<img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/b4897231-e956-4b44-bd62-63b6355592be" />
