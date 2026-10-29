# Kubernetes 프로젝트 학습

## 5장 3교시 — CoreDNS · DNS · NetworkPolicy

### 1. DNS란?

**DNS(Domain Name System)는 도메인이나 서비스 이름을 IP 주소로 변환하는 시스템이다.**

**필요한 이유**
- IP 주소를 일일이 기억할 필요가 없다.
- Service의 IP를 직접 사용하지 않고 이름으로 통신할 수 있다.
- 애플리케이션 간 통신 설정을 관리하기 편리하다.

**동작 원리**

```text
nginx-svc
    ↓
DNS 조회
    ↓
10.96.100.50 (예시)
    ↓
Service 접속
```

**★ 반드시 기억: DNS는 이름을 IP 주소로 변환한다.**

---

### 2. CoreDNS란?

**CoreDNS는 Kubernetes에서 Service 이름 등을 IP 주소로 변환해 주는 DNS 서버 소프트웨어이다.**

일반적인 kubeadm Kubernetes 환경에서 사용된다.

**동작 원리**

```text
Spring Boot Pod
      ↓
mysql-svc 접속 요청
      ↓
CoreDNS
      ↓
Service IP 응답
      ↓
MySQL Service
      ↓
MySQL Pod
```

**주요 역할**
- Kubernetes Service 이름 해석
- 클러스터 내부 DNS 조회 처리
- Pod가 서비스 이름으로 통신할 수 있도록 지원

**주의**

CoreDNS는 이름을 IP 주소로 변환하는 역할을 한다. 실제 애플리케이션 요청은 Service의 트래픽 전달 기능을 통해 Pod로 전달된다.

**★ 반드시 기억: CoreDNS는 Kubernetes 내부 DNS 서버이다.**

---

### 3. Kubernetes Service DNS 이름

Kubernetes Service의 일반적인 DNS 이름 구조는 다음과 같다.

```text
<Service이름>.<Namespace>.svc.cluster.local
```

예시:

```text
study-web.dns-lab.svc.cluster.local
```

| 구성 | 의미 |
|---|---|
| study-web | Service 이름 |
| dns-lab | Namespace 이름 |
| svc | Service를 나타내는 영역 |
| cluster.local | 클러스터 도메인(일반적인 기본값) |

**같은 Namespace에서 접근**

```bash
wget -qO- http://study-web
```

**다른 Namespace에서 접근**

```bash
wget -qO- http://study-web.dns-lab
```

**전체 DNS 이름으로 접근**

```bash
wget -qO- http://study-web.dns-lab.svc.cluster.local
```

**★ 반드시 기억**

- 같은 Namespace: `Service이름`
- 다른 Namespace: `Service이름.Namespace`
- 전체 DNS 이름: `Service이름.Namespace.svc.cluster.local`

※ `cluster.local`은 일반적인 기본 도메인이며 클러스터 설정에 따라 변경할 수 있다.

---

### 4. NetworkPolicy란?

**NetworkPolicy는 Pod로 들어오거나 나가는 네트워크 통신을 조건에 따라 허용하도록 정의하는 Kubernetes 리소스이다.**

**필요한 이유**
- 불필요한 Pod 간 통신을 제한한다.
- 허가된 애플리케이션만 DB에 접근하도록 설정할 수 있다.
- 네트워크 접근 범위를 줄여 보안을 강화한다.

**예시: 3-Tier 애플리케이션**

```text
WEB → WAS : 허용

WAS → DB  : 허용

WEB → DB  : 차단
```

NetworkPolicy는 Pod Label, Namespace Label, IP 대역, 포트 등을 기준으로 통신을 제어할 수 있다.

**★ 반드시 기억: NetworkPolicy는 Pod 간 네트워크 통신을 제한하는 정책이다.**

---

### 5. Ingress와 Egress의 차이

NetworkPolicy에서는 통신 방향을 두 가지로 구분한다.

| 구분 | 의미 |
|---|---|
| **Ingress** | Pod로 들어오는 통신 |
| **Egress** | Pod에서 나가는 통신 |

예시:

```text
WAS Pod ──────→ DB Pod
   │                │
 Egress           Ingress
```

- WAS Pod 기준: DB로 나가는 Egress 통신
- DB Pod 기준: WAS에서 들어오는 Ingress 통신

**주의**

5장 2교시에서 학습한 `Ingress` 리소스와 NetworkPolicy의 `Ingress`는 서로 다른 개념이다.

- Ingress 리소스: HTTP/HTTPS 도메인·경로별 라우팅
- NetworkPolicy Ingress: Pod로 들어오는 통신 제어

**★ 반드시 기억: Ingress = 들어오는 통신, Egress = 나가는 통신**

---

### 6. NetworkPolicy YAML

다음 정책은 `app: dns-web` 라벨을 가진 Pod의 TCP 80 포트에 대해, 같은 Namespace에서 `access: allowed` 라벨을 가진 Pod의 접근만 허용한다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx-from-approved

