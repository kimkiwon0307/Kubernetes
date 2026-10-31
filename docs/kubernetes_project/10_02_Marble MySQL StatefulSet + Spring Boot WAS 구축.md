# Kubernetes 프로젝트식 학습

## 10장 2교시 — Marble MySQL StatefulSet + Spring Boot WAS 구축

### 1. 이번 교시의 목표

기존 VM에서 운영하던 Marble Spring Boot와 MySQL을 Kubernetes 환경으로 전환한다.

**구현 목표**

- MySQL StatefulSet으로 DB Pod 생성
- StorageClass·PV·PVC로 데이터 영속성 구성
- Secret으로 MySQL 비밀번호 관리
- Docker 이미지로 Marble Spring Boot 실행
- Deployment와 Service로 WAS 구성
- Spring Boot와 MySQL 연결
- `/health`, `/health/db` 엔드포인트 검증

이번에는 **DB와 WAS 계층만 구축**한다. Nginx WEB 계층은 10장 3교시에서 연결한다.

---

## 2. 전체 아키텍처

```text
Namespace: marble-k8s

      Marble Spring Boot
      Deployment: marble-was
              │
              │ ConfigMap 참조
              │ Secret 참조
              ▼
      WAS Pod (Spring Boot)
              │
              │ JDBC 연결
              ▼
      MySQL ClusterIP Service
         marble-db-svc:3306
              │
              │ selector ↔ labels
              ▼
      MySQL StatefulSet
          marble-db
              │
              ├── Secret 참조
              ├── Headless Service 참조
              │
              ▼
      MySQL Pod: marble-db-0
              │
              ▼
      PVC: data-marble-db-0
              │
              ▼
      PV: marble-db-pv
              │
              ▼
      Worker1 실제 저장소
         /mnt/marble-db
```

**★ 반드시 기억**

Spring Boot는 MySQL Pod IP가 아니라 `marble-db-svc:3306`을 통해 DB에 연결한다.

---

## 3. 필요한 Kubernetes 리소스

### DB 계층

| 리소스 | 이름 | 역할 |
|---|---|---|
| Secret | marble-db-secret | MySQL 비밀번호 |
| StorageClass | marble-local-sc | Local PV 저장소 방식 |
| PV | marble-db-pv | Worker1 저장 공간 |
| PVC | data-marble-db-0 | MySQL 저장 공간 요청 |
| Headless Service | marble-db-hs | Pod별 DNS 이름 |
| ClusterIP Service | marble-db-svc | WAS의 DB 접속 |
| StatefulSet | marble-db | MySQL Pod 및 PVC 관리 |

PVC는 별도 YAML로 만들지 않고 `volumeClaimTemplates`로 생성한다.

### WAS 계층

| 리소스 | 이름 | 역할 |
|---|---|---|
| ConfigMap | marble-app-config | DB 주소 등 일반 설정 |
| Secret | marble-db-secret | DB 비밀번호 참조 |
| Deployment | marble-was | Spring Boot Pod 관리 |
| Service | marble-was-svc | WAS 접근용 내부 주소 |

### ★ 반드시 기억

- **DB = StatefulSet + Service + PVC/PV**
- **WAS = Deployment + Service**
- **ConfigMap = 일반 설정**
- **Secret = 비밀번호 등 민감정보**

---

## 4. DB 저장소 구조

```text
StorageClass: marble-local-sc
             │
             ├── PV에서 참조
             │
             └── PVC 템플릿에서 참조

StatefulSet
     │
     ▼
volumeClaimTemplates
     │
     ▼
PVC: data-marble-db-0
     │
     ▼
PV: marble-db-pv
     │
     ▼
Worker1: /mnt/marble-db
```

### StorageClass 핵심 설정

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: marble-local-sc
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

| 설정 | 의미 |
|---|---|
| metadata.name | StorageClass 이름 |
| no-provisioner | PV 자동 생성하지 않음 |
| WaitForFirstConsumer | Pod 배치를 고려해 PV/PVC 바인딩 |

