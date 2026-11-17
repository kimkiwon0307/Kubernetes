<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/7cbbf363-9e53-4b08-b49a-608662eff978" />


# Kubernetes 프로젝트식 학습 — 10장 2교시

## 주제: Service·NodePort·DNS·Pod 확장·Nginx Reverse Proxy

## 1. 학습 목표

- Service가 필요한 이유와 동작 원리 이해
- ClusterIP와 NodePort 차이 이해
- Service Selector와 Pod Labels 관계 이해
- Kubernetes 내부 DNS 통신 이해
- Deployment의 Pod 복제본 확장(Scaling)
- ConfigMap으로 Nginx 설정 관리
- Nginx에서 API Service로 Reverse Proxy 구성
- 서비스 연결 및 장애 검증

---

## 2. 전체 아키텍처

```text
Windows Chrome
      |
      | http://192.168.111.152:30080
      v
[mini-frontend-svc]
  Type: NodePort
  Port: 80
  NodePort: 30080
      |
      | Service Selector
      | app: mini-frontend
      v
[Nginx Frontend Pod]
  Container Port: 80
      |
      | /api 요청 발생
      | proxy_pass
      v
[mini-api-svc]
  Type: ClusterIP
  Port: 5678
      |
      | Service Selector
      | app: mini-api
      v
[API Deployment]
      |
      +-- API Pod 1
      +-- API Pod 2
      +-- API Pod 3
```

**핵심:** 외부 사용자는 NodePort로 Nginx에 접근하고, Nginx는 내부 ClusterIP Service를 통해 API 서버에 접근한다.

---

## 3. Service란?

### 등장 배경

Kubernetes의 Pod는 삭제되거나 재생성될 때 IP 주소가 변경될 수 있다.

따라서 다른 애플리케이션이 Pod IP를 직접 사용하면 연결이 불안정해질 수 있다.

### Service의 역할

Service는 여러 Pod에 접근할 수 있는 안정적인 네트워크 접점을 제공하는 Kubernetes 리소스다.

주요 기능:

- 안정적인 Service 주소 제공
- Selector를 이용해 대상 Pod 선택
- 여러 정상 백엔드 Pod로 트래픽 분산
- 내부 또는 외부 통신 구성

**★ 핵심: Service는 Pod 자체가 아니라 Pod에 접근하는 네트워크 창구다.**

---

## 4. ClusterIP와 NodePort

| 구분 | ClusterIP | NodePort |
|---|---|---|
| 목적 | 클러스터 내부 통신 | 노드 포트를 통한 외부 접속 |
| 접근 방법 | Service IP 또는 DNS | Node IP + NodePort |
| 사용 대상 | API, DB 등 내부 서비스 | 외부에서 접속할 웹서비스 |
| 이번 프로젝트 | mini-api-svc | mini-frontend-svc |

### ① ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mini-api-svc
  namespace: k8s-mini
spec:
  type: ClusterIP
  selector:
    app: mini-api
  ports:
    - port: 5678
      targetPort: 5678
      protocol: TCP
```

해석:

- `type: ClusterIP`: 내부 통신용 Service
- `selector`: `app: mini-api` 라벨을 가진 Pod 선택
- `port`: Service가 제공하는 포트
- `targetPort`: 실제 Pod의 목적지 포트

### ② NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mini-frontend-svc
  namespace: k8s-mini
spec:
  type: NodePort
  selector:
    app: mini-frontend
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
      protocol: TCP
```

해석:

- `type: NodePort`: 노드 포트로 외부 접근
- `port: 80`: Service 포트
- `targetPort: 80`: Nginx Pod 목적지 포트
- `nodePort: 30080`: 노드에서 외부 요청을 받는 포트

```text
Node IP:30080
      ↓
Service:80
      ↓
Nginx Pod:80
```

★ `port`, `targetPort`, `nodePort`는 각각 역할이 다르다.

NodePort는 일반적으로 30000~32767 범위를 사용하며, 실제 외부 접속에는 네트워크 및 방화벽 설정도 영향을 준다.

---

## 5. Service Selector와 Pod Labels

Service는 Selector를 이용해 연결할 Pod를 찾는다.

### Service 설정

```yaml
selector:
  app: mini-api
```

### Pod Labels

```yaml
metadata:
  labels:
    app: mini-api
```

두 값이 일치하면 Service가 해당 Pod를 선택할 수 있다.

```text
Service Selector
   app: mini-api
        |
        | 일치하는 라벨 탐색
        v
API Pod Labels
   app: mini-api
```

### Deployment Selector와의 차이

| 항목 | 목적 |
|---|---|
| Deployment Selector | Deployment가 관리할 Pod 식별 |
| Service Selector | 네트워크 요청을 전달할 Pod 식별 |