spec:
  podSelector:
    matchLabels:
      app: dns-web

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              access: allowed
      ports:
        - protocol: TCP
          port: 80
```

**YAML 설정 설명**

| 설정 | 의미 |
|---|---|
| `kind: NetworkPolicy` | NetworkPolicy 생성 |
| `podSelector` | 정책을 적용할 Pod 선택 |
| `policyTypes: Ingress` | 들어오는 통신에 정책 적용 |
| `ingress.from` | 허용할 통신 출발지 |
| `access: allowed` | 허용할 출발지 Pod의 Label |
| `protocol: TCP` | TCP 프로토콜 사용 |
| `port: 80` | 목적지 포트 80 허용 |

**중요한 원리**
- NetworkPolicy는 기본적으로 허용 규칙을 정의한다.
- 특정 Pod에 Ingress 정책이 적용되면 해당 방향의 통신은 정책이 허용한 경우에만 가능하다.
- 여러 NetworkPolicy가 적용되면 허용 규칙이 합쳐진다.
- 실제 정책 적용에는 이를 지원하는 네트워크 구성요소가 필요하다.

---

### 7. Flannel과 NetworkPolicy의 관계

현재 실습 환경은 다음과 같다.

```text
Kubernetes: kubeadm
Container Runtime: containerd
CNI: Flannel
Node: Control Plane 1대 + Worker 2대
```

**Flannel은 Pod 간 네트워크 연결을 지원하는 CNI 구성요소이다.**

기본 Flannel 구성에서는 NetworkPolicy를 실제로 적용하는 기능이 없을 수 있다.

따라서 NetworkPolicy YAML을 생성해도 통신이 차단되지 않을 수 있다.

**NetworkPolicy를 구현하는 구성의 예**
- Calico
- Cilium
- Flannel과 별도 NetworkPolicy 컨트롤러 조합

| 상황 | 의미 |
|---|---|
| NetworkPolicy 생성 성공 | Kubernetes에 정책 리소스 등록 |
| NetworkPolicy 통신 차단 성공 | 네트워크 구현에서 정책 적용 |
| 정책 생성 성공, 통신 계속 가능 | 정책을 적용할 구성요소가 없는지 점검 필요 |

**★ 반드시 기억**

NetworkPolicy YAML을 작성했다고 실제 네트워크 통신이 반드시 차단되는 것은 아니다.

---

### 8. 핵심 실습 명령어

**① CoreDNS 상태 확인**

```bash
kubectl get deployment coredns -n kube-system

kubectl get pods -n kube-system -l k8s-app=kube-dns

kubectl get svc kube-dns -n kube-system
```

**② DNS 실습 환경 배포**

```bash
kubectl apply -n dns-lab -f dns-lab.yaml

kubectl rollout status deployment/dns-web \
  -n dns-lab

kubectl get pods -n dns-lab -o wide

kubectl get svc -n dns-lab
```

※ `dns-lab` Namespace를 먼저 생성하고, 5장 3교시에서 작성한 YAML 파일을 사용한다.

**③ 같은 Namespace에서 DNS 조회**

```bash
kubectl exec -n dns-lab dns-allowed -- \
  nslookup study-web
```

**④ 다른 Namespace에서 DNS 조회**

```bash
kubectl exec -n dns-other dns-external -- \
  nslookup study-web.dns-lab
```

**⑤ Service 이름으로 통신 확인**

```bash
kubectl exec -n dns-lab dns-allowed -- \
  wget -qO- http://study-web
```

**⑥ Pod DNS 설정 확인**

```bash
kubectl exec -n dns-lab dns-allowed -- \
  cat /etc/resolv.conf
```

**⑦ NetworkPolicy 적용**

```bash
kubectl apply -n dns-lab -f nginx-networkpolicy.yaml

kubectl get networkpolicies -n dns-lab

kubectl describe networkpolicy \
  allow-nginx-from-approved -n dns-lab
```

**⑧ 허용 Pod에서 통신 테스트**

```bash
kubectl exec -n dns-lab dns-allowed -- sh -c \
  'wget -q -T 3 -O /dev/null http://study-web && echo OK || echo FAILED'
```

**⑨ 차단 대상 Pod에서 통신 테스트**

```bash
kubectl exec -n dns-lab dns-blocked -- sh -c \
  'wget -q -T 3 -O /dev/null http://study-web && echo CONNECTED || echo FAILED'
```

NetworkPolicy를 지원하는 환경이라면 허용 대상만 통신에 성공해야 한다.

기본 Flannel 환경이라면 차단 대상도 통신에 성공할 수 있다.

---

### 9. 장애 대응

**공통 공식: 상태 → 상세 → 로그 → 원인 → 검증**

**① DNS 조회 실패**

```bash
# 상태
kubectl get pods -n kube-system
kubectl get svc -n dns-lab

# 상세
kubectl describe svc study-web -n dns-lab

