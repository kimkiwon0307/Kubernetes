# Kubernetes 프로젝트 학습

## 5장 2교시 — NodePort · LoadBalancer · Ingress

### 1. Kubernetes 외부 접속이 필요한 이유

5장 1교시에서 학습한 `ClusterIP`는 Kubernetes 클러스터 내부에서 사용하는 Service 타입이다.

그러나 외부 사용자가 Kubernetes에서 실행되는 웹 애플리케이션에 접근하려면 외부 접속 방법이 필요하다.

대표적인 방법은 다음과 같다.

| 구분 | 역할 |
|---|---|
| **ClusterIP** | 클러스터 내부 통신 |
| **NodePort** | Node IP와 특정 포트를 이용한 접속 |
| **LoadBalancer** | 외부 로드 밸런서를 이용한 접속 |
| **Ingress** | 도메인·URL 경로에 따른 HTTP/HTTPS 라우팅 |

**★ 반드시 기억**

- ClusterIP = 내부 통신
- NodePort = Node IP + Port
- LoadBalancer = 외부 로드 밸런서
- Ingress = 도메인·경로별 라우팅

※ Ingress는 Service 타입이 아니라 별도의 Kubernetes 리소스이다.

---

### 2. NodePort란?

**NodePort는 Node의 IP 주소와 지정된 포트를 통해 Kubernetes Service에 접근하도록 하는 Service 타입이다.**

예시:

```text
http://192.168.111.152:30080
```

- `192.168.111.152`: Worker1 IP
- `30080`: NodePort

**동작 원리**

```text
외부 사용자
     ↓
Worker1:30080
     ↓
NodePort Service
     ↓
Nginx Pod
```

일반적인 `externalTrafficPolicy: Cluster` 환경에서는 요청을 받은 Node와 실제 Pod가 실행되는 Node가 달라도 통신할 수 있다.

**기본 포트 범위**

```text
30000 ~ 32767
```

클러스터 설정에 따라 변경 가능하다.

**장점**
- 별도의 외부 로드 밸런서 없이 접근 가능
- VMware 등의 자체 구축 환경에서 실습하기 쉬움
- 설정이 비교적 간단함

**단점**
- Node IP와 포트 번호를 알아야 함
- Node 장애 시 외부 트래픽 분산을 별도로 고려해야 함
- 방화벽 및 외부 노출 보안 설정 필요

**NodePort YAML**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: net-lab-nodeport
spec:
  type: NodePort
  selector:
    app: net-lab-nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
      protocol: TCP
```

| 설정 | 의미 |
|---|---|
| `type: NodePort` | Node 포트를 통한 접속 |
| `selector` | 연결할 Pod 선택 |
| `port` | Service 포트 |
| `targetPort` | Pod의 대상 포트 |
| `nodePort` | Node에서 외부 요청을 받을 포트 |

**★ 반드시 기억:** NodePort는 Node IP와 포트 번호로 Service에 접근하는 방식이다.

---

### 3. LoadBalancer란?

**LoadBalancer는 외부 로드 밸런서를 통해 Kubernetes Service에 접근하도록 하는 Service 타입이다.**

**동작 원리**

```text
외부 사용자
     ↓
외부 Load Balancer
     ↓
Kubernetes Service
     ↓
Nginx Pod
```

**LoadBalancer YAML**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: net-lab-lb
spec:
  type: LoadBalancer
  selector:
    app: net-lab-nginx
  ports:
    - port: 80
      targetPort: 80
```

**클라우드와 자체 구축 환경의 차이**

| 환경 | LoadBalancer |
|---|---|
| AWS 등 클라우드 | 클라우드 LB 연동으로 생성 가능 |
| VMware + kubeadm | 별도 LB 구현이 필요 |
| 자체 구축 + MetalLB | 구성에 따라 외부 IP 할당 가능 |

현재 VMware 환경에서 별도 LoadBalancer 구현이 없다면 다음과 같이 표시될 수 있다.

```text
NAME         TYPE           EXTERNAL-IP
net-lab-lb   LoadBalancer   <pending>
```

`<pending>`은 외부 IP가 아직 할당되지 않았다는 의미이다.

**주의:** Kubernetes의 `LoadBalancer` Service를 생성한다고 반드시 AWS ALB가 생성되는 것은 아니다. 실제 생성되는 로드 밸런서는 환경과 컨트롤러 구현에 따라 다르다.

**★ 반드시 기억:** LoadBalancer 타입을 실제 외부 접속에 사용하려면 외부 로드 밸런서를 제공하는 구성요소가 필요하다.

