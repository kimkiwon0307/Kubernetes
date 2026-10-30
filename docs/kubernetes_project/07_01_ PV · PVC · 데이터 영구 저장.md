# Kubernetes 프로젝트 학습

## 7장 1교시 — PV · PVC · 데이터 영구 저장

### 1. Kubernetes에서 영구 저장소가 필요한 이유

Kubernetes의 Pod는 장애, 업데이트, 재배포 등에 따라 삭제되거나 새로 생성될 수 있다.

Pod 내부 컨테이너의 쓰기 가능한 계층에만 데이터를 저장하면 컨테이너가 교체되었을 때 기존 데이터가 사라질 수 있다.

**예시: Marble MySQL**

```text
Spring Boot Pod
       ↓
    MySQL Pod
       ↓
회원정보 · 주문정보 · 상품정보 저장
```

MySQL Pod가 교체될 때 데이터까지 사라진다면 서비스 운영에 문제가 발생한다.

따라서 **Pod의 생명주기와 데이터를 저장하는 공간을 분리**해야 한다.

**★ 반드시 기억: Pod는 교체될 수 있으므로 중요한 데이터는 Pod와 별개로 관리되는 저장소에 보관해야 한다.**

---

### 2. emptyDir이란?

**emptyDir은 Pod의 생명주기에 연결된 임시 저장 공간이다.**

```text
Pod 생성
   ↓
emptyDir 생성
   ↓
컨테이너가 파일 저장
   ↓
Pod 삭제
   ↓
emptyDir 데이터 삭제
```

#### emptyDir의 특징

- Pod가 생성될 때 사용할 수 있는 임시 저장 공간이다.
- 같은 Pod 안에서 컨테이너만 재시작하면 일반적으로 데이터가 유지된다.
- Pod 자체가 제거되면 데이터도 삭제된다.
- 임시 작업 파일이나 캐시 등에 사용할 수 있다.

#### emptyDir YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: demo
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - "while true; do sleep 3600; done"
      volumeMounts:
        - name: temp-data
          mountPath: /data
  volumes:
    - name: temp-data
      emptyDir: {}
```

| 설정 | 의미 |
|---|---|
| volumes | Pod에서 사용할 볼륨 정의 |
| emptyDir | 임시 저장소 사용 |
| volumeMounts | 컨테이너에 볼륨 연결 |
| mountPath | 컨테이너 내부에서 볼륨을 사용할 경로 |

**★ 반드시 기억: emptyDir은 Pod가 삭제되면 데이터도 사라진다.**

---

### 3. PV(PersistentVolume)란?

**PV는 Kubernetes 클러스터에서 사용할 수 있는 저장 공간을 나타내는 리소스이다.**

PV가 가리키는 실제 저장소는 다음과 같이 다양하다.

- Worker Node의 로컬 디스크
- NFS 공유 저장소
- 클라우드 블록 스토리지
- CSI 기반 스토리지

PV는 Namespace에 속하지 않는 **클러스터 범위 리소스**이다.

**★ 반드시 기억: PV = Kubernetes에서 사용할 저장 공간**

---

### 4. PVC(PersistentVolumeClaim)란?

**PVC는 Pod가 사용할 저장 공간을 요청하는 리소스이다.**

PVC를 생성할 때 필요한 저장 용량과 접근 방식 등을 정의한다.

PV와 달리 PVC는 Namespace에 속한다.

#### PV와 PVC의 관계

```text
       Pod
        │
        │ 데이터를 저장하고 싶다
        ▼
       PVC
        │
        │ 저장 공간 1Gi 요청
        ▼
        PV
        │
        │ 저장 공간 제공
        ▼
     실제 디스크
```

| 리소스 | 역할 |
|---|---|
| Pod | 데이터를 사용하는 애플리케이션 |
| PVC | 저장 공간을 요청 |
| PV | 저장 공간을 나타냄 |
| 실제 스토리지 | 파일이 실제 저장되는 장소 |

**★ 반드시 기억: Pod → PVC → PV → 실제 저장소**

---

### 5. emptyDir과 PV/PVC 비교

| 구분 | emptyDir | PV + PVC |
|---|---|---|
| 목적 | 임시 데이터 저장 | Pod와 별도로 저장소 관리 |
| Pod 삭제 | 데이터 삭제 | 같은 저장소 재연결 시 데이터 유지 가능 |
| 컨테이너 재시작 | 일반적으로 유지 | 유지 |
| 활용 | 캐시, 임시 파일 | DB 데이터, 영구 파일 |
| Node 장애 대응 | 보장하지 않음 | 스토리지 종류에 따라 다름 |

#### 핵심 차이

```text
[emptyDir]

