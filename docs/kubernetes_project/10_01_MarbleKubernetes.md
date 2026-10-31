# Kubernetes 프로젝트식 학습
## 10장 1교시 — Marble Kubernetes 아키텍처 설계

### 1. 이번 교시의 목표

기존 VMware/AWS 환경에서 구축했던 **Marble 3-Tier 프로젝트를 Kubernetes 환경으로 전환하기 위한 아키텍처를 설계한다.**

이번 교시의 핵심 학습 내용은 다음과 같다.

- 기존 VM 기반 인프라와 Kubernetes 구조 비교
- WEB/WAS/DB별 필요한 Kubernetes 리소스 선정
- 리소스 간 의존 관계와 YAML 연결 방식 이해
- VMware Kubernetes 클러스터 환경 확인
- Namespace와 프로젝트 폴더 준비
- 향후 Zabbix 모니터링과 Jenkins CI/CD 연동 구조 설계

**★ 반드시 기억: Kubernetes에서는 기존 서버에 프로그램을 직접 설치하는 방식에서, 컨테이너와 Kubernetes 리소스를 이용해 애플리케이션을 관리하는 방식으로 전환한다.**

---

## 2. 기존 Marble과 Kubernetes 구조 비교

| 구분 | 기존 Marble | Kubernetes 전환 |
|---|---|---|
| WEB | Nginx / Apache 설치 | Nginx Deployment |
| WAS | Spring Boot JAR + systemd | Spring Boot Deployment |
| DB | MySQL 서버 | MySQL StatefulSet |
| WEB/WAS 이중화 | VM/EC2 복제 | Deployment replicas |
| 로드밸런싱 | ALB / Apache | Kubernetes Service |
| 서버 간 연결 | 서버 IP와 포트 | Service DNS와 포트 |
| 설정 관리 | 설정 파일 | ConfigMap |
| 비밀번호 관리 | 환경변수·설정 파일 | Secret |
| DB 저장소 | 서버 디스크 | PV / PVC |
| 상태 관리 | systemd, 모니터링 | Kubernetes 컨트롤러·Probe |
| 모니터링 | Zabbix | Zabbix + Kubernetes 모니터링 |
| CI/CD | Jenkins → SSH/SCP → JAR 배포 | Jenkins → 이미지 빌드 → Kubernetes 배포 |

**핵심 변화**

기존에는 서버 IP를 기준으로 애플리케이션을 연결했다.

Kubernetes에서는 주로 **Service의 DNS 이름을 통해 애플리케이션을 연결**한다.

---

## 3. 최종 Marble Kubernetes 아키텍처

```text
          사용자 / 웹 브라우저
                  │
                  ▼
       WEB Service (NodePort)
                  │
                  ▼
       WEB Deployment (Nginx)
          ├── WEB Pod 1
          └── WEB Pod 2
                  │
                  ▼
      WAS Service (ClusterIP)
           marble-was-svc
                  │
                  ▼
    WAS Deployment (Spring Boot)
          ├── WAS Pod 1
          └── WAS Pod 2
                  │
                  ▼
      DB Service (ClusterIP)
           marble-db-svc
                  │
                  ▼
       DB StatefulSet (MySQL)
           marble-db-0
                  │
                  ▼
                 PVC
                  │
                  ▼
                  PV
                  │
                  ▼
          Worker Node 디스크


     [클러스터 외부 관리 VM]

      Zabbix Server
           └── Node / Pod 모니터링

      Jenkins Server
           └── Build / Test / Image / Deploy
```

### VMware 환경

- Control Plane: 1대
- Worker Node: 2대
- Zabbix Server: 별도 관리 VM
- Jenkins Server: 별도 관리 VM

**주의:** WEB/WAS Pod를 각각 2개 생성한다고 반드시 서로 다른 Worker에 배치되는 것은 아니다. Node 분산이 필요하면 Pod Anti-Affinity 또는 Topology Spread Constraints 같은 배치 정책을 추가해야 한다.

