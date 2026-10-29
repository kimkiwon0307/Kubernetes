# Kubernetes 프로젝트 학습

## 6장 2교시 — Secret · 환경변수 · 설정 관리

### 1. Secret이란?

**Secret은 Kubernetes에서 비밀번호, API 키, 토큰 등 민감정보를 관리하는 리소스이다.**

#### Secret이 필요한 이유

- DB 비밀번호를 Deployment YAML에 직접 작성하지 않기 위해
- 애플리케이션과 민감정보를 분리하기 위해
- 동일한 컨테이너 이미지에 환경별 접속 정보를 전달하기 위해
- 민감정보에 대한 접근 권한을 별도로 관리하기 위해

**예시**

```text
Kubernetes Cluster
       │
       ├── ConfigMap
       │     ├── APP_ENV=development
       │     └── DB_HOST=mysql-svc
       │
       ├── Secret
       │     ├── DB_USERNAME
       │     └── DB_PASSWORD
       │
       └── Spring Boot Pod
             ├── 일반 설정은 ConfigMap 참조
             └── 민감정보는 Secret 참조
```

**★ 반드시 기억: Secret은 민감정보를 애플리케이션과 분리하여 관리하는 Kubernetes 리소스이다.**

---

### 2. ConfigMap과 Secret 차이

| 구분 | ConfigMap | Secret |
|---|---|---|
| 목적 | 일반 설정 관리 | 민감정보 관리 |
| 예시 | APP_ENV, DB_HOST | DB_PASSWORD, API_KEY |
| 전달 방법 | 환경변수·파일 | 환경변수·파일 |
| Namespace | 속함 | 속함 |
| 보안 | 민감정보 저장 부적절 | 접근 권한·암호화 관리 필요 |

**★ 반드시 기억**

- ConfigMap = 일반 설정
- Secret = 민감정보

---

### 3. Base64란?

Base64는 데이터를 문자 형태로 변환하는 **인코딩 방식**이다.

```text
원본 데이터: hello
       ↓ 인코딩
Base64 데이터: aGVsbG8=
       ↓ 디코딩
원본 데이터: hello
```

**Base64는 암호화가 아니다.**

누구나 Base64 데이터를 디코딩하여 원래 값을 확인할 수 있다.

#### Secret 보안 주의사항

- Base64를 암호화로 착각하면 안 된다.
- Secret 데이터는 기본 설정에서 etcd에 암호화되지 않은 상태로 저장될 수 있다.
- 운영 환경에서는 Encryption at Rest 설정을 고려해야 한다.
- RBAC로 Secret 조회·수정 권한을 제한해야 한다.
- 실제 비밀번호를 GitHub에 업로드하면 안 된다.
- 애플리케이션 로그에 비밀번호를 출력하지 않는다.

**★ 반드시 기억: Secret의 Base64 인코딩은 암호화가 아니며, 추가 보안 설정이 필요하다.**

---

### 4. Secret을 Pod에서 사용하는 방법

Secret은 환경변수 또는 파일 형태로 컨테이너에 전달할 수 있다.

#### ① 특정 값만 환경변수로 전달

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: marble-db-secret
        key: DB_PASSWORD
```

| 항목 | 설명 |
|---|---|
| env | 컨테이너 환경변수 정의 |
| name: DB_PASSWORD | 컨테이너 내부 환경변수 이름 |
| valueFrom | 다른 리소스에서 값 가져오기 |
| secretKeyRef | Secret의 특정 키 참조 |
| name: marble-db-secret | Secret 이름 |
| key: DB_PASSWORD | Secret 내부의 키 이름 |

#### ② ConfigMap의 특정 값 가져오기

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: marble-app-config
        key: DB_HOST
```

#### ③ 여러 값을 한 번에 전달하기

```yaml
envFrom:
  - configMapRef:
      name: marble-app-config
  - secretRef:
      name: marble-db-secret
```

**차이점**

| 설정 | 역할 |
|---|---|
| configMapKeyRef | ConfigMap의 특정 값 선택 |
| secretKeyRef | Secret의 특정 값 선택 |
| envFrom | ConfigMap 또는 Secret의 여러 값을 환경변수로 전달 |
| Volume Mount | Secret을 파일 형태로 전달 |

**★ 반드시 기억: secretKeyRef는 Secret에서 특정 값을 가져오는 설정이다.**

---

### 5. Secret 생성 실습

실습 Namespace: `secret-lab`

#### ① Namespace 및 작업 폴더 생성

```bash
kubectl create namespace secret-lab

mkdir -p ~/k8s-study/ch06/secret-lab
cd ~/k8s-study/ch06/secret-lab
```

Namespace가 이미 있다면 생성 명령은 생략한다.

#### ② 일반 설정용 ConfigMap 작성

```bash
nano app-config.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: marble-app-config
data:
  APP_MODE: "practice"
  DB_HOST: "mysql-svc"
```