이번 실습에서는 PV를 직접 생성한다.

### PV 핵심 설정

```yaml
storageClassName: marble-local-sc

local:
  path: /mnt/marble-db
```

실제 PV에는 `capacity`, `accessModes`, `persistentVolumeReclaimPolicy`, `nodeAffinity` 등도 설정한다.

- `ReadWriteOnce`: 한 Node에서 읽기·쓰기 마운트
- `Retain`: PVC 삭제 후 실제 데이터를 자동 삭제하지 않음
- `nodeAffinity`: 저장소가 존재하는 Worker1 조건
- `local.path`: 실제 Worker 디렉터리 경로

### ★ 반드시 기억

**StorageClass는 저장소 제공 방식, PV는 저장 공간, PVC는 저장 공간 요청이다.**

Local PV는 Worker1에 종속되므로 Worker1 장애 시 다른 Node에서 동일 데이터를 자동 사용할 수 있는 구조가 아니다.

---

## 5. StatefulSet과 Headless Service 연결

### Headless Service

```yaml
metadata:
  name: marble-db-hs

spec:
  clusterIP: None
  selector:
    app: marble-db
```

### StatefulSet

```yaml
spec:
  serviceName: marble-db-hs
  replicas: 1

  selector:
    matchLabels:
      app: marble-db

  template:
    metadata:
      labels:
        app: marble-db
```

### 연결 관계

```text
Headless Service
name: marble-db-hs
        │
        ▼
StatefulSet
serviceName: marble-db-hs
        │
        ▼
Pod: marble-db-0
```

**중요한 두 가지 연결**

1. StatefulSet의 `serviceName` ↔ Headless Service의 이름
2. Service의 `selector` ↔ Pod의 `labels`

Headless Service는 StatefulSet Pod별 안정적인 DNS 식별에 사용한다.

---

## 6. StatefulSet과 Secret 연결

MySQL 인증정보는 Secret으로 관리한다.

Secret 이름:

```text
marble-db-secret
```

사용한 키:

```text
MYSQL_ROOT_PASSWORD
MYSQL_PASSWORD
```

### StatefulSet에서 참조

```yaml
env:
  - name: MYSQL_PASSWORD
    valueFrom:
      secretKeyRef:
        name: marble-db-secret
        key: MYSQL_PASSWORD
```

### 핵심 연결 관계

| 설정 | 의미 |
|---|---|
| secretKeyRef.name | Secret 이름 |
| secretKeyRef.key | Secret 내부의 데이터 키 |
| env.name | 컨테이너에 전달할 환경변수 이름 |

**★ 반드시 기억**

Secret 이름과 키가 일치해야 컨테이너에 비밀번호를 전달할 수 있다.

실습 비밀번호는 운영 환경에서 재사용하지 않으며 비밀번호 파일을 GitHub에 업로드하지 않는다.

---

## 7. StatefulSet과 PVC 연결

### PVC 자동 생성

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: marble-local-sc
      resources:
        requests:
          storage: 5Gi
```

StatefulSet 이름은 `marble-db`, PVC 템플릿 이름은 `data`이다.

생성되는 리소스:

```text
Pod: marble-db-0
PVC: data-marble-db-0
```

### PVC와 MySQL 경로 연결

```yaml
volumeMounts:
  - name: data
    mountPath: /var/lib/mysql
```

`volumeMounts.name`과 `volumeClaimTemplates.metadata.name`이 일치해야 한다.

### ★ 반드시 기억

StatefulSet은 PVC를 생성하고 MySQL 컨테이너가 해당 저장소를 사용할 수 있도록 구성한다.

Pod가 삭제되어도 기존 PVC와 PV가 보존되어 있다면 재생성된 Pod가 같은 데이터를 다시 사용할 수 있다.

**StatefulSet만으로 MySQL 복제·자동 Failover·백업이 구현되는 것은 아니다.**

---

## 8. MySQL Service 연결

WAS가 MySQL에 접근하기 위한 Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: marble-db-svc
  namespace: marble-k8s
spec:
  type: ClusterIP
  selector:
    app: marble-db
  ports:
    - port: 3306
      targetPort: 3306
```