Pod A
 └── emptyDir
       └── test.txt
          ↓
       Pod 삭제
          ↓
       파일 사라짐


[PV + PVC]

Pod A
 └── PVC
      └── PV
           └── marble.txt
                 ↓
              Pod 삭제
                 ↓
             Pod B 생성
                 ↓
             같은 PVC 연결
                 ↓
           marble.txt 유지
```

**주의:** PV/PVC 자체가 데이터 복제나 백업을 제공하는 것은 아니다. 실제 스토리지가 정상이고 기존 데이터에 다시 접근할 수 있어야 한다.

---

### 6. local PV란?

**local PV는 특정 Node에 연결된 로컬 디스크나 디렉터리를 사용하는 PV이다.**

이번 VMware 실습에서는 Worker1의 디렉터리를 사용했다.

```text
Kubernetes Cluster
       │
       ├── Control Plane
       │
       ├── Worker1
       │     ├── Pod
       │     └── /mnt/k8s-study-ch07
       │            └── marble.txt
       │
       └── Worker2
```

#### local PV의 특징

- 특정 Worker Node의 실제 저장소를 사용한다.
- `nodeAffinity`로 저장소가 있는 Node를 지정한다.
- Pod 삭제 후 같은 저장소에 다시 연결하면 데이터가 유지될 수 있다.
- Worker의 디스크가 고장 나면 데이터에 접근하지 못할 수 있다.
- 자동 복제나 백업 기능은 없다.

**★ 반드시 기억: local PV는 특정 Node에 종속되므로 Node 장애까지 자동으로 해결하지 못한다.**

---

### 7. PV YAML 핵심 구조

파일: `local-pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ch07-local-pv

spec:
  capacity:
    storage: 1Gi

  volumeMode: Filesystem

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""

  local:
    path: /mnt/k8s-study-ch07

  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: lab.example.com/ch07-storage
              operator: In
              values:
                - "true"
```

#### 주요 설정

| 설정 | 의미 |
|---|---|
| kind: PersistentVolume | PV 생성 |
| capacity.storage | 선언된 저장 용량 |
| volumeMode: Filesystem | 파일시스템 형태 사용 |
| ReadWriteOnce | 한 Node에서 읽기·쓰기 마운트 |
| Retain | PVC 삭제 후 데이터 보존 정책 |
| storageClassName: "" | StorageClass 없이 정적 PV 사용 |
| local.path | 실제 저장 경로 |
| nodeAffinity | 저장소를 사용할 Node 조건 |

**주의:** local PV에서 `storage: 1Gi`를 선언했다고 실제 디렉터리에 1Gi 디스크 사용량 제한이 자동 적용되는 것은 아니다.

---

### 8. PVC YAML 핵심 구조

파일: `local-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ch07-data-pvc

spec:
  storageClassName: ""
  volumeName: ch07-local-pv

  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 1Gi
```

#### 주요 설정

| 설정 | 의미 |
|---|---|
| kind: PersistentVolumeClaim | PVC 생성 |
| volumeName | 연결할 PV 지정 |
| accessModes | 요청할 접근 방식 |
| requests.storage | 필요한 저장 용량 |
| storageClassName: "" | StorageClass를 사용하지 않는 요청 |

PV와 PVC는 용량, 접근 모드, StorageClass 등의 조건을 충족해야 연결될 수 있다.

---

### 9. ReadWriteOnce(RWO)란?

**ReadWriteOnce는 하나의 Node에서 읽기·쓰기 방식으로 마운트할 수 있다는 의미이다.**

**Pod 한 개만 사용할 수 있다는 뜻은 아니다.**

#### 접근 모드 비교

| 모드 | 의미 |
|---|---|
| RWO (ReadWriteOnce) | 한 Node에서 읽기·쓰기 마운트 |
| ROX (ReadOnlyMany) | 여러 Node에서 읽기 전용 마운트 |
| RWX (ReadWriteMany) | 여러 Node에서 읽기·쓰기 마운트 |
| RWOP (ReadWriteOncePod) | 지원되는 CSI 환경에서 하나의 Pod만 읽기·쓰기 사용 |

실제 지원 여부는 스토리지 종류와 드라이버에 따라 달라진다.

**★ 반드시 기억: RWO는 Node 기준이지 Pod 한 개라는 뜻이 아니다.**

---

### 10. Pod에서 PVC 사용하는 방법

파일: `pv-demo-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ch07-pv-demo