---

### 4. Ingress란?

**Ingress는 HTTP/HTTPS 요청을 도메인이나 URL 경로에 따라 내부 Service로 전달하도록 규칙을 정의하는 Kubernetes 리소스이다.**

**필요한 이유**

애플리케이션마다 NodePort를 별도로 사용하면 주소와 포트를 관리하기 복잡해질 수 있다.

Ingress를 사용하면 다음처럼 요청을 구분할 수 있다.

```text
app.marble.test/
        ↓
    WEB Service
        ↓
     WEB Pod

app.marble.test/admin
        ↓
   ADMIN Service
        ↓
    ADMIN Pod
```

**Ingress YAML 예시**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: net-lab-ingress
spec:
  ingressClassName: traefik
  rules:
    - host: app.marble.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: net-lab-internal
                port:
                  number: 80
```

| 설정 | 의미 |
|---|---|
| `kind: Ingress` | Ingress 생성 |
| `ingressClassName` | 요청을 처리할 Ingress Controller 선택 |
| `host` | 요청 도메인 |
| `path` | 요청 URL 경로 |
| `pathType: Prefix` | 경로 접두사 기준 매칭 |
| `service.name` | 요청을 전달할 Service |
| `service.port.number` | 해당 Service의 포트 |

**★ 반드시 기억:** Ingress는 HTTP/HTTPS 요청을 도메인과 URL 경로에 따라 전달하도록 규칙을 정의한다.

---

### 5. Ingress와 Ingress Controller의 차이

두 구성요소는 서로 다르다.

| 구분 | 역할 |
|---|---|
| **Ingress** | 트래픽 전달 규칙 정의 |
| **Ingress Controller** | Ingress 규칙을 실제로 처리 |
| **Service** | 연결할 백엔드 Pod 제공 |

**동작 구조**

```text
외부 사용자
     ↓
Ingress Controller (Traefik)
     ↓
Ingress 규칙 적용
     ↓
ClusterIP Service
     ↓
Nginx Pod
```

**대표적인 Ingress Controller**
- Traefik
- HAProxy 기반 컨트롤러
- 클라우드 제공 컨트롤러

**주의:** Ingress 리소스만 생성한다고 외부 접속이 자동으로 되는 것은 아니다. Ingress Controller와 외부에서 컨트롤러에 도달할 수 있는 네트워크 경로가 필요하다.

**2026년 기준 참고**
- Kubernetes Ingress API는 계속 사용할 수 있지만 기능 개발이 동결된 상태이다.
- 새로운 라우팅 설계에서는 Gateway API를 고려한다.
- 기존 커뮤니티 `ingress-nginx` 컨트롤러는 유지보수가 종료되었으므로 신규 환경에서 사용하지 않는 것이 좋다.

---

### 6. NodePort와 Ingress 차이

**NodePort**

```text
192.168.111.152:30080
        ↓
NodePort Service
        ↓
Nginx Pod
```

**Ingress**

```text
app.marble.test
        ↓
Ingress Controller
        ↓
Ingress 규칙
        ↓
ClusterIP Service
        ↓
Nginx Pod
```

**핵심 차이**

- NodePort는 Node IP와 포트를 통해 접근한다.
- Ingress는 HTTP/HTTPS 요청의 도메인과 경로로 백엔드를 구분한다.
- Ingress Controller 자체를 외부에 노출하기 위해 NodePort나 LoadBalancer를 사용할 수도 있다.

---

### 7. 핵심 실습 명령어

실습 환경:

- Control Plane 1대
- Worker1: `192.168.111.152`
- Worker2: `192.168.111.151`
- Namespace: `net-lab`

**① Namespace 생성**

```bash
kubectl create namespace net-lab
```

**② Deployment · ClusterIP · NodePort 생성**

```bash
kubectl apply -n net-lab -f network-lab.yaml

kubectl rollout status -n net-lab \
  deployment/net-lab-nginx

kubectl get pods -n net-lab -o wide
kubectl get svc -n net-lab
```

**③ NodePort 외부 접속 확인**

```bash
curl -I http://192.168.111.152:30080

curl -I http://192.168.111.151:30080
```

정상 목표:

```text
HTTP/1.1 200 OK
```

**④ LoadBalancer 생성 및 확인**

```bash
kubectl apply -n net-lab -f loadbalancer-lab.yaml

kubectl get svc -n net-lab

kubectl describe svc net-lab-lb -n net-lab
```

외부 LB 연동이 없다면 `EXTERNAL-IP`가 `<pending>`으로 나타날 수 있다.

**⑤ Ingress 생성 및 확인**

```bash
kubectl apply -n net-lab -f ingress-lab.yaml