### 연결 구조

```text
Spring Boot WAS
       │
       │ marble-db-svc:3306
       ▼
MySQL ClusterIP Service
       │
       │ selector: app=marble-db
       ▼
MySQL Pod
labels: app=marble-db
```

**★ 반드시 기억**

Service는 Pod 이름이나 StatefulSet 이름이 아니라 Pod의 `labels`를 기준으로 연결한다.

---

## 9. Spring Boot를 컨테이너로 전환

기존 Marble WAS는 VM에서 다음과 같이 실행되었다.

```text
Ubuntu
  └── systemd
        └── marble.service
              └── java -jar app.jar
```

Kubernetes에서는 Docker 이미지로 실행한다.

```text
Docker Image
     │
     ▼
Spring Boot Container
     │
     ▼
Kubernetes Deployment
```

### Dockerfile

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY app.jar /app/app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

### Dockerfile 설명

| 명령어 | 의미 |
|---|---|
| FROM | Java 17 기반 이미지 사용 |
| WORKDIR | 컨테이너 작업 디렉터리 |
| COPY | Spring Boot JAR 복사 |
| EXPOSE | 애플리케이션 포트 문서화 |
| ENTRYPOINT | 컨테이너 시작 시 Java 실행 |

`EXPOSE`만으로 외부 접속 포트가 자동 공개되는 것은 아니다.

### 이미지 생성 및 업로드

```bash
docker build -t YOUR_DOCKERHUB_ID/marble-app:v1 .

docker push YOUR_DOCKERHUB_ID/marble-app:v1
```

`YOUR_DOCKERHUB_ID`는 실제 이미지 저장소 계정으로 변경한다.

**★ 반드시 기억**

기존에는 JAR 파일을 VM에 복사했지만 Kubernetes에서는 컨테이너 이미지를 배포한다.

추후 Jenkins CI/CD에서 이미지 빌드·업로드·배포를 자동화한다.

---

## 10. WAS ConfigMap과 Secret 연결

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: marble-app-config
  namespace: marble-k8s
data:
  SPRING_DATASOURCE_URL: jdbc:mysql://marble-db-svc:3306/marble
  SPRING_DATASOURCE_USERNAME: marble
  SERVER_PORT: "8080"
```

실제 MySQL JDBC 옵션은 애플리케이션과 DB 인증·TLS 설정에 맞춰 추가한다.

### Deployment에서 ConfigMap 참조

```yaml
envFrom:
  - configMapRef:
      name: marble-app-config
```

### Deployment에서 Secret 참조

```yaml
env:
  - name: SPRING_DATASOURCE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: marble-db-secret
        key: MYSQL_PASSWORD
```

### 전체 관계

```text
ConfigMap
marble-app-config
        │
        │ envFrom.configMapRef
        ▼
Spring Boot Deployment
        ▲
        │ env.secretKeyRef
        │
Secret
marble-db-secret
```

### ★ 반드시 기억

- ConfigMap: DB 주소, 사용자명 등 일반 설정
- Secret: DB 비밀번호
- Spring Boot: 환경변수를 통해 설정 전달받음

---

## 11. WAS Deployment와 Service

### Deployment 핵심 설정

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: marble-was
  namespace: marble-k8s
spec:
  replicas: 1
  selector:
    matchLabels:
      app: marble-was
  template:
    metadata:
      labels:
        app: marble-was
    spec:
      containers:
        - name: marble-app
          image: YOUR_DOCKERHUB_ID/marble-app:v1
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: marble-app-config
          env:
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: marble-db-secret
                  key: MYSQL_PASSWORD
```

이번에는 WAS Pod 1개로 먼저 검증하고, 정상 동작을 확인한 뒤 2개로 확장한다.

### WAS Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: marble-was-svc
  namespace: marble-k8s
spec:
  type: ClusterIP
  selector:
    app: marble-was
  ports:
    - port: 8080
      targetPort: 8080