**★ Service가 존재해도 Selector와 Labels가 맞지 않으면 통신이 실패할 수 있다.**

---

## 6. Kubernetes 내부 DNS

Kubernetes는 CoreDNS 등을 통해 Service 이름을 내부 DNS로 조회할 수 있도록 제공한다.

이번 API Service 이름은 `mini-api-svc`이다.

같은 Namespace에서는 다음 주소를 사용할 수 있다.

```text
http://mini-api-svc:5678
```

전체 DNS 이름은 일반적인 기본 클러스터 도메인 기준으로 다음과 같다.

```text
mini-api-svc.k8s-mini.svc.cluster.local
```

동작 흐름:

```text
Nginx Pod
    |
    | mini-api-svc DNS 조회
    v
CoreDNS
    |
    | Service 주소 응답
    v
API ClusterIP Service
    |
    v
API Pod
```

**★ Pod IP를 직접 작성하지 않고 Service 이름을 사용하므로 Pod가 교체되어도 안정적인 통신 구조를 유지할 수 있다.**

### 내부 DNS 확인

```bash
kubectl run curl-test \
  -n k8s-mini \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never -- \
  curl -s http://mini-api-svc:5678/
```

정상 기대 결과:

```text
Hello from Kubernetes Mini API
```

---

## 7. Scaling — Pod 개수 확장

Scaling은 애플리케이션의 실행 복제본 수를 변경하는 작업이다.

이번 실습에서는 API Pod를 1개에서 3개로 확장한다.

```bash
kubectl scale deployment mini-api \
  -n k8s-mini \
  --replicas=3
```

검증:

```bash
kubectl get deployment mini-api -n k8s-mini

kubectl get pods -n k8s-mini -o wide

kubectl get endpointslices -n k8s-mini
```

기대 결과:

```text
API Deployment
      |
      v
  ReplicaSet
      |
      +-- API Pod 1
      +-- API Pod 2
      +-- API Pod 3
```

Pod를 여러 개 운영하는 이유:

- 요청 처리 용량 확장
- 여러 Pod에 트래픽 분산
- 일부 Pod 장애 시 정상 Pod를 통한 서비스 유지 가능

단, Pod 수가 늘어난다고 성능이 무조건 비례해서 증가하는 것은 아니다.

**실무 주의:** `kubectl scale`로 복제본을 변경했더라도 기존 매니페스트에 `replicas: 1`이 남아 있으면 해당 매니페스트를 재적용할 때 다시 1개로 변경될 수 있다. 선언적 설정 파일도 함께 관리해야 한다.

---

## 8. ConfigMap과 Nginx Reverse Proxy

### ConfigMap이 필요한 이유

Nginx 설정을 컨테이너 이미지와 분리해서 관리하기 위해 사용한다.

이번에는 Nginx의 `/api` 요청을 내부 API Service로 전달하도록 구성했다.

### Nginx 설정

```nginx
server {
    listen 80;
    server_name _;

    location = / {
        default_type text/html;
        return 200 'Kubernetes Mini Frontend';
    }

    location /api {
        proxy_pass http://mini-api-svc:5678/;
    }
}
```

여기서 핵심 설정은 다음과 같다.

```nginx
proxy_pass http://mini-api-svc:5678/;
```

의미:

> Nginx에 들어온 API 요청을 mini-api-svc의 5678번 포트로 전달한다.

`proxy_pass` 뒤의 `/`는 요청 URI 전달 방식에도 영향을 준다. 이번 설정에서는 `/api` 요청이 API 서버의 `/` 경로로 전달된다.

### ConfigMap과 Deployment 연결

```yaml
volumeMounts:
  - name: nginx-conf
    mountPath: /etc/nginx/conf.d/default.conf
    subPath: default.conf

volumes:
  - name: nginx-conf
    configMap:
      name: mini-frontend-nginx-conf
```

| 설정 | 의미 |
|---|---|
| ConfigMap | Nginx 설정 데이터 보관 |
| `volumes` | Pod에서 사용할 설정 볼륨 정의 |
| `volumeMounts` | 컨테이너 내부 경로에 볼륨 연결 |
| `mountPath` | 설정 파일이 연결될 경로 |
| `subPath` | 볼륨 내 특정 파일만 마운트 |

**★ ConfigMap을 생성했다고 자동 적용되는 것이 아니라, Deployment에서 연결해야 한다.**

또한 `subPath`로 마운트한 파일은 ConfigMap 수정만으로 실행 중인 컨테이너에 자동 갱신되지 않는다. Pod 재생성 등을 통한 설정 반영이 필요하다.

---

## 9. 핵심 실습 명령어

