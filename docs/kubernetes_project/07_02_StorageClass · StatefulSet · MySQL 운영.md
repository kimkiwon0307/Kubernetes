# Kubernetes 프로젝트 학습

## 7장 2교시 — StorageClass · StatefulSet · MySQL 운영

### 1. 이번 교시의 최종 목표

**MySQL Pod가 삭제되어 다시 생성되더라도 기존 데이터와 Pod의 식별자를 유지하는 구조를 구축한다.**

MySQL은 회원정보, 주문정보 등 중요한 데이터를 저장하므로 Pod가 교체될 때도 기존 데이터를 다시 사용할 수 있어야 한다.

이를 위해 StatefulSet과 Kubernetes 스토리지 리소스를 함께 사용한다.

---

### 2. 반드시 이해해야 할 핵심 개념 6개

| 리소스 | 역할 | 쉬운 설명 |
|---|---|---|
| PV | 실제 저장소를 나타냄 | 준비된 창고 |
| PVC | 저장 공간 요청 | 창고 사용 신청서 |
| StorageClass | 저장소 제공 방식 정의 | 창고 종류·제공 방식 |
| StatefulSet | 상태 저장 Pod 관리 | Pod 이름·저장소 관리 |
| Headless Service | Pod별 DNS 식별 | Pod의 네트워크 주소 |
| volumeClaimTemplates | Pod별 PVC 생성 | Pod마다 창고 신청서 생성 |

#### ★ 반드시 기억

- **StorageClass = 저장소 제공 방식**
- **StatefulSet = 상태 저장 애플리케이션 관리**
- **Headless Service = Pod별 DNS 식별**
- **volumeClaimTemplates = Pod마다 PVC 생성**

---

### 3. Deployment와 StatefulSet 차이

#### Deployment

주로 Spring Boot, Nginx 같은 무상태(Stateless) 애플리케이션에 사용한다.

```text
Deployment: marble-app
    │
    ├── marble-app-abc12
    └── marble-app-def34

Pod가 교체되면 이름이 바뀔 수 있음
```

#### StatefulSet

Pod별 안정적인 이름과 저장소가 필요한 애플리케이션에 사용한다.

```text
StatefulSet: marble-db
    │
    ├── Pod: marble-db-0
    │     └── PVC: data-marble-db-0
    │
    └── Pod: marble-db-1
          └── PVC: data-marble-db-1
```

#### 비교

| 구분 | Deployment | StatefulSet |
|---|---|---|
| 대표 용도 | 일반 웹·WAS | 상태 저장 애플리케이션 |
| Pod 이름 | 교체 시 변경 가능 | 일정한 이름·번호 유지 |
| Pod별 DNS | 기본적으로 고정 식별자 없음 | Headless Service로 구성 가능 |
| 저장소 | PVC 사용 가능 | Pod별 PVC 자동 생성 가능 |
| 대표 사례 | Spring Boot | MySQL |

**주의: Deployment도 PVC를 사용할 수 있으며, MySQL을 반드시 StatefulSet으로만 실행해야 하는 것은 아니다.**

#### ★ 반드시 기억

**StatefulSet은 Pod가 다시 생성되어도 안정적인 Pod 식별자를 유지한다.**

---

### 4. StatefulSet을 구축하려면 무엇이 필요할까?

이번 MySQL 실습에서는 다음 리소스를 사용한다.

| 리소스 | 필요한 이유 |
|---|---|
| Secret | MySQL 비밀번호 관리 |
| StorageClass | 저장소 제공 방식 정의 |
| PV | Worker1의 저장 공간 등록 |
| Headless Service | MySQL Pod의 DNS 식별 |
| StatefulSet | MySQL Pod 및 PVC 생성·관리 |

**총 5개의 YAML 파일을 직접 작성한다.**

```text
stateful-lab/
│
├── mysql-secret.yaml
│     └── MySQL 비밀번호
│
├── manual-local-sc.yaml
│     └── StorageClass
│
├── mysql-local-pv-0.yaml
│     └── 실제 저장소
│
├── marble-db-hs.yaml
│     └── Headless Service
│
└── marble-mysql-sts.yaml
      └── StatefulSet
            ├── MySQL Pod 생성
            └── PVC 자동 생성
```