```

### 연결 관계

```text
WAS Service
selector: app=marble-was
           │
           ▼
WAS Deployment의 Pod
labels: app=marble-was
```

**★ 반드시 기억**

Deployment는 Pod 생성·관리, Service는 Pod에 접근할 안정적인 주소 제공을 담당한다.

---

## 12. 전체 구축 순서

```text
① Kubernetes Node 상태 확인
              ↓
② Worker1 로컬 저장소 준비
              ↓
③ Worker1 Node 라벨 설정
              ↓
④ MySQL Secret 생성
              ↓
⑤ StorageClass 생성
              ↓
⑥ PV 생성
              ↓
⑦ Headless Service 생성
              ↓
⑧ MySQL ClusterIP Service 생성
              ↓
⑨ MySQL StatefulSet 생성
              ↓
⑩ MySQL 접속 및 데이터 저장 확인
              ↓
⑪ Spring Boot Docker 이미지 준비
              ↓
⑫ ConfigMap 생성
              ↓
⑬ WAS Deployment 생성
              ↓
⑭ WAS Service 생성
              ↓
⑮ /health 및 /health/db 검증
```

---

## 13. 실습 핵심 명령어

### ① 클러스터 상태 확인

```bash
kubectl get nodes -o wide

kubectl get pods -n marble-k8s
```

### ② DB 리소스 생성

```bash
kubectl apply -f 01-db/storageclass.yaml

kubectl apply -f 01-db/pv.yaml

kubectl apply -n marble-k8s -f 01-db/mysql-headless.yaml

kubectl apply -n marble-k8s -f 01-db/mysql-service.yaml

kubectl apply -n marble-k8s -f 01-db/mysql-statefulset.yaml
```

Secret은 사전에 로컬 비밀번호 파일에서 생성한다.

### ③ MySQL 배포 확인

```bash
kubectl rollout status statefulset/marble-db \
  -n marble-k8s --timeout=300s

kubectl get statefulset,pods,pvc -n marble-k8s

kubectl get pv
```

**목표 상태**

```text
StatefulSet: marble-db       READY 1/1
Pod:         marble-db-0     Running
PVC:         data-marble-db-0 Bound
PV:          marble-db-pv     Bound
```

### ④ MySQL 접속

```bash
kubectl exec -it -n marble-k8s marble-db-0 -- \
  mysql -u marble -p marble
```

MySQL에서 다음 내용을 확인한다.

```sql
SHOW DATABASES;

USE marble;

SHOW TABLES;
```

### ⑤ WAS 리소스 생성

```bash
kubectl apply -f 02-was/app-configmap.yaml

kubectl apply -f 02-was/app-deployment.yaml

kubectl apply -f 02-was/app-service.yaml
```

### ⑥ WAS 배포 확인

```bash
kubectl rollout status deployment/marble-was \
  -n marble-k8s --timeout=300s

kubectl get deployments,pods,svc -n marble-k8s

kubectl logs -n marble-k8s \
  deployment/marble-was --tail=100
```

---

## 14. Spring Boot 동작 검증

### 검증할 엔드포인트

| 엔드포인트 | 확인 목적 |
|---|---|
| /health | 애플리케이션 기본 응답 |
| /health/db | MySQL 연결 |

### Control Plane에서 포트포워딩

```bash
kubectl port-forward -n marble-k8s \
  svc/marble-was-svc 18080:8080
```

이 명령은 기본적으로 **실행한 서버의 localhost에서만 접속**을 받는다.

따라서 Control Plane에서 실행했다면 같은 서버에서 확인한다.

```bash
curl http://localhost:18080/health

curl http://localhost:18080/health/db
```

### Windows Chrome에서 확인

Windows PC에서 Control Plane의 IP에 직접 접속하려면 VMware 실습망에서만 다음과 같이 실행할 수 있다.

```bash
kubectl port-forward \
  --address 0.0.0.0 \
  -n marble-k8s \
  svc/marble-was-svc 18080:8080
