# Kubernetes 프로젝트 학습

## 5장 1교시 — Service와 ClusterIP

### 1. Service란?

**Service는 Pod에 접근할 수 있는 안정적인 네트워크 주소를 제공하는 Kubernetes 리소스이다.**

**필요한 이유**
- Pod는 삭제되거나 재생성될 때 IP가 변경될 수 있다.
- Pod IP를 직접 사용하면 IP 변경 시 통신이 끊길 수 있다.
- Service를 사용하면 Pod IP가 변경되어도 일정한 주소로 접근할 수 있다.
- 여러 Pod를 하나의 Service로 연결할 수 있다.

**동작 구조**

```text
클라이언트 Pod
      ↓
Service (nginx-svc)
      ↓
  ┌───┴───┐
  ↓       ↓
Nginx   Nginx
Pod 1   Pod 2
```

**핵심:** Service는 Pod에 접속할 수 있는 주소를 제공하고, 실제 요청은 연결된 Pod가 처리한다.

---

### 2. ClusterIP란?

**ClusterIP는 Kubernetes 클러스터 내부에서 통신할 때 사용하는 기본 Service 타입이다.**

예시:

```text
Service 이름 : nginx-svc
Service IP   : 10.96.100.50 (예시)
Service Port : 80
```

클러스터 내부의 다른 Pod에서 다음과 같이 접근할 수 있다.

```bash
wget -qO- http://nginx-svc:80
```

**Pod IP와 ClusterIP의 차이**

| 구분 | Pod IP | ClusterIP |
|---|---|---|
| 대상 | 개별 Pod | Service |
| 목적 | Pod에 직접 접근 | 안정적인 서비스 접근 |
| Pod 재생성 시 | 변경될 수 있음 | 유지됨 |
| 사용 예 | Pod 직접 통신 | 애플리케이션 간 통신 |

※ ClusterIP는 Pod가 재생성되어도 유지되지만, Service 자체를 삭제하고 다시 생성하면 달라질 수 있다.

---

### 3. Service의 주요 종류

| 종류 | 역할 |
|---|---|
| **ClusterIP** | 클러스터 내부 통신 |
| NodePort | Node의 포트를 통해 외부 접근 |
| LoadBalancer | 외부 로드 밸런서를 통한 접근 |

※ Ingress는 Service 타입이 아니라 HTTP/HTTPS 요청을 라우팅하는 별도의 Kubernetes 리소스이다.

NodePort, LoadBalancer, Ingress는 5장 2교시에서 학습한다.

---

### 4. Service는 어떻게 Pod를 선택할까?

**Service는 Selector를 사용하여 Label이 일치하는 Pod를 선택한다.**

Pod Label:

```yaml
metadata:
  labels:
    app: svc-demo
```

Service Selector:

```yaml
selector:
  app: svc-demo
```

두 값이 일치하면 해당 Pod가 Service의 연결 대상이 될 수 있다.

```text
Service
selector: app=svc-demo
          ↓
Pod 1: app=svc-demo  → 선택
Pod 2: app=svc-demo  → 선택
Pod 3: app=mysql     → 제외
```

**핵심:** Service Selector와 Pod Label이 일치해야 한다.

---

### 5. EndpointSlice란?

**EndpointSlice는 Service가 연결할 백엔드의 IP 주소, 포트, 준비 상태 등의 정보를 관리하는 Kubernetes 리소스이다.**

```text
Service (nginx-svc)
        ↓
EndpointSlice
        ↓
Pod 1: 10.244.1.10
Pod 2: 10.244.2.15
```

Pod가 교체되면 연결 대상 정보도 갱신될 수 있다.

**핵심:** EndpointSlice는 Service의 실제 연결 대상 정보를 관리한다.

---

### 6. port와 targetPort의 차이

```yaml
ports:
  - port: 8080
    targetPort: 80
```

| 설정 | 의미 |
|---|---|
| `port` | Service가 요청을 받는 포트 |
| `targetPort` | Pod로 전달하는 대상 포트 |

```text
클라이언트
    ↓
Service:8080
    ↓
Nginx Pod:80
```

**핵심 암기**

- `port` = Service 포트
- `targetPort` = Pod 포트

---

### 7. Service YAML 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc

spec:
  type: ClusterIP

  selector:
    app: svc-demo

  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

**주요 설정**

| 설정 | 의미 |
|---|---|
| `kind: Service` | Service 리소스 생성 |
| `name: nginx-svc` | Service 이름 |
| `type: ClusterIP` | 내부 통신용 Service |
| `selector` | 연결할 Pod 선택 |
| `port` | Service 포트 |
| `targetPort` | Pod 대상 포트 |
| `protocol: TCP` | TCP 통신 사용 |

Service는 Pod를 직접 생성하지 않는다. Pod는 Deployment 등의 리소스를 통해 생성한다.

---

### 8. 핵심 실습 명령어

실습 파일 `service-lab.yaml`에는 Nginx Deployment와 ClusterIP Service가 함께 정의되어 있다.

**① Deployment와 Service 생성**