spec:
  automountServiceAccountToken: false

  securityContext:
    runAsUser: 1000
    runAsGroup: 1000

  containers:
    - name: demo
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - "while true; do sleep 3600; done"

      volumeMounts:
        - name: data-volume
          mountPath: /data

  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: ch07-data-pvc
```

#### 핵심 설정

```yaml
volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: ch07-data-pvc
```

Pod가 사용할 PVC를 지정한다.

```yaml
volumeMounts:
  - name: data-volume
    mountPath: /data
```

PVC에 연결된 저장 공간을 컨테이너의 `/data`에 마운트한다.

`runAsUser: 1000`과 `runAsGroup: 1000`은 이번 실습에서 준비한 저장 디렉터리의 소유권과 일치시키기 위한 설정이다.

---

### 11. 실습 명령어 핵심 정리

#### ① Namespace 준비 — Control Plane

```bash
kubectl get nodes -o wide
kubectl get namespace storage-lab --ignore-not-found
kubectl create namespace storage-lab
```

이미 Namespace가 존재한다면 마지막 생성 명령을 생략한다.

#### ② Worker1에 저장 공간 준비 — Worker1 터미널

```bash
sudo mkdir -p /mnt/k8s-study-ch07

sudo chown 1000:1000 /mnt/k8s-study-ch07

sudo chmod 750 /mnt/k8s-study-ch07

ls -ld /mnt/k8s-study-ch07
```

#### ③ Worker1에 Node 라벨 지정 — Control Plane

`kubectl get nodes -o wide`로 Worker1의 실제 Node 이름을 확인한다.

```bash
kubectl label node worker1 \
  lab.example.com/ch07-storage=true --overwrite
```

`worker1`은 예시이며 실제 Node 이름이 다르면 변경한다.

#### ④ PV 생성

```bash
kubectl apply -f local-pv.yaml

kubectl get pv

kubectl describe pv ch07-local-pv
```

#### ⑤ PVC 생성

```bash
kubectl apply -n storage-lab -f local-pvc.yaml

kubectl get pvc -n storage-lab

kubectl get pv
```

#### ⑥ Pod 생성

```bash
kubectl apply -n storage-lab -f pv-demo-pod.yaml

kubectl wait --for=condition=Ready \
  pod/ch07-pv-demo \
  -n storage-lab \
  --timeout=180s

kubectl get pods -n storage-lab -o wide
```

Pod가 저장소를 준비한 Worker1에 배치되었는지 확인한다.

#### ⑦ 데이터 저장

```bash
kubectl exec -n storage-lab ch07-pv-demo -- \
  sh -c 'echo "Marble Persistent Data" > /data/marble.txt'
```

#### ⑧ 데이터 확인

```bash
kubectl exec -n storage-lab ch07-pv-demo -- \
  cat /data/marble.txt
```

예상 결과:

```text
Marble Persistent Data
```

#### ⑨ Pod 삭제

```bash
kubectl delete pod ch07-pv-demo \
  -n storage-lab --wait=true
```

#### ⑩ Pod 다시 생성

```bash
kubectl apply -n storage-lab -f pv-demo-pod.yaml

kubectl wait --for=condition=Ready \
  pod/ch07-pv-demo \
  -n storage-lab \
  --timeout=180s
```

#### ⑪ 기존 데이터 유지 검증

```bash
kubectl exec -n storage-lab ch07-pv-demo -- \
  cat /data/marble.txt