#### ③ 학습용 Secret 파일 작성

```bash
umask 077
nano db-secret.env
```

```dotenv
DB_USERNAME=marble_lab
DB_PASSWORD=practice-only-v1
```

**주의: 예시 비밀번호는 학습용이며 실제 운영 비밀번호를 파일이나 GitHub에 그대로 저장하면 안 된다.**

파일 권한 설정:

```bash
chmod 600 db-secret.env
```

#### ④ ConfigMap 적용

```bash
kubectl apply -n secret-lab -f app-config.yaml
```

#### ⑤ Secret 생성

```bash
kubectl create secret generic marble-db-secret \
  --from-env-file=db-secret.env \
  -n secret-lab \
  --dry-run=client -o yaml | kubectl apply -n secret-lab -f -
```

**주요 옵션**

| 명령어·옵션 | 의미 |
|---|---|
| create secret generic | 일반 Secret 생성 |
| --from-env-file | 환경변수 파일에서 값 읽기 |
| --dry-run=client | 서버에 바로 생성하지 않고 구성 생성 |
| -o yaml | YAML 형식으로 출력 |
| kubectl apply -f - | 표준 입력으로 받은 설정 적용 |

#### ⑥ Secret 확인

```bash
kubectl get secrets -n secret-lab

kubectl describe secret marble-db-secret -n secret-lab
```

예상 결과:

```text
NAME               TYPE     DATA
marble-db-secret   Opaque   2
```

- `Opaque`: 일반적인 Secret 타입
- `DATA 2`: 저장된 키가 2개라는 의미

**주의: `kubectl get secret -o yaml`은 Base64로 표현된 민감정보를 노출할 수 있으므로 불필요하게 출력하지 않는다.**

---

### 6. Deployment에서 ConfigMap과 Secret 사용

핵심 YAML:

```yaml
env:
  - name: APP_MODE
    valueFrom:
      configMapKeyRef:
        name: marble-app-config
        key: APP_MODE

  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: marble-app-config
        key: DB_HOST

  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: marble-db-secret
        key: DB_USERNAME

  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: marble-db-secret
        key: DB_PASSWORD
```

**동작 흐름**

```text
ConfigMap                    Secret
    │                           │
    │ 일반 설정                 │ 민감정보
    │                           │
    └────────────┬──────────────┘
                 ↓
         Spring Boot Pod
                 ↓
        DB 접속 정보 사용
                 ↓
              MySQL
```

실습에서는 실제 Spring Boot 대신 BusyBox를 사용하여 환경변수 전달만 검증했다.

#### 배포 및 검증 명령어

```bash
# Deployment 생성
kubectl apply -n secret-lab -f marble-secret-demo.yaml

# 배포 상태 확인
kubectl rollout status -n secret-lab \
  deployment/marble-secret-demo

# Pod 상태 확인
kubectl get pods -n secret-lab

# 컨테이너 로그 확인
kubectl logs -n secret-lab \
  deployment/marble-secret-demo --tail=6
```

예상 결과:

```text
APP_MODE=practice
DB_HOST=mysql-svc
DB_CREDENTIALS=LOADED
```

`DB_CREDENTIALS=LOADED`는 Secret 값이 환경변수로 전달되었다는 의미이다.

실제 MySQL 연결 성공을 뜻하지는 않는다.

---

### 7. Secret을 변경하면 어떻게 될까?

**환경변수로 전달한 Secret은 값이 변경되어도 기존 컨테이너에 자동 반영되지 않는다.**

#### ① Secret 값 변경

```bash
nano db-secret.env
```

학습용 비밀번호를 `practice-only-v2`로 변경한다.

#### ② Secret 업데이트

```bash
kubectl create secret generic marble-db-secret \
  --from-env-file=db-secret.env \
  -n secret-lab \
  --dry-run=client -o yaml | kubectl apply -n secret-lab -f -
```

#### ③ Deployment 재시작

```bash
kubectl rollout restart -n secret-lab \
  deployment/marble-secret-demo
```

#### ④ 배포 상태 확인

```bash
kubectl rollout status -n secret-lab \
  deployment/marble-secret-demo
```

새 Pod가 생성되면서 변경된 Secret 값을 환경변수로 전달받는다.

**Secret 변경 방식 비교**

| 사용 방식 | 변경 시 동작 |
|---|---|
| 환경변수 | Pod 재생성 필요 |
| 일반 Volume Mount | 파일 내용이 일정 시간 후 갱신될 수 있음 |
| subPath Mount | 일반적인 자동 갱신 대상 아님 |

Volume으로 전달한 Secret 파일이 변경되더라도 애플리케이션이 파일을 다시 읽는지는 별도로 확인해야 한다.

**★ 반드시 기억: 환경변수로 사용한 Secret을 변경하면 새 값을 적용하기 위해 Pod 재생성이 필요하다.**

---

### 8. Secret 장애 대응