**중요:** 이것은 이번 실습의 구성이다. 모든 StatefulSet에 Secret이나 영구 저장소가 반드시 필요한 것은 아니다.

---

### 5. 전체 리소스 연결 관계

```text
[StatefulSet: marble-db]
        │
        ├── MySQL Pod 생성
        │      └── marble-db-0
        │
        ├── Secret 참조
        │      └── marble-mysql-secret
        │
        ├── Headless Service 연결
        │      └── marble-db-hs
        │
        └── volumeClaimTemplates
                │
                ▼
         PVC 자동 생성
         data-marble-db-0
                │
                │ storageClassName: manual-local
                ▼
         PV: mysql-local-pv-0
                │
                │ storageClassName: manual-local
                ▼
        Worker1 실제 저장소
       /mnt/k8s-study-ch07-sts
```

#### 핵심 원리

1. StatefulSet이 MySQL Pod를 생성한다.
2. StatefulSet이 volumeClaimTemplates에 따라 PVC를 생성한다.
3. PVC는 조건에 맞는 PV와 연결된다.
4. Pod는 PVC를 통해 저장소를 사용한다.
5. Secret에서 MySQL 비밀번호를 전달받는다.
6. Headless Service를 통해 Pod별 DNS 이름을 사용할 수 있다.

**★ 반드시 기억: 각각의 리소스가 서로 참조하면서 하나의 서비스를 구성한다.**

---

### 6. StorageClass란?

**StorageClass는 스토리지를 어떤 방식으로 제공할지 정의하는 리소스이다.**

StorageClass 자체가 디스크를 뜻하는 것은 아니다.

#### 이번 실습의 StorageClass

파일: `manual-local-sc.yaml`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual-local
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

#### 설정 설명

| 설정 | 의미 |
|---|---|
| name: manual-local | StorageClass 이름 |
| no-provisioner | 자동 PV 생성 기능 사용 안 함 |
| WaitForFirstConsumer | Pod가 저장소를 필요로 할 때 바인딩 결정 |

이번 실습에서는 PV를 직접 만들어야 한다.

반면 동적 프로비저닝을 지원하는 StorageClass를 사용하면 PVC 요청에 맞춰 PV가 자동 생성될 수 있다.

**★ 반드시 기억: StorageClass가 있다고 언제나 PV가 자동 생성되는 것은 아니다.**

---

### 7. PV와 StorageClass의 연결

파일: `mysql-local-pv-0.yaml`

핵심 YAML:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-local-pv-0
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual-local

  local:
    path: /mnt/k8s-study-ch07-sts

  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: lab.example.com/ch07-storage
              operator: In
              values:
                - "true"
```

#### 핵심 설정

| 설정 | 의미 |
|---|---|
| storage: 2Gi | PV의 선언된 용량 |
| ReadWriteOnce | 한 Node에서 읽기·쓰기 마운트 |
| Retain | PVC 삭제 후 데이터 보존 |
| storageClassName | StorageClass와 연결 |
| local.path | Worker1의 실제 저장 경로 |
| nodeAffinity | 저장소를 사용할 수 있는 Node 조건 |

이번 실습에서는 `lab.example.com/ch07-storage=true` 라벨을 실제 저장소가 있는 Worker1에만 지정한다.

#### 연결 관계

```text
StorageClass
name: manual-local
        ↕
PV
storageClassName: manual-local
```

두 이름이 일치해야 한다.

---

### 8. Headless Service란?

**Headless Service는 대표 ClusterIP를 생성하지 않고 Pod별 네트워크 주소를 식별하는 데 사용하는 Service이다.**

파일: `marble-db-hs.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: marble-db-hs
spec:
  clusterIP: None
  selector:
    app: marble-db
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
```

#### 핵심 설정

| 설정 | 의미 |
|---|---|
| clusterIP: None | Headless Service 지정 |
| selector | 연결할 Pod의 라벨 선택 |
| port: 3306 | MySQL 통신 포트 |

#### StatefulSet과의 연결

Headless Service:

```yaml
metadata:
  name: marble-db-hs
```

StatefulSet:

```yaml
spec:
  serviceName: marble-db-hs