```

정상적으로 같은 저장소를 재연결했다면 다음 값이 유지된다.

```text
Marble Persistent Data
```

**★ 반드시 기억: Pod를 삭제해도 같은 PVC로 기존 저장소를 다시 연결하면 데이터를 유지할 수 있다.**

---

### 12. PV와 PVC 상태

#### PV 상태

| 상태 | 의미 |
|---|---|
| Available | 아직 PVC와 연결되지 않은 PV |
| Bound | PVC와 연결된 PV |
| Released | PVC가 삭제되어 후속 처리가 필요한 PV |
| Failed | 자동 반환 처리 등에서 실패 발생 |

#### PVC 상태

| 상태 | 의미 |
|---|---|
| Pending | 연결할 저장소를 기다림 |
| Bound | PV와 연결됨 |
| Lost | 연결된 저장소가 손실된 상태 |

**★ 반드시 기억: Bound는 PV와 PVC가 연결되었다는 의미이다.**

단, Bound 상태만으로 실제 디스크의 정상 동작이나 데이터의 안전성이 보장되지는 않는다.

---

### 13. Retain과 Delete 정책

`persistentVolumeReclaimPolicy`는 PVC 삭제 이후 저장소를 어떻게 처리할지 정의한다.

| 정책 | 설명 |
|---|---|
| Retain | PVC 삭제 후 저장소 데이터 보존 및 수동 처리 |
| Delete | 지원하는 스토리지에서 PV와 실제 저장소 삭제 처리 |

#### Retain 예시

```yaml
persistentVolumeReclaimPolicy: Retain
```

PVC가 삭제되어도 데이터를 자동 삭제하지 않는다.

하지만 Retain은 **백업이나 복제 기능이 아니다.**

디스크 자체가 손상되면 데이터를 잃을 수 있다.

**★ 반드시 기억: Retain은 PVC 삭제 후 데이터를 보존하는 정책이며, 디스크 장애까지 보호하지 않는다.**

---

### 14. 장애 대응

**공통 공식: 상태 → 상세 → Events → 원인 → 검증**

#### ① PV/PVC 상태 확인

```bash
kubectl get pv

kubectl get pvc -n storage-lab
```

#### ② PVC 상세 확인

```bash
kubectl describe pvc ch07-data-pvc -n storage-lab
```

#### ③ PV 상세 확인

```bash
kubectl describe pv ch07-local-pv
```

#### ④ Pod 상태 및 Events 확인

```bash
kubectl get pods -n storage-lab -o wide

kubectl describe pod ch07-pv-demo -n storage-lab

kubectl get events -n storage-lab \
  --sort-by=.lastTimestamp
```

#### 대표 장애 상황

| 증상 | 확인 항목 |
|---|---|
| PVC Pending | PV 존재, 용량, AccessMode, StorageClass |
| PV Available 유지 | PVC 조건 및 바인딩 확인 |
| Pod Pending | Worker 상태, PV nodeAffinity |
| ContainerCreating 지속 | 볼륨 마운트 및 Events |
| Permission denied | 저장소 소유권과 컨테이너 UID/GID |
| 재생성 후 파일 없음 | PVC 연결, 실제 경로, 데이터 존재 |
| Worker 장애 | local PV의 Node 종속성 |

**★ 반드시 기억: PVC가 Pending이라면 PV와 PVC의 조건부터 비교한다.**

---

### 15. Marble 프로젝트와 연결

최종 10장에서는 MySQL 데이터 저장에 PVC를 활용한다.

```text
Marble Spring Boot
        │
        ▼
     MySQL Pod
        │
        ▼
      PVC
        │
        ▼
       PV
        │
        ▼
   실제 저장소