---

## 4. WEB 계층에 필요한 리소스

| 리소스 | 역할 |
|---|---|
| ConfigMap | Nginx Reverse Proxy 설정 |
| Deployment | Nginx Pod 생성 및 개수 관리 |
| Service | 외부 요청을 WEB Pod로 전달 |

### WEB 구성 흐름

```text
브라우저
    ↓
marble-web-svc (NodePort)
    ↓
Nginx Pod
    ↓
marble-was-svc:8080
```

### 핵심 YAML 연결

WEB Service:

```yaml
selector:
  app: marble-web
```

WEB Deployment의 Pod:

```yaml
template:
  metadata:
    labels:
      app: marble-web
```

두 설정이 일치해야 Service가 WEB Pod를 선택한다.

### Nginx와 WAS 연결

```nginx
location / {
    proxy_pass http://marble-was-svc:8080;
}
```

이 설정은 Nginx가 WAS Service를 통해 Spring Boot로 요청을 전달하도록 하는 예시이다.

**★ 반드시 기억**

- Deployment는 Pod를 생성하고 관리한다.
- Service는 Pod의 labels를 기준으로 요청 대상을 선택한다.
- Nginx는 WAS Pod IP 대신 WAS Service DNS를 사용한다.

---

## 5. WAS 계층에 필요한 리소스

| 리소스 | 역할 |
|---|---|
| ConfigMap | DB 주소 등 일반 설정 |
| Secret | DB 비밀번호 |
| Deployment | Spring Boot Pod 관리 |
| Service | WEB에서 WAS 접근 |

### WAS 구성 흐름

```text
Nginx
   ↓
marble-was-svc:8080
   ↓
Spring Boot Pod
   ↓
marble-db-svc:3306
   ↓
MySQL Pod
```

### Spring Boot DB 연결 예시

```properties
spring.datasource.url=jdbc:mysql://marble-db-svc:3306/marble
spring.datasource.username=marble
```

DB 비밀번호는 Secret에서 환경변수로 전달할 예정이다.

예시:

```yaml
env:
  - name: SPRING_DATASOURCE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: marble-db-secret
        key: MYSQL_PASSWORD
```

### 핵심 개념

- `marble-was-svc`: WEB이 WAS에 접근하는 주소
- `marble-db-svc`: WAS가 MySQL에 접근하는 주소
- ConfigMap: 일반 설정 관리
- Secret: 비밀번호 등 민감정보 관리

기존 Marble 애플리케이션의 `/health`, `/health/db` 엔드포인트를 활용해 상태를 확인할 예정이다.

**★ 반드시 기억: Spring Boot는 MySQL Pod IP를 직접 사용하지 않고 MySQL Service 이름을 사용한다.**

---

## 6. DB 계층에 필요한 리소스

| 리소스 | 역할 |
|---|---|
| Secret | MySQL 비밀번호 |
| StorageClass | 저장소 제공 방식 정의 |
| PV | Worker의 실제 저장 공간 |
| PVC | 저장 공간 요청 |
| Headless Service | StatefulSet Pod별 DNS 식별 |
| ClusterIP Service | WAS에서 MySQL 접속 |
| StatefulSet | MySQL Pod 및 PVC 관리 |

### DB 구성 흐름

```text
Spring Boot
     ↓
marble-db-svc:3306
     ↓
MySQL Pod: marble-db-0
     ↓
PVC: data-marble-db-0
     ↓
PV: marble-db-pv
     ↓
Worker Node 실제 디스크
```

### 연결 ① StatefulSet ↔ Headless Service

Headless Service:

```yaml
metadata:
  name: marble-db-hs

spec:
  clusterIP: None
```

StatefulSet:

```yaml
spec:
  serviceName: marble-db-hs
```

`serviceName`은 Headless Service 이름을 참조한다.

### 연결 ② StatefulSet ↔ Secret