```

Chrome 주소:

```text
http://<Control-Plane-IP>:18080/
http://<Control-Plane-IP>:18080/health
http://<Control-Plane-IP>:18080/health/db
```

Control Plane의 실제 IP로 변경해야 하며, Windows에서 해당 IP에 접근할 수 있어야 한다.

`--address 0.0.0.0`은 외부 인터페이스에도 포트를 열므로 신뢰할 수 있는 격리된 실습망에서만 사용하고 테스트 후 종료한다.

**★ 반드시 기억**

- `/health` 정상: Spring Boot 애플리케이션 기본 동작 확인
- `/health/db` 정상: Spring Boot에서 MySQL 연결 확인
- 10장 3교시부터 Nginx WEB을 경유하여 접속하도록 구성

---

## 15. 장애 대응 핵심 정리

| 증상 | 주요 원인 | 확인 대상 |
|---|---|---|
| PVC Pending | StorageClass·PV 조건 불일치 | PVC/PV 상세 정보 |
| Pod Pending | Node 자원·nodeAffinity | Pod Events |
| MySQL CrashLoopBackOff | 권한·초기화·설정 오류 | MySQL 로그 |
| Secret 오류 | 이름·키 불일치 | Pod describe |
| WAS ImagePullBackOff | 이미지 이름·접근 권한 오류 | Pod Events |
| WAS CrashLoopBackOff | Spring Boot 실행·설정 오류 | WAS 로그 |
| /health/db 실패 | DB 주소·계정·비밀번호·네트워크 | WAS 및 MySQL 로그 |

### 장애 분석 명령어

```bash
# 전체 상태
kubectl get pods,pvc,svc -n marble-k8s

# PV 상태
kubectl get pv

# MySQL 상세
kubectl describe pod marble-db-0 -n marble-k8s

# PVC 상세
kubectl describe pvc data-marble-db-0 -n marble-k8s

# MySQL 로그
kubectl logs marble-db-0 -n marble-k8s --tail=100

# WAS 로그
kubectl logs deployment/marble-was \
  -n marble-k8s --tail=100

# Events
kubectl get events -n marble-k8s \
  --sort-by=.lastTimestamp