```

#### PV/PVC가 필요한 이유

- MySQL 데이터 파일을 Pod와 별도로 저장하기 위해
- MySQL Pod가 재생성되어도 기존 데이터에 다시 접근하기 위해
- 애플리케이션과 저장소의 생명주기를 분리하기 위해

#### 추가로 고려해야 하는 사항

- 디스크 장애 대비 백업
- 저장소의 실제 가용성
- 데이터베이스 복제
- Node 장애 시 복구 전략
- 스토리지 성능 및 접근 모드

**PV/PVC만 구축한다고 데이터베이스 고가용성이 자동으로 완성되지는 않는다.**

---

### 16. 면접 핵심 질문

**Q1. PV란 무엇인가요?**

Kubernetes 클러스터에서 사용할 수 있는 저장 공간을 나타내는 리소스입니다.

**Q2. PVC란 무엇인가요?**

Pod가 필요한 저장 공간의 용량과 접근 방식 등을 요청하는 리소스입니다.

**Q3. emptyDir과 PV/PVC의 차이는 무엇인가요?**

emptyDir은 Pod가 제거되면 데이터도 삭제되는 임시 저장소이고, PV/PVC는 Pod와 별도로 관리되는 저장소를 사용하도록 하는 방식입니다.

**Q4. Bound 상태는 무엇인가요?**

PV와 PVC가 정상적으로 연결되었다는 의미입니다.

**Q5. ReadWriteOnce는 Pod 하나만 사용할 수 있다는 뜻인가요?**

아닙니다. 한 Node에서 읽기·쓰기 방식으로 마운트할 수 있다는 의미입니다.

**Q6. Retain 정책의 역할은 무엇인가요?**

PVC 삭제 후에도 저장소 데이터를 보존하고 관리자가 후속 처리를 할 수 있도록 하는 정책입니다.

**Q7. PVC를 사용하면 Node 장애까지 자동으로 복구되나요?**

아닙니다. 저장소 종류에 따라 다르며, 특히 local PV는 특정 Node에 종속되므로 Node 장애 시 데이터 접근이 불가능할 수 있습니다.

**Q8. MySQL을 Kubernetes에서 운영할 때 PVC가 필요한 이유는 무엇인가요?**

MySQL Pod가 삭제되거나 재생성되더라도 데이터 파일을 별도 저장소에서 유지하고 다시 연결하기 위해서입니다.

---

## 17. 최종 핵심 암기

| 개념 | 한 줄 암기 |
|---|---|
| emptyDir | Pod 삭제 시 데이터도 삭제되는 임시 저장소 |
| PV | Kubernetes에서 사용할 저장 공간 |
| PVC | 저장 공간을 요청하는 리소스 |
| Bound | PV와 PVC가 연결된 상태 |
| RWO | 한 Node에서 읽기·쓰기 마운트 |
| local PV | 특정 Node의 로컬 저장소 |
| nodeAffinity | 저장소가 위치한 Node 조건 |
| Retain | PVC 삭제 후 저장소 데이터 보존 |
| StorageClass | 저장소 제공·프로비저닝 방식 정의 (다음 교시) |

### ★ 반드시 기억할 5문장

1. **emptyDir은 Pod가 삭제되면 데이터도 삭제되는 임시 저장소이다.**
2. **PV는 저장 공간을 나타내고, PVC는 저장 공간을 요청하는 리소스이다.**
3. **Pod는 PVC를 통해 PV에 연결된 실제 저장소를 사용한다.**
4. **Pod가 삭제되어도 기존 PVC로 동일한 저장소를 다시 사용하면 데이터를 유지할 수 있다.**
5. **PV/PVC는 백업이나 Node 장애 복구를 자동으로 보장하지 않는다.**

### 이번 교시 핵심 한 문장

> Kubernetes에서 PV는 사용할 저장 공간을 나타내고 PVC는 이를 요청하며, Pod가 교체되어도 동일한 저장소를 재연결할 수 있도록 하여 애플리케이션 데이터의 지속성을 관리한다.

---

## 학습 결과

- 학습 범위: **7장 1교시 — PV · PVC · 데이터 영구 저장**
- 객관식 확인문제: **8문항 중 8문항 정답 (100점)**
- 전체 강의 진도: **16/26교시 (61.5%)**
- 남은 강의: **10교시**
- 실제 PV/PVC 연결과 데이터 유지 실습 성공 여부: 별도 확인 필요
- 다음 교시: **7장 2교시 — StorageClass · StatefulSet · MySQL 운영**

- <img width="1055" height="1491" alt="image" src="https://github.com/user-attachments/assets/2373dcdf-584d-4d37-8694-7610b32d0a8f" />