```yaml
secretKeyRef:
  name: marble-db-secret
  key: MYSQL_PASSWORD
```

- `name`: 참조할 Secret 이름
- `key`: Secret 내부의 데이터 키

### 연결 ③ StatefulSet ↔ PVC

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
```

StatefulSet이 `marble-db-0` Pod를 생성하면 다음 PVC를 생성할 수 있다.

```text
data-marble-db-0
```

### 연결 ④ PVC ↔ 컨테이너

```yaml
volumeMounts:
  - name: data
    mountPath: /var/lib/mysql
```

`volumeMounts.name`은 `volumeClaimTemplates.metadata.name`과 일치해야 한다.

### 연결 ⑤ PVC ↔ PV

```yaml
storageClassName: marble-local-sc
```

PV와 PVC 템플릿이 동일한 StorageClass를 사용하고, 용량·접근 모드 등 바인딩 조건이 맞아야 한다.

이번 Local PV 구성에서는 StorageClass가 PV를 자동 생성하지 않으므로 PV를 직접 준비한다.

**★ 반드시 기억**

StatefulSet은 DB 복제나 자동 Failover를 구현해 주는 기능이 아니다.

이번 프로젝트는 우선 **MySQL 단일 Pod의 데이터 영속성**을 검증한다.

---

## 7. 가장 중요한 YAML 연결 관계 표

| 리소스 | YAML 설정 | 연결 대상 |
|---|---|---|
| WEB Service | selector | WEB Pod labels |
| WEB Deployment | volumes.configMap.name | Nginx ConfigMap |
| Nginx | proxy_pass | WAS Service |
| WAS Service | selector | WAS Pod labels |
| Spring Boot | datasource URL | DB Service |
| WAS Deployment | secretKeyRef | Secret |
| DB Service | selector | MySQL Pod labels |
| DB StatefulSet | serviceName | Headless Service |
| DB StatefulSet | secretKeyRef | Secret |
| DB StatefulSet | volumeClaimTemplates | PVC |
| MySQL 컨테이너 | volumeMounts.name | PVC 템플릿 이름 |
| PVC 템플릿 | storageClassName | StorageClass |
| PV | storageClassName | StorageClass |
| PV | nodeAffinity | 저장소가 있는 Worker Node |

### Kubernetes 리소스의 3가지 주요 연결 방식

**① 이름으로 연결**

```text
StatefulSet.serviceName
          ↓
Headless Service.metadata.name
```

**② Labels와 Selector로 연결**

```text
Service.selector
       ↓
Pod.metadata.labels
```

**③ 주소와 포트로 연결**

```text
Nginx
   ↓
marble-was-svc:8080
   ↓
Spring Boot
   ↓
marble-db-svc:3306
   ↓
MySQL
```

**★ 반드시 기억: YAML 파일명을 일치시키는 것이 아니라, 리소스들이 참조하는 설정값을 올바르게 연결해야 한다.**

---

## 8. Zabbix와 Jenkins 구성 계획

### Zabbix 모니터링

Zabbix Server는 Kubernetes 외부의 VMware 관리 VM에서 운영한다.

```text
Kubernetes Cluster
    ├── Worker Node 상태
    ├── Pod 상태
    ├── Pod 재시작
    └── Deployment 상태
            ↓
      Zabbix 수집 구성
            ↓
       Zabbix Server
            ↓
      Dashboard / Trigger / 알림
```

**외부에 배치하는 이유**

Kubernetes 클러스터 장애 시에도 외부에서 클러스터 상태를 감시할 수 있도록 하기 위해서이다.

모니터링 대상은 기존 서버 CPU·메모리 중심에서 Node, Pod, Deployment 등의 Kubernetes 리소스까지 확장된다.

### Jenkins CI/CD

기존에는 Jenkins가 JAR 파일을 WAS 서버로 복사하고 systemd를 재시작했다.

Kubernetes에서는 컨테이너 이미지를 사용한다.

```text
Git
 ↓