```

### 주의사항

Local PV 권한 문제 발생 시 무조건 `chmod 777`을 적용하지 않는다.

MySQL 이미지의 실제 사용자 UID/GID, Worker 저장소 권한, 로그를 확인하여 필요한 권한만 수정한다.

MySQL 초기화 환경변수는 일반적으로 빈 데이터 디렉터리를 처음 초기화할 때 적용된다. 기존 데이터가 있다면 환경변수를 수정해도 DB 사용자나 비밀번호가 자동 변경되지는 않는다.

---

## 16. 전체 YAML 연결 관계 — 시험·면접 핵심

| 시작 리소스 | 설정 | 연결 대상 |
|---|---|---|
| StatefulSet | serviceName | Headless Service |
| StatefulSet | secretKeyRef.name | Secret |
| StatefulSet | volumeClaimTemplates | PVC |
| PVC 템플릿 | storageClassName | StorageClass |
| PV | storageClassName | StorageClass |
| PV | nodeAffinity | Worker Node |
| MySQL 컨테이너 | volumeMounts.name | PVC 템플릿 이름 |
| MySQL Service | selector | MySQL Pod labels |
| WAS Deployment | configMapRef.name | ConfigMap |
| WAS Deployment | secretKeyRef.name | Secret |
| Spring Boot | SPRING_DATASOURCE_URL | MySQL Service |
| WAS Service | selector | WAS Pod labels |

### ★ 반드시 기억

**Kubernetes 리소스는 YAML 파일명이 아니라 리소스 이름, 참조 필드, selector/labels 및 Service DNS로 연결된다.**

---

## 17. 면접 핵심 질문

**Q1. 기존 Spring Boot JAR를 Kubernetes에서 어떻게 실행하나요?**

Spring Boot JAR를 포함한 컨테이너 이미지를 생성하고 Registry에 업로드한 뒤 Deployment에서 해당 이미지를 사용해 Pod를 실행합니다.

**Q2. 왜 MySQL은 StatefulSet을 사용하나요?**

안정적인 Pod 식별자와 저장소를 관리하기 적합하기 때문입니다.

**Q3. Spring Boot는 MySQL과 어떻게 연결되나요?**

MySQL Service DNS인 `marble-db-svc:3306`을 JDBC 주소로 사용합니다.

**Q4. 왜 Secret과 ConfigMap을 나누어 사용하나요?**

일반 설정은 ConfigMap으로 관리하고 비밀번호 등 민감정보는 Secret으로 관리하기 위해서입니다.

**Q5. Pod를 삭제해도 MySQL 데이터를 유지하려면 어떻게 해야 하나요?**

PVC/PV를 통해 영구 저장소를 연결하고, 재생성된 Pod가 기존 저장소를 다시 사용하도록 구성해야 합니다.

**Q6. WAS Pod가 Running인데 DB 연결이 안 된다면 무엇을 확인하나요?**

Spring Boot의 DB URL, Secret, MySQL Service, Pod 상태, 로그, 네트워크 연결을 확인합니다.

**Q7. 기존 Jenkins와 Kubernetes 배포 방식의 차이는 무엇인가요?**

기존에는 JAR를 서버로 복사했지만 Kubernetes에서는 컨테이너 이미지를 빌드·등록하고 Deployment를 업데이트하는 방식으로 배포합니다.

---

## 18. 최종 핵심 암기

| 개념 | 한 줄 암기 |
|---|---|
| StatefulSet | MySQL Pod 및 저장소 관리 |
| Headless Service | Pod별 DNS 식별 |
| StorageClass | 저장소 제공 방식 |
| PV | 실제 저장 공간 |
| PVC | 저장 공간 요청 |
| volumeClaimTemplates | Pod별 PVC 생성 |
| Secret | 비밀번호 관리 |
| ConfigMap | 일반 설정 관리 |
| Dockerfile | 컨테이너 이미지 빌드 명세 |
| Deployment | Spring Boot Pod 관리 |
| ClusterIP Service | 클러스터 내부 통신 |
| port-forward | 로컬 포트로 Service/Pod 접근 |

### ★ 반드시 기억할 7문장

1. **MySQL은 StatefulSet과 PVC/PV를 사용해 운영한다.**
2. **StatefulSet의 serviceName은 Headless Service 이름과 연결된다.**
3. **volumeClaimTemplates는 Pod별 PVC를 생성한다.**
4. **Local PV는 특정 Worker Node의 저장소에 의존한다.**
5. **Spring Boot는 MySQL Service DNS로 DB에 접속한다.**
6. **ConfigMap은 일반 설정, Secret은 인증정보 관리에 사용한다.**
7. **Kubernetes에서는 JAR를 VM에 복사하는 대신 컨테이너 이미지로 배포한다.**

### 이번 교시 핵심 한 문장

> 기존 Marble의 MySQL과 Spring Boot를 Kubernetes 환경으로 전환하기 위해 MySQL StatefulSet과 PVC/PV로 영구 저장소를 구성하고, Spring Boot를 컨테이너 이미지로 배포한 뒤 ConfigMap·Secret·Service를 통해 DB 연결과 애플리케이션 상태를 검증한다.

---

## 19. 학습 진행 현황

- 학습 범위: **10장 2교시 — MySQL StatefulSet + Marble WAS 구축**
- 객관식 확인문제: **8/8 정답 (100점)**
- 10장 1~2교시 확인문제: **16/16 정답**
- 이론 및 구축 절차 학습: 완료
- 실제 Kubernetes 배포와 `/health/db` 성공 여부: 별도 검증 필요
- 정규 과정 진행: **19/26교시**
- 남은 정규 과정: **7교시**
- 다음 수업: **10장 3교시 — Nginx WEB 구축 및 WEB → WAS → DB 통합**

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/262d6588-951a-4d82-a3f5-31d8cb872474" />