# CoreDNS 로그
kubectl logs -n kube-system \
  deployment/coredns --tail=50

# DNS 재검증
kubectl exec -n dns-lab dns-allowed -- \
  nslookup study-web
```

확인할 원인:
- CoreDNS Pod 정상 실행 여부
- Service 이름과 Namespace
- Pod의 `/etc/resolv.conf`
- DNS 트래픽을 차단하는 NetworkPolicy

**② NetworkPolicy가 적용되지 않을 때**

```bash
kubectl get networkpolicies -n dns-lab

kubectl describe networkpolicy \
  allow-nginx-from-approved -n dns-lab

kubectl get pods -n dns-lab --show-labels

kubectl get daemonsets -A
```

확인할 원인:
- NetworkPolicy가 올바른 Namespace에 있는가?
- Pod Label과 podSelector가 일치하는가?
- 정책을 실제로 적용하는 네트워크 구성요소가 있는가?

**③ Egress 정책 적용 후 DNS 조회가 실패할 때**

DNS 서버로 향하는 UDP/TCP 53번 통신이 허용되어 있는지 확인한다.

---

### 10. 면접 핵심 질문

**Q1. CoreDNS란 무엇인가요?**

Kubernetes에서 Service 이름 등을 IP 주소로 변환하는 DNS 서버입니다.

**Q2. 같은 Namespace와 다른 Namespace에서 Service에 어떻게 접근하나요?**

같은 Namespace에서는 Service 이름을 사용하고, 다른 Namespace에서는 Service 이름과 Namespace를 함께 지정할 수 있습니다.

**Q3. NetworkPolicy란 무엇인가요?**

Pod로 들어오거나 나가는 네트워크 통신을 조건에 따라 제한하는 Kubernetes 리소스입니다.

**Q4. Ingress와 Egress의 차이는 무엇인가요?**

Ingress는 Pod로 들어오는 통신이고, Egress는 Pod에서 나가는 통신입니다.

**Q5. NetworkPolicy YAML은 적용되었는데 통신이 차단되지 않는다면?**

Pod Label과 정책 설정을 확인하고, NetworkPolicy를 실제로 적용하는 네트워크 구성요소가 설치되어 있는지 점검해야 합니다.

---

## 11. 5장 3교시 최종 핵심 암기

| 개념 | 한 줄 암기 |
|---|---|
| DNS | 이름을 IP 주소로 변환 |
| CoreDNS | Kubernetes 내부 DNS 서버 |
| Service DNS | Service 이름으로 통신 |
| Namespace | Kubernetes 리소스를 논리적으로 구분 |
| FQDN | Namespace와 클러스터 도메인을 포함한 전체 DNS 이름 |
| NetworkPolicy | Pod 간 네트워크 통신 제한 규칙 |
| Ingress | Pod로 들어오는 통신 |
| Egress | Pod에서 나가는 통신 |
| podSelector | Label을 기준으로 Pod 선택 |
| Flannel | Pod 간 네트워크 연결 지원 |

### ★ 반드시 기억할 5문장

1. **DNS는 이름을 IP 주소로 변환하는 시스템이다.**
2. **CoreDNS는 Kubernetes에서 DNS 조회를 처리하는 서버이다.**
3. **같은 Namespace에서는 Service 이름만으로 접근할 수 있다.**
4. **NetworkPolicy는 Pod 간 네트워크 통신을 제한하는 규칙이다.**
5. **NetworkPolicy의 실제 동작에는 이를 지원하는 네트워크 구성요소가 필요하다.**

**이번 교시 핵심 한 문장**

> CoreDNS는 Kubernetes 내부에서 Service 이름을 IP 주소로 변환하고, NetworkPolicy는 Pod 간 네트워크 통신을 제한하기 위한 규칙을 정의한다.

---

# 5장 전체 핵심 개념 요약

| 교시 | 개념 | 핵심 역할 |
|---|---|---|
| 1교시 | Service | Pod에 안정적으로 접근할 주소 제공 |
| 1교시 | ClusterIP | 클러스터 내부 통신 |
| 1교시 | EndpointSlice | Service 백엔드 정보 관리 |
| 2교시 | NodePort | Node IP와 포트를 통한 접근 |
| 2교시 | LoadBalancer | 외부 로드 밸런서를 통한 접근 |
| 2교시 | Ingress | HTTP/HTTPS 도메인·경로별 라우팅 규칙 |
| 2교시 | Ingress Controller | Ingress 규칙을 실제로 처리 |
| 3교시 | CoreDNS | Service 이름을 IP로 변환 |
| 3교시 | NetworkPolicy | Pod 간 네트워크 통신 제한 |

**5장 전체 핵심 한 문장**

> Kubernetes에서는 Service로 Pod에 안정적으로 접근하고, NodePort·LoadBalancer·Ingress로 외부 접속을 구성하며, CoreDNS로 이름을 해석하고 NetworkPolicy로 네트워크 통신을 제어한다.