Jenkins
 ↓
Build / Test
 ↓
Docker Image Build
 ↓
Container Registry
 ↓
Kubernetes Deployment Update
 ↓
RollingUpdate
 ↓
Health Check
```

**★ 반드시 기억**

- Zabbix = 모니터링과 장애 알림
- Jenkins = 빌드·테스트·배포 자동화
- Kubernetes 전환 후에도 기존 Zabbix/Jenkins 서버를 유지할 수 있다.

---

## 9. 프로젝트 파일 구조

```text
k8s-marble/
│
├── 00-base/
│   └── namespace.yaml
│
├── 01-db/
│   ├── storageclass.yaml
│   ├── pv.yaml
│   ├── mysql-headless.yaml
│   ├── mysql-service.yaml
│   └── mysql-statefulset.yaml
│
├── 02-was/
│   ├── app-configmap.yaml
│   ├── app-deployment.yaml
│   └── app-service.yaml
│
├── 03-web/
│   ├── nginx-configmap.yaml
│   ├── nginx-deployment.yaml
│   └── nginx-service.yaml
│
├── 04-ops/
│   └── Zabbix 관련 설정
│
└── 05-cicd/
    └── Jenkins 관련 설정
```

비밀번호 등이 포함된 Secret 원본 파일은 GitHub에 공개하지 않는다.

---

## 10. Namespace와 환경 점검

### Kubernetes Node 확인

```bash
kubectl get nodes -o wide

kubectl cluster-info

kubectl get pods -n kube-system
```

확인할 사항:

- Control Plane 1대와 Worker 2대가 Ready인지
- CoreDNS, Flannel 등 필수 구성요소가 정상인지
- 실제 Worker IP와 Node 이름이 무엇인지

### 스토리지 상태 확인

```bash
kubectl get storageclass

kubectl get pv

kubectl get pvc -A
```

### Node 자원 확인

```bash
kubectl describe nodes
```

Metrics Server가 준비된 경우:

```bash
kubectl top nodes
```

`kubectl top`이 동작하지 않으면 Metrics Server 설치·동작 여부를 확인한다.

### 작업 폴더 생성

```bash
mkdir -p ~/k8s-marble/{00-base,01-db,02-was,03-web,04-ops,05-cicd}

cd ~/k8s-marble
```

### Namespace YAML

파일: `00-base/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: marble-k8s
  labels:
    app.kubernetes.io/part-of: marble
```

적용:

```bash
kubectl apply -f 00-base/namespace.yaml

kubectl get namespace marble-k8s
```

**이번 프로젝트의 Namespace는 `marble-k8s`이다.**

---

## 11. 향후 실제 구축 순서

```text
① Kubernetes 클러스터 상태 확인
                 ↓
② Marble 아키텍처 및 Namespace 준비
                 ↓
③ MySQL StatefulSet + PVC/PV 구축
                 ↓
④ Spring Boot 컨테이너 이미지 준비
                 ↓
⑤ WAS Deployment + Service 구성
                 ↓
⑥ Nginx WEB Deployment + Service 구성
                 ↓
⑦ 브라우저 → WEB → WAS → DB 검증
                 ↓
⑧ 8장 운영 기능 적용
                 ↓
⑨ 9장 장애 대응 학습
                 ↓
⑩ Zabbix 모니터링 구성
                 ↓
⑪ Jenkins CI/CD 구성
                 ↓