### Service 생성

```bash
kubectl apply -f services.yaml

kubectl get svc -n k8s-mini
```

### Service와 Pod 연결 확인

```bash
kubectl get endpointslices -n k8s-mini

kubectl describe svc mini-api-svc -n k8s-mini
```

### API Pod 확장

```bash
kubectl scale deployment mini-api \
  -n k8s-mini --replicas=3
```

### ConfigMap 및 Nginx 수정 적용

```bash
kubectl apply -f frontend-configmap.yaml

kubectl apply -f frontend-deployment-v2.yaml

kubectl rollout status \
  deployment/mini-frontend \
  -n k8s-mini
```

### 전체 리소스 확인

```bash
kubectl get deploy,rs,pods,svc \
  -n k8s-mini -o wide
```

### 내부 통신 확인

```bash
kubectl run curl-test \
  -n k8s-mini \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never -- \
  curl -s http://mini-frontend-svc/api
```

정상 기대 결과:

```text
Hello from Kubernetes Mini API
```

### Windows Chrome 접속

Frontend:

```text
http://192.168.111.152:30080
```

API:

```text
http://192.168.111.152:30080/api
```

위 HTTP 응답과 브라우저 접속은 이번 교시의 검증 목표이며, 실제 실행 성공 여부는 별도로 확인해야 한다.

---

## 10. 기본 장애 대응

| 장애 증상 | 주요 확인 사항 |
|---|---|
| NodePort 접속 실패 | Service 유형·포트·노드 네트워크·방화벽 |
| Service는 있지만 연결 실패 | Selector와 Pod Labels |
| API 요청 시 502 오류 | API Service·Pod·Nginx 프록시 설정 |
| Service의 연결 대상 없음 | EndpointSlice 및 Pod Ready 상태 |
| Nginx Pod 실행 실패 | ConfigMap·설정 문법·Volume Mount |

### 장애 확인 명령어

```bash
# Pod 상태
kubectl get pods -n k8s-mini -o wide

# Service 상세 정보
kubectl describe svc mini-api-svc -n k8s-mini

# 연결 대상 확인
kubectl get endpointslices -n k8s-mini

# Nginx 로그
kubectl logs deployment/mini-frontend \
  -n k8s-mini --tail=50

# API 로그
kubectl logs deployment/mini-api \
  -n k8s-mini --tail=50

# 최근 이벤트
kubectl get events -n k8s-mini \
  --sort-by=.metadata.creationTimestamp
```

---

## 11. 실무에서 알아야 할 핵심

Kubernetes 리소스는 반드시 하나의 고정된 순서로 생성해야 하는 것은 아니다.

예를 들어 다음과 같이 구성할 수 있다.

```text
Namespace
    ↓
ConfigMap / Secret
    ↓
Service
    ↓
Deployment
    ↓
외부 접근 설정
    ↓
정상 동작 검증
```

Service를 Deployment보다 먼저 생성해도 된다. 이 경우 연결 대상 Pod가 준비되기 전까지는 실제 요청 처리가 불가능하다.

**중요한 것은 생성 순서의 암기가 아니라 각 리소스의 의존 관계를 이해하는 것이다.**

---

## 12. 최종 핵심 정리

**반드시 기억할 8가지**

1. Service는 Pod에 접근하는 안정적인 네트워크 접점이다.
2. ClusterIP는 클러스터 내부 통신에 사용한다.
3. NodePort는 노드 IP와 포트로 외부 접근을 제공한다.
4. Service Selector와 Pod Labels가 일치해야 대상 Pod를 선택할 수 있다.
5. CoreDNS를 통해 Service 이름으로 통신할 수 있다.
6. Deployment의 replicas를 변경하면 Pod 개수를 확장하거나 축소할 수 있다.
7. ConfigMap은 애플리케이션 설정을 분리해 관리한다.
8. Nginx의 proxy_pass를 사용해 요청을 다른 내부 서비스로 전달할 수 있다.

### 1~2교시 연결 정리

**1교시**

```text
Deployment → ReplicaSet → Pod
```

**2교시**

```text
사용자 → NodePort Service → Nginx Pod
                                   ↓
                          API ClusterIP Service
                                   ↓
                                API Pod
```

### 한 문장 요약

> Kubernetes Service를 사용하면 Pod IP가 변경되더라도 안정적인 통신 구조를 구성할 수 있으며, NodePort와 내부 DNS, Nginx Reverse Proxy를 이용해 외부 사용자부터 내부 API까지 요청을 전달할 수 있다.

---

**다음 학습: 10장 3교시 — ConfigMap 설정 변경, Rolling Update, Rollback, Probe를 통한 애플리케이션 운영**