```bash
kubectl apply -f service-lab.yaml

kubectl rollout status deployment/svc-demo-nginx
```

**② Pod와 Service 확인**

```bash
kubectl get pods -l app=svc-demo -o wide

kubectl get svc
kubectl describe svc nginx-svc
```

**③ EndpointSlice 확인**

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=nginx-svc
```

**④ 통신 테스트용 Pod 생성**

```bash
kubectl run svc-client \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600

kubectl wait \
  --for=condition=Ready pod/svc-client \
  --timeout=180s
```

**⑤ Service를 통한 HTTP 통신**

```bash
kubectl exec svc-client -- \
  wget -qO- http://nginx-svc:80/
```

정상적으로 Nginx HTML 페이지가 출력되는지 확인한다.

**⑥ Service DNS 조회**

```bash
kubectl exec svc-client -- nslookup nginx-svc
```

Service 이름이 ClusterIP로 해석되는지 확인한다.

**⑦ Pod 삭제 후 Service 통신 검증**

```bash
kubectl get pods -l app=svc-demo -o wide

kubectl delete pod <Nginx-Pod-이름>

kubectl rollout status deployment/svc-demo-nginx

kubectl get pods -l app=svc-demo -o wide
kubectl get svc nginx-svc
```

다시 HTTP 요청:

```bash
kubectl exec svc-client -- \
  wget -qO- http://nginx-svc:80/
```

**검증할 내용**
- Deployment가 새로운 Pod를 생성하는가?
- Pod IP가 변경되었는가?
- Service의 ClusterIP는 유지되는가?
- Service 이름으로 계속 통신할 수 있는가?

**⑧ 실습 리소스 정리**

```bash
kubectl delete pod svc-client

kubectl delete -f service-lab.yaml
```

---

### 9. Service 장애 대응

**공통 공식: 상태 → 상세 → 로그 → 원인 → 검증**

**① 상태 확인**

```bash
kubectl get svc
kubectl get pods -o wide
```

**② 상세 확인**

```bash
kubectl describe svc nginx-svc
```

**③ 실제 연결 대상 확인**

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=nginx-svc
```

**④ Pod 로그 확인**

```bash
kubectl logs <Pod-이름>
```

**⑤ 원인 확인 및 재검증**

```bash
kubectl get pods --show-labels

kubectl exec svc-client -- \
  wget -qO- http://nginx-svc:80/
```

**대표적인 장애 원인**

| 증상 | 확인할 내용 |
|---|---|
| Service에 연결된 Pod가 없음 | Selector와 Pod Label |
| Pod가 Ready가 아님 | Readiness Probe, 애플리케이션 |
| DNS 조회 실패 | CoreDNS 및 DNS 설정 |
| 연결 거부 | targetPort, 애플리케이션 포트 |
| 통신 시간 초과 | Pod 네트워크, Service 전달 규칙, NetworkPolicy |

※ Pod가 실행되지 않았다면 컨테이너 로그 대신 Pod Events와 상세 상태를 우선 확인한다.

---

### 10. 면접 핵심 질문

**Q1. Kubernetes Service란?**

Service는 Pod에 접근할 수 있는 안정적인 주소를 제공하는 Kubernetes 리소스입니다.

**Q2. Service가 필요한 이유는?**

Pod는 재생성되면서 IP가 변경될 수 있기 때문에 일정한 주소로 통신하기 위해 Service를 사용합니다.

**Q3. ClusterIP란?**

Kubernetes 클러스터 내부에서 통신할 때 사용하는 기본 Service 타입입니다.

**Q4. Service는 어떻게 Pod를 선택하나요?**

Service의 Selector와 일치하는 Label을 가진 Pod를 연결 대상으로 선택합니다.

**Q5. port와 targetPort의 차이는?**

port는 Service가 요청을 받는 포트이고, targetPort는 요청이 전달되는 Pod의 포트입니다.

---

## 11. 5장 1교시 최종 핵심 암기

1. **Service:** Pod에 접근할 수 있는 안정적인 주소 제공
2. **ClusterIP:** Kubernetes 클러스터 내부 통신
3. **Pod IP:** Pod가 재생성되면 변경될 수 있음
4. **Selector:** Label이 일치하는 Pod 선택
5. **EndpointSlice:** 실제 연결 대상의 IP·포트 등 정보 관리
6. **port:** Service가 요청을 받는 포트
7. **targetPort:** 요청을 전달할 Pod의 포트

### 반드시 기억할 3문장

**① Pod는 컨테이너를 실행하는 최소 배포 단위이다.**

**② Deployment는 Pod의 배포와 복제본을 관리한다.**

**③ Service는 Pod에 안정적으로 접근할 수 있는 주소를 제공한다.**

**이번 교시 핵심 한 문장**

> Service는 Pod의 IP가 변경되더라도 일정한 주소로 통신할 수 있도록 해주며, ClusterIP는 클러스터 내부 통신에 사용되는 기본 Service 타입이다.

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/1248d78a-2938-4a4a-8a24-18ce76a335c4" />