⑫ 최종 장애·배포 검증 및 포트폴리오 정리
```

8~9장의 학습은 생략하지 않고 Marble 기본 시스템 구축 후 실제 애플리케이션에 적용한다.

---

## 12. 면접 핵심 질문

**Q1. 기존 VM 기반 Marble 프로젝트를 Kubernetes로 전환하는 이유는 무엇인가요?**

컨테이너 기반으로 애플리케이션을 배포하고, Pod 개수 유지, 롤링 업데이트, 서비스 연결, 상태 관리 등을 Kubernetes에서 일관되게 수행하기 위해서입니다.

**Q2. WEB/WAS는 왜 Deployment를 사용하나요?**

동일한 애플리케이션 Pod를 여러 개 실행하고 원하는 개수를 유지하며 버전을 업데이트하기 적합하기 때문입니다.

**Q3. MySQL에 StatefulSet을 사용하는 이유는 무엇인가요?**

안정적인 Pod 식별자와 Pod별 영구 저장소를 관리하기 위해서입니다.

**Q4. Kubernetes에서는 서버 간 IP 대신 무엇을 사용하나요?**

일반적으로 Service DNS 이름과 포트를 사용합니다.

**Q5. Service와 Pod는 어떻게 연결되나요?**

Service의 selector와 Pod labels가 일치하는 Pod를 선택합니다.

**Q6. Spring Boot의 DB 비밀번호는 어디에 저장하나요?**

Secret으로 관리하고 Deployment에서 `secretKeyRef` 등을 통해 참조합니다.

**Q7. 기존 Jenkins와 Zabbix를 Kubernetes에서도 사용할 수 있나요?**

가능합니다. Jenkins는 컨테이너 이미지 기반 배포에 활용하고, Zabbix는 Kubernetes Node 및 리소스 상태 감시에 활용할 수 있습니다.

---

## 13. 최종 핵심 암기

| 개념 | 반드시 기억 |
|---|---|
| WEB | Nginx Deployment |
| WAS | Spring Boot Deployment |
| DB | MySQL StatefulSet |
| 외부 접근 | NodePort Service |
| 내부 통신 | ClusterIP Service |
| 일반 설정 | ConfigMap |
| 비밀번호 | Secret |
| 영구 저장 | PV/PVC |
| Pod별 DNS | Headless Service |
| 자동 배포 | Jenkins |
| 모니터링 | Zabbix |
| 환경 분리 | Namespace |

### ★ 반드시 기억할 7가지

1. **기존 Marble 3-Tier 구조를 유지하면서 애플리케이션 실행 방식을 Kubernetes로 전환한다.**
2. **WEB과 WAS는 Deployment, DB는 StatefulSet을 기본으로 사용한다.**
3. **Service는 selector와 Pod labels를 비교하여 연결할 Pod를 선택한다.**
4. **Nginx는 WAS Service DNS를 사용하고, Spring Boot는 MySQL Service DNS를 사용한다.**
5. **MySQL 영구 데이터는 PVC와 PV를 통해 저장한다.**
6. **Zabbix와 Jenkins는 별도 VMware 관리 VM에서 Kubernetes와 연동한다.**
7. **각 YAML 파일은 이름 참조, selector/labels, DNS와 포트 설정을 통해 연결된다.**

### 이번 교시 핵심 한 문장

> 기존 Marble WEB/WAS/DB 프로젝트를 Kubernetes로 전환하기 위해 Nginx·Spring Boot Deployment와 MySQL StatefulSet을 설계하고, Service·ConfigMap·Secret·PV/PVC를 연결하며, Zabbix 모니터링과 Jenkins CI/CD까지 연동할 수 있는 전체 아키텍처를 준비한다.

---

## 14. 학습 진행 현황

- 학습 범위: 10장 1교시 — Marble Kubernetes 아키텍처 설계
- 객관식 확인문제: **8/8 정답 (100점)**
- 아키텍처 및 YAML 연결 관계: 강의 진행 완료
- VMware 실제 환경 점검·Namespace 적용: 실습 결과 별도 확인 필요
- Kubernetes 정규 과정: **18/26교시 강의 진행**
- 남은 정규 과정: **8교시**
- 다음 수업: **10장 2교시 — MySQL StatefulSet 및 Marble Spring Boot 구축**

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/e1b16b6d-2a8a-475f-998f-aa365269a498" />