```

**serviceName과 Headless Service 이름이 연결된다.**

또한 Service의 `selector`와 Pod의 `labels`가 일치해야 한다.

Pod별 DNS 예시:

```text
marble-db-0.marble-db-hs.stateful-lab.svc.cluster.local
```

**★ 반드시 기억: Headless Service는 StatefulSet Pod별 DNS 식별에 사용한다.**

---

### 9. StatefulSet의 핵심 YAML

파일: `marble-mysql-sts.yaml`

전체 YAML을 한 번에 암기하기보다는 중요한 연결 부분만 이해한다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: marble-db

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

    spec:
      containers:
        - name: mysql
          image: mysql:8.0

          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: marble-mysql-secret
                  key: MYSQL_ROOT_PASSWORD

          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql

  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: manual-local
        resources:
          requests:
            storage: 2Gi
```

이 YAML은 리소스 연결을 학습하기 위한 핵심 발췌본이다. 실제 실습에서는 MySQL 데이터베이스 환경변수, 상태 확인 및 권한 설정 등이 추가될 수 있다.

#### 중요한 설정 5개

**① StatefulSet 이름**

```yaml
metadata:
  name: marble-db
```

생성되는 Pod 이름:

```text
marble-db-0
```

**② Headless Service 연결**

```yaml
serviceName: marble-db-hs
```

**③ Pod 라벨 연결**

```yaml
selector:
  matchLabels:
    app: marble-db
```

```yaml
template:
  metadata:
    labels:
      app: marble-db
```

**④ Secret 연결**

```yaml
secretKeyRef:
  name: marble-mysql-secret
  key: MYSQL_ROOT_PASSWORD
```

**⑤ PVC 자동 생성**

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
```

자동 생성되는 PVC:

```text
data-marble-db-0
```

**★ 반드시 기억: volumeClaimTemplates는 StatefulSet의 Pod마다 PVC를 생성한다.**

---

### 10. 가장 중요한 YAML 연결 관계 정리

이번 교시에서 반드시 복습해야 할 표이다.

| YAML 설정 | 연결 대상 | 일치해야 할 값 |
|---|---|---|
| PV의 storageClassName | StorageClass | manual-local |
| volumeClaimTemplates의 storageClassName | StorageClass | manual-local |
| StatefulSet의 serviceName | Headless Service | marble-db-hs |
| Service의 selector | Pod의 labels | app: marble-db |
| StatefulSet의 matchLabels | Pod의 labels | app: marble-db |
| secretKeyRef.name | Secret 이름 | marble-mysql-secret |
| secretKeyRef.key | Secret 데이터 키 | MYSQL_ROOT_PASSWORD |
| volumeMounts.name | volumeClaimTemplates 이름 | data |

**주의:** 모든 이름을 똑같이 만드는 것이 아니라, 서로 참조하는 설정끼리 일치시켜야 한다.

#### ★ 반드시 기억

**Kubernetes YAML을 이해한다는 것은 각 설정의 의미뿐 아니라 다른 리소스와 어디서 연결되는지 이해하는 것이다.**

---

### 11. 구축 순서 정리

이번 실습에서 사용한 순서이다.

```text
① Namespace 생성
       ↓
② Worker1 저장 디렉터리 준비
       ↓
③ Secret 생성
       ↓
④ StorageClass 생성
       ↓
⑤ PV 생성
       ↓
⑥ Headless Service 생성
       ↓
⑦ StatefulSet 생성
       ↓
⑧ MySQL Pod와 PVC 생성 확인
       ↓
⑨ MySQL 데이터 저장
       ↓
⑩ Pod 삭제 후 데이터 유지 검증
```

이것은 실습을 이해하기 편하게 정한 순서이며, Kubernetes의 모든 리소스에 항상 동일한 생성 순서가 강제되는 것은 아니다.

---

### 12. 실습 핵심 명령어

#### ① Namespace

```bash
kubectl create namespace stateful-lab
```

이미 존재하면 생성하지 않는다.

#### ② StorageClass

```bash
kubectl apply -f manual-local-sc.yaml

kubectl get storageclass
```

#### ③ PV

```bash
kubectl apply -f mysql-local-pv-0.yaml