**공통 공식: 상태 → 상세 → 로그 → 원인 → 검증**

#### 주요 명령어

```bash
# ① Pod 상태 확인
kubectl get pods -n secret-lab -o wide

# ② Pod 상세 정보 및 Events 확인
kubectl describe pod <Pod이름> -n secret-lab

# ③ Secret 존재 확인
kubectl get secrets -n secret-lab

# ④ ConfigMap 존재 확인
kubectl get configmaps -n secret-lab

# ⑤ 실행된 컨테이너 로그 확인
kubectl logs -n secret-lab \
  deployment/marble-secret-demo --tail=20

# ⑥ 배포 상태 검증
kubectl rollout status -n secret-lab \
  deployment/marble-secret-demo
```

#### 대표 장애 상황

| 증상 | 원인 및 확인 사항 |
|---|---|
| CreateContainerConfigError | Secret 이름·키·Namespace 오류 |
| 환경변수 값 누락 | secretKeyRef 설정 확인 |
| Secret 변경 후 기존 값 유지 | Pod 재생성 필요 |
| ImagePullBackOff | 컨테이너 이미지 다운로드 문제 |
| Secret 조회 권한 오류 | RBAC 권한 확인 |
| 로그에서 비밀번호 노출 | 로그 출력 방식 수정 |

**★ 반드시 기억: 컨테이너가 시작되지 않았다면 로그보다 `kubectl describe pod`의 Events를 먼저 확인한다.**

---

### 9. 면접 핵심 질문

**Q1. Kubernetes Secret이란 무엇인가요?**

비밀번호, 토큰 등 민감정보를 애플리케이션과 분리하여 관리하는 Kubernetes 리소스입니다.

**Q2. ConfigMap과 Secret의 차이는 무엇인가요?**

ConfigMap은 일반적인 설정값을 관리하고, Secret은 비밀번호와 API 키 같은 민감정보를 관리합니다.

**Q3. Secret의 Base64는 암호화인가요?**

아닙니다. Base64는 인코딩 방식이므로 디코딩하면 원래 값을 확인할 수 있습니다. 따라서 저장 시 암호화 및 RBAC 같은 보안 설정이 필요합니다.

**Q4. Secret을 Pod에서 사용하는 방법은 무엇인가요?**

환경변수로 전달하거나 볼륨을 이용해 파일 형태로 전달할 수 있습니다.

**Q5. Secret을 수정했는데 기존 Pod의 환경변수가 바뀌지 않는 이유는 무엇인가요?**

환경변수는 컨테이너 시작 시 설정되기 때문에 변경된 Secret을 적용하려면 Pod를 재생성해야 합니다.

**Q6. Secret 참조 오류로 Pod가 시작되지 않으면 어떻게 확인하나요?**

Pod 상태와 Events를 확인하고, 참조한 Secret의 이름·키·Namespace가 올바른지 검증합니다.

---

## 10. 최종 핵심 암기

| 개념 | 한 줄 암기 |
|---|---|
| ConfigMap | 일반 설정값 관리 |
| Secret | 민감정보 관리 |
| Base64 | 인코딩이며 암호화가 아님 |
| secretKeyRef | Secret의 특정 키 참조 |
| configMapKeyRef | ConfigMap의 특정 키 참조 |
| envFrom | 여러 설정값을 환경변수로 전달 |
| rollout restart | Deployment의 Pod를 다시 배포 |
| RBAC | Kubernetes 리소스 접근 권한 관리 |
| Encryption at Rest | 저장된 데이터 암호화 |

### ★ 반드시 기억할 5문장

1. **Secret은 비밀번호, API 키, 토큰 등 민감정보를 저장하는 Kubernetes 리소스이다.**
2. **ConfigMap은 일반 설정값을, Secret은 민감정보를 관리한다.**
3. **Base64는 암호화가 아니라 인코딩이다.**
4. **Secret은 환경변수 또는 파일 형태로 Pod에 전달할 수 있다.**
5. **환경변수로 사용한 Secret은 변경해도 기존 컨테이너에 자동 반영되지 않으므로 Pod 재생성이 필요하다.**

### 이번 교시 핵심 한 문장

> Kubernetes Secret은 애플리케이션의 민감정보를 분리하여 관리하는 리소스이며, Base64 인코딩 자체는 보안 기능이 아니므로 접근 권한과 암호화 등의 추가 보안 조치가 필요하다.

---

## 학습 결과

- 학습 범위: **6장 2교시 — Secret · 환경변수 · 설정 관리**
- 객관식 확인문제: **8문항 중 8문항 정답 (100점)**
- 전체 강의 진도: **14/26교시 (53.8%)**
- 실제 실습 수행 및 성공 여부: 별도 검증 필요
- 다음 수업: **6장 3교시 — ServiceAccount · RBAC · 권한 관리**

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/cbe09a8a-45e7-4f22-87bf-feb4a9a87fa6" />