kubectl get ingress -n net-lab

kubectl get ingressclass
```

Ingress Controller가 설치되어 있어야 실제 요청을 처리할 수 있다.

**⑥ Traefik을 NodePort 30081로 설치한 경우 통신 테스트**

```bash
curl -i -H "Host: app.marble.test" \
  http://192.168.111.152:30081/
```

여기서 Host 헤더는 Ingress가 도메인 규칙을 확인할 수 있도록 전달한다.

**⑦ 실습 정리**

```bash
kubectl delete namespace net-lab
```

이 명령은 `net-lab` Namespace와 그 안의 모든 리소스를 삭제하므로, 해당 Namespace에 필요한 리소스가 없는지 먼저 확인한다.

---

### 8. 장애 대응

**공통 공식: 상태 → 상세 → 로그 → 원인 → 검증**

**NodePort 접속 장애**

```bash
# 상태
kubectl get svc -n net-lab
kubectl get pods -n net-lab -o wide

# 상세
kubectl describe svc net-lab-nodeport -n net-lab

# 연결 대상
kubectl get endpointslices -n net-lab \
  -l kubernetes.io/service-name=net-lab-nodeport

# Node 상태
kubectl get nodes -o wide

# 최종 검증
curl -I http://192.168.111.152:30080
```

**대표적인 장애 원인**

| 증상 | 점검할 내용 |
|---|---|
| NodePort 접속 실패 | Node IP·포트·방화벽·kube-proxy |
| Service 연결 대상 없음 | Selector와 Pod Label |
| Pod가 Ready가 아님 | Readiness Probe, 애플리케이션 |
| LoadBalancer pending | 외부 LB 구현·연동 여부 |
| Ingress 404 | Host·Path·IngressClass·Controller |
| Ingress 503 등 | 백엔드 Service 및 Pod 상태 |

---

### 9. 면접 핵심 질문

**Q1. NodePort란 무엇인가요?**

Node IP와 특정 포트를 통해 Kubernetes Service에 접근할 수 있도록 하는 Service 타입입니다.

**Q2. LoadBalancer란 무엇인가요?**

외부 로드 밸런서를 통해 Kubernetes Service에 접근할 수 있도록 하는 타입입니다.

**Q3. Ingress란 무엇인가요?**

HTTP/HTTPS 요청을 도메인이나 URL 경로에 따라 내부 Service로 전달하도록 규칙을 정의하는 리소스입니다.

**Q4. Ingress와 Ingress Controller의 차이는 무엇인가요?**

Ingress는 트래픽 전달 규칙이고, Ingress Controller는 해당 규칙을 실제로 적용하여 요청을 처리하는 구성요소입니다.

**Q5. VMware Kubernetes에서 LoadBalancer가 pending인 이유는 무엇인가요?**

외부 LoadBalancer를 할당하는 클라우드 연동이나 별도의 구현이 없기 때문일 수 있습니다.

---

## 10. 5장 2교시 최종 핵심 암기

1. **ClusterIP:** 클러스터 내부 통신
2. **NodePort:** Node IP와 특정 포트를 통한 접속
3. **LoadBalancer:** 외부 로드 밸런서를 통한 접속
4. **Ingress:** HTTP/HTTPS 도메인·경로별 라우팅 규칙
5. **Ingress Controller:** Ingress 규칙을 실제로 적용
6. **Traefik:** Ingress Controller의 한 종류
7. **Gateway API:** 새로운 네트워크 라우팅 설계에 권장되는 API
8. **NodePort 기본 범위:** 30000~32767

### ★ 반드시 기억할 4문장

**① NodePort는 Node IP와 포트를 이용해 Service에 접속하는 방식이다.**

**② LoadBalancer는 외부 로드 밸런서를 이용해 Service에 접속하는 방식이다.**

**③ Ingress는 HTTP/HTTPS 요청을 도메인과 경로에 따라 라우팅하는 규칙이다.**

**④ Ingress Controller는 Ingress 규칙을 실제로 처리하는 구성요소이다.**

**이번 교시 핵심 한 문장**

> Kubernetes에서 외부 접속을 제공하기 위해 NodePort와 LoadBalancer를 사용할 수 있으며, Ingress는 Ingress Controller를 통해 HTTP/HTTPS 요청을 도메인과 경로에 따라 내부 Service로 전달한다.

<img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/a2fafb0e-561e-4fd3-9d71-3074dfb9a42b" />