kubectl get pv
```

#### ④ Secret

```bash
kubectl apply -n stateful-lab -f mysql-secret.yaml

kubectl get secrets -n stateful-lab
```

**주의:** 실제 비밀번호가 들어 있는 Secret YAML은 GitHub에 업로드하지 않는다. 실무에서는 별도의 비밀정보 관리 방식과 접근 권한 설정을 적용한다.

#### ⑤ Headless Service

```bash
kubectl apply -n stateful-lab -f marble-db-hs.yaml

kubectl get svc -n stateful-lab
```

#### ⑥ StatefulSet

```bash
kubectl apply -n stateful-lab -f marble-mysql-sts.yaml

kubectl get statefulsets -n stateful-lab

kubectl get pods -n stateful-lab

kubectl get pvc -n stateful-lab
```

#### ⑦ 배포 상태 확인

```bash
kubectl rollout status statefulset/marble-db \
  -n stateful-lab --timeout=300s
```

#### 예상 결과

```text
StatefulSet: marble-db
Pod:         marble-db-0
PVC:         data-marble-db-0
PV:          mysql-local-pv-0
```

PV와 PVC가 `Bound`, Pod가 `Running/Ready`인지 확인한다.

`WaitForFirstConsumer` 방식에서는 PVC를 생성한 직후 Pod가 아직 없다면 `Pending` 상태가 정상일 수 있다.

---

### 13. StatefulSet과 데이터 유지

#### 실습 목표

MySQL에 데이터를 저장한 뒤 Pod를 삭제하여 기존 데이터가 유지되는지 확인한다.

```text
MySQL Pod: marble-db-0
        │
        ▼
PVC: data-marble-db-0
        │
        ▼
PV: mysql-local-pv-0
        │
        ▼
Worker1 디스크
        │
        ▼
MySQL 데이터 저장
```

Pod를 삭제한다.

```bash
kubectl delete pod marble-db-0 \
  -n stateful-lab --wait=true
```

StatefulSet은 원하는 Pod 수를 유지하도록 같은 이름의 Pod를 다시 생성한다.

```bash
kubectl get pods -n stateful-lab

kubectl get pvc -n stateful-lab
```

기존 PVC와 저장소가 정상적으로 유지되었다면 새 Pod에서 기존 MySQL 데이터를 다시 사용할 수 있다.

#### ★ 반드시 기억

**StatefulSet은 Pod의 안정적인 식별자를 유지하고, 기존 PVC가 보존되면 재생성된 Pod가 같은 저장소를 다시 사용할 수 있다.**

---

### 14. StatefulSet의 한계

StatefulSet을 사용한다고 다음 기능이 자동으로 만들어지지는 않는다.

- MySQL Primary/Replica 복제
- 자동 Failover
- 데이터베이스 백업
- 디스크 장애 복구
- 무중단 데이터베이스 운영

특히 local PV는 특정 Worker Node에 의존한다.

Worker1이 영구적으로 고장 나면 데이터를 사용할 수 없을 수 있다.

**★ 반드시 기억: StatefulSet은 DB 고가용성 전체를 자동으로 완성하는 기능이 아니다.**

---

### 15. 장애 대응

공통 공식:

**상태 → 상세 → Events/로그 → 원인 → 검증**

| 증상 | 우선 확인 |
|---|---|
| PVC Pending | StorageClass, PV 용량, AccessMode, 바인딩 시점 |
| Pod Pending | Node 상태, PV nodeAffinity, 스케줄링 Events |
| MySQL 실행 실패 | Secret, MySQL 초기화 로그 |
| Permission denied | Worker 디렉터리 권한, 컨테이너 UID/GID |
| DNS 연결 실패 | Headless Service, selector, Pod labels |
| 데이터가 사라짐 | 기존 PVC와 PV 연결 여부 |

#### 핵심 명령어

```bash
# 상태
kubectl get pods,pvc -n stateful-lab
kubectl get pv

# 상세
kubectl describe pod marble-db-0 -n stateful-lab
kubectl describe pvc data-marble-db-0 -n stateful-lab

# 로그
kubectl logs marble-db-0 -n stateful-lab

# Events
kubectl get events -n stateful-lab \
  --sort-by=.lastTimestamp

# 최종 검증
kubectl rollout status statefulset/marble-db \
  -n stateful-lab
```

참고로 MySQL 이미지의 실행 UID/GID는 이미지와 버전에 따라 확인해야 한다. `999:999`가 모든 이미지에서 동일하다고 가정하지 않는다.

---

### 16. 면접 핵심 질문

**Q1. StorageClass란 무엇인가요?**

Kubernetes에서 스토리지의 제공 방식과 종류를 정의하는 리소스입니다.

**Q2. Deployment와 StatefulSet의 차이는 무엇인가요?**

Deployment는 주로 무상태 애플리케이션을 관리하고, StatefulSet은 안정적인 Pod 식별자와 저장소가 필요한 상태 저장 애플리케이션에 적합합니다.

**Q3. Headless Service는 왜 사용하나요?**

대표 ClusterIP 없이 Pod별 DNS 이름을 사용하도록 하기 위해 구성합니다.

**Q4. volumeClaimTemplates는 무엇인가요?**

StatefulSet이 Pod마다 고정된 PVC를 자동 생성하도록 하는 설정입니다.

**Q5. StatefulSet에서 Pod가 삭제되면 어떻게 되나요?**

StatefulSet이 같은 이름의 Pod를 재생성하며, 기존 PVC와 저장소가 보존되어 있다면 같은 데이터를 다시 사용할 수 있습니다.

**Q6. StatefulSet을 사용하면 DB 복제와 자동 Failover가 구현되나요?**

아닙니다. 데이터베이스 복제, 백업, 자동 장애조치 등은 별도로 설계해야 합니다.

**Q7. YAML 파일끼리 어떻게 연결되나요?**

Service의 selector와 Pod labels, StatefulSet의 serviceName과 Service 이름, storageClassName과 StorageClass 이름처럼 서로 참조하는 설정값을 연결하여 동작합니다.

---

## 17. 최종 핵심 암기

| 개념 | 반드시 기억 |
|---|---|
| StorageClass | 저장소 제공 방식 |
| PV | 저장 공간 |
| PVC | 저장 공간 요청 |
| StatefulSet | 안정적인 Pod 이름과 저장소 관리 |
| Headless Service | Pod별 DNS 식별 |
| volumeClaimTemplates | Pod별 PVC 자동 생성 |
| serviceName | StatefulSet에서 사용할 Service 이름 |
| selector / labels | Service와 Pod 또는 컨트롤러와 Pod 연결 |
| secretKeyRef | Secret 참조 |
| nodeAffinity | PV를 사용할 수 있는 Node 조건 |

### ★ 반드시 기억할 5문장

1. **StorageClass는 저장소를 제공하는 방식을 정의한다.**
2. **StatefulSet은 상태 저장 애플리케이션의 안정적인 Pod 식별자와 저장소를 관리한다.**
3. **Headless Service는 StatefulSet의 Pod별 DNS 식별에 사용한다.**
4. **volumeClaimTemplates를 이용하면 StatefulSet의 Pod마다 PVC를 자동 생성할 수 있다.**
5. **StatefulSet, Headless Service, Secret, PV, PVC는 서로 참조하는 YAML 설정을 통해 연결된다.**

### 이번 교시 핵심 한 문장

> StatefulSet은 MySQL 같은 상태 저장 애플리케이션의 Pod를 안정적으로 관리하고, Headless Service로 Pod별 네트워크 식별자를 제공하며, volumeClaimTemplates와 PV/PVC를 통해 Pod가 재생성되어도 기존 저장소를 다시 사용할 수 있도록 구성한다.

---

## 학습 결과

- 학습 범위: **7장 2교시 — StorageClass · StatefulSet · MySQL 운영**
- 강의: 학습 완료
- 객관식 확인문제: 아직 미채점
- 실제 MySQL 구축·데이터 유지 검증: 별도 확인 필요
- 전체 강의 진도: **17/26교시 (65.4%)**
- 남은 강의: **9교시**
- 다음 수업: **8장 1교시 — Probe · Health Check**

<img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/feae5473-92c5-4ed7-bf4b-a329e770720b" />


<img width="941" height="1672" alt="image" src="https://github.com/user-attachments/assets/e275934c-712d-405a-b900-05842852e9bc" />
