# Kubernetes 프로젝트 학습

## 6장 3교시 — ServiceAccount · RBAC · 권한 관리

### 1. ServiceAccount란?

**ServiceAccount는 Pod 내부의 애플리케이션 등 프로그램이 Kubernetes API에 인증할 때 사용하는 신원(Identity)이다.**

#### ServiceAccount가 필요한 이유

- Pod 내부 프로그램이 Kubernetes API에 접근할 때 신원을 확인하기 위해
- 애플리케이션별로 서로 다른 신원을 사용하기 위해
- 프로그램에 필요한 Kubernetes API 권한만 부여하기 위해

**예시**

```text
모니터링 프로그램(Pod)
        │
        │ Kubernetes API 요청
        ▼
ServiceAccount (marble-reader)
        │
        │ 신원 인증
        ▼
Kubernetes API Server
        │
        │ RBAC 권한 검사
        ▼
Pod 조회 허용 또는 거부
```

#### 사용자 계정과 ServiceAccount 비교

| 구분 | 사용자 계정(User) | ServiceAccount |
|---|---|---|
| 사용 대상 | 사람 | Pod 내부 프로그램 등 |
| 목적 | 클러스터 관리 | Kubernetes API 인증 |
| 예시 | 관리자·개발자 | 모니터링 프로그램 |
| 권한 관리 | RBAC 등 | RBAC 등 |

**주의 사항**

- ServiceAccount는 Linux 사용자 계정과 다르다.
- Namespace마다 일반적으로 `default` ServiceAccount가 존재한다.
- ServiceAccount를 생성한다고 관리자 권한이 자동으로 부여되는 것은 아니다.
- Kubernetes API를 사용하지 않는 Pod에서는 불필요한 토큰 자동 마운트를 제한할 수 있다.

**★ 반드시 기억: ServiceAccount = Kubernetes API에 접근하는 프로그램의 신원**

---

### 2. RBAC란?

**RBAC(Role-Based Access Control)는 Kubernetes에서 누가 어떤 리소스에 어떤 작업을 할 수 있는지 관리하는 역할 기반 접근 제어 방식이다.**

#### RBAC가 필요한 이유

모니터링 프로그램이 Pod 상태만 확인해야 한다고 가정한다.

```text
Pod 조회       → 허용
Pod 목록 조회  → 허용
Pod 삭제       → 권한 없음
Secret 조회    → 권한 없음
Deployment 수정 → 권한 없음
```

프로그램에 불필요한 권한까지 부여하면 보안 사고의 위험이 커진다.

따라서 **최소 권한 원칙(Least Privilege)**에 따라 필요한 작업만 허용해야 한다.

#### ServiceAccount와 RBAC 차이

| 구분 | 의미 |
|---|---|
| ServiceAccount | 누구인가? → 프로그램의 신원 |
| RBAC | 무엇을 할 수 있는가? → 접근 권한 |

**★ 반드시 기억: ServiceAccount는 신원, RBAC는 권한 관리이다.**

---

### 3. RBAC의 핵심 리소스 4개

| 리소스 | 역할 | 범위 |
|---|---|---|
| Role | 허용할 작업 정의 | Namespace |
| RoleBinding | 권한을 대상에게 연결 | Namespace |
| ClusterRole | 클러스터 범위에서 사용할 수 있는 권한 규칙 정의 | 클러스터 범위 리소스 |
| ClusterRoleBinding | ClusterRole의 권한을 클러스터 범위로 연결 | 클러스터 |

#### Role과 RoleBinding의 차이

```text
Role
 └── Pod get, list 권한 정의
           │
           ▼
      RoleBinding
           │
           ▼
ServiceAccount (marble-reader)
           │
           ▼
     Pod 조회 가능
```

- **Role:** 어떤 리소스에 어떤 작업을 허용할지 정의
- **RoleBinding:** 정의된 권한을 사용자나 ServiceAccount 등에 부여

Role만 생성하면 ServiceAccount에 권한이 자동으로 연결되지 않는다.

#### ClusterRole과 ClusterRoleBinding

- ClusterRole은 Node 같은 클러스터 범위 리소스의 권한도 정의할 수 있다.
- ClusterRoleBinding은 ClusterRole의 권한을 클러스터 범위로 부여한다.
- RoleBinding이 ClusterRole을 참조하면 해당 Namespace에 한정된 권한을 부여할 수도 있다.

**★ 반드시 기억: Role = 권한 정의 / RoleBinding = 권한 연결**

---

### 4. RBAC의 verbs란?

`verbs`는 리소스에 수행할 수 있는 작업의 종류를 의미한다.

| Verb | 의미 |
|---|---|
| get | 특정 리소스 하나 조회 |
| list | 리소스 목록 조회 |
| watch | 리소스 변경 감시 |
| create | 리소스 생성 |
| update | 리소스 수정 |
| patch | 리소스 일부 수정 |
| delete | 리소스 삭제 |

예시:

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

의미:

- `pods`: 권한 대상은 Pod
- `get`: 특정 Pod 조회 허용
- `list`: Pod 목록 조회 허용
- `delete`: 부여하지 않음

단, 다른 RoleBinding 등으로 추가 권한을 받았다면 삭제할 수도 있다.

**★ 반드시 기억: verbs는 Kubernetes 리소스에 허용할 작업을 정의한다.**

---

### 5. ServiceAccount 생성 실습

실습 Namespace: `rbac-lab`

#### ① Namespace 생성

```bash
kubectl create namespace rbac-lab
```

이미 존재한다면 생성 명령은 생략한다.

#### ② ServiceAccount 생성

```bash
kubectl create serviceaccount marble-reader -n rbac-lab
```

#### ③ 생성 확인

```bash
kubectl get serviceaccounts -n rbac-lab

kubectl describe serviceaccount marble-reader -n rbac-lab
```

**핵심 개념**

ServiceAccount를 생성해도 Pod를 조회하거나 삭제할 권한이 자동으로 생기는 것은 아니다.

일반적으로 현대 Kubernetes에서 ServiceAccount의 `SECRETS` 개수가 0으로 표시되어도 정상일 수 있다.

---

### 6. Role 생성 실습

파일: `pod-reader-role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

#### YAML 해설

| 항목 | 의미 |
|---|---|
| kind: Role | Namespace 범위 권한 규칙 |
| metadata.name | Role 이름 |
| apiGroups: [""] | Pod가 속한 핵심 API 그룹 |
| resources: ["pods"] | Pod에 대한 권한 |
| verbs: ["get", "list"] | 조회 권한 허용 |

#### Role 적용

```bash
kubectl apply -n rbac-lab -f pod-reader-role.yaml
```

#### Role 확인

```bash
kubectl get roles -n rbac-lab

kubectl describe role pod-reader -n rbac-lab
```

**★ 반드시 기억: Role은 권한을 정의할 뿐, 특정 대상에게 자동 부여하지 않는다.**

---

### 7. RoleBinding 생성 실습

파일: `marble-read-binding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: marble-read-binding

subjects:
  - kind: ServiceAccount
    name: marble-reader
    namespace: rbac-lab

roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### YAML 해설

| 항목 | 의미 |
|---|---|
| kind: RoleBinding | 권한 연결 리소스 |
| subjects | 권한을 받을 대상 |
| kind: ServiceAccount | 대상의 종류 |
| name: marble-reader | 권한을 받을 ServiceAccount |
| roleRef | 연결할 Role 또는 ClusterRole |
| name: pod-reader | 적용할 Role 이름 |

#### RoleBinding 적용

```bash
kubectl apply -n rbac-lab -f marble-read-binding.yaml
```

#### RoleBinding 확인

```bash
kubectl get rolebindings -n rbac-lab

kubectl describe rolebinding marble-read-binding -n rbac-lab
```

**동작 결과**

```text
ServiceAccount: marble-reader
        │
        ▼
RoleBinding: marble-read-binding
        │
        ▼
Role: pod-reader
        │
        ▼
Pod get, list 권한 부여
```

---

### 8. RBAC 권한 검증

**kubectl auth can-i는 특정 작업이 허용되는지 확인하는 명령어이다.**

`--as`를 사용하려면 명령을 실행하는 계정에 다른 신원으로 가장할 권한이 있어야 한다.

#### ① Pod 목록 조회 권한

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:rbac-lab:marble-reader \
  -n rbac-lab
```

예상 결과:

```text
yes
```

#### ② Pod 삭제 권한

```bash
kubectl auth can-i delete pods \
  --as=system:serviceaccount:rbac-lab:marble-reader \
  -n rbac-lab
```

예상 결과:

```text
no
```

#### ③ Secret 조회 권한

```bash
kubectl auth can-i get secrets \
  --as=system:serviceaccount:rbac-lab:marble-reader \
  -n rbac-lab
```

예상 결과:

```text
no
```

#### ④ 다른 Namespace의 Pod 조회

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:rbac-lab:marble-reader \
  -n default
```

예상 결과:

```text
no
```

#### 예상 권한 결과

| 작업 | 결과 |
|---|---|
| rbac-lab Pod 조회 | 허용 |
| rbac-lab Pod 목록 조회 | 허용 |
| rbac-lab Pod 삭제 | 거부 |
| rbac-lab Secret 조회 | 거부 |
| default Namespace Pod 조회 | 거부 |

위 결과는 다른 RBAC 권한이 추가되지 않은 실습 환경을 가정한다.

**★ 반드시 기억: kubectl auth can-i로 Kubernetes API 작업 권한을 검증할 수 있다.**

---

### 9. Pod에 ServiceAccount 지정하기

Pod YAML에서 다음 설정을 사용한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: marble-rbac-demo
spec:
  serviceAccountName: marble-reader

  containers:
    - name: demo
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - "while true; do sleep 3600; done"
```

#### 핵심 설정

```yaml
serviceAccountName: marble-reader
```

이 Pod는 `marble-reader` ServiceAccount 신원을 사용한다.

#### 적용 및 확인

```bash
kubectl apply -n rbac-lab -f marble-rbac-demo.yaml

kubectl get pods -n rbac-lab

kubectl get pod marble-rbac-demo \
  -n rbac-lab \
  -o jsonpath='{.spec.serviceAccountName}{"\n"}'
```

예상 결과:

```text
marble-reader
```

ServiceAccount를 지정했다는 것과 실제 Kubernetes API 요청이 성공했다는 것은 구분해야 한다.

이번 BusyBox 실습은 ServiceAccount 연결을 확인하는 예시이다.

---

### 10. Pod 로그 조회 권한

Pod 조회 권한과 Pod 로그 조회 권한은 별도로 관리할 수 있다.

Pod 로그 조회에는 일반적으로 다음 권한이 필요하다.

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
```

#### 로그 조회 권한 확인

```bash
kubectl auth can-i get pods \
  --subresource=log \
  --as=system:serviceaccount:rbac-lab:marble-reader \
  -n rbac-lab
```

#### 핵심 개념

| 리소스 | Verb | 역할 |
|---|---|---|
| pods | get | Pod 상세 조회 |
| pods | list | Pod 목록 조회 |
| pods/log | get | Pod 로그 조회 |
| pods | delete | Pod 삭제 |

**★ 반드시 기억: Pod 조회와 Pod 로그 조회는 서로 다른 API 권한으로 구분할 수 있다.**

---

### 11. ServiceAccount 토큰 자동 마운트

Kubernetes API를 호출할 필요가 없는 Pod에서는 불필요한 ServiceAccount 토큰 자동 마운트를 비활성화할 수 있다.

```yaml
spec:
  automountServiceAccountToken: false
```

#### 필요한 이유

- Kubernetes API 인증 정보가 불필요하게 컨테이너에 노출되는 것을 줄이기 위해
- 최소 권한 원칙을 적용하기 위해

**주의:** Pod 내부에서 ServiceAccount 토큰을 사용해 Kubernetes API를 호출해야 한다면 해당 설정을 사용하면 안 될 수 있다.

**★ 반드시 기억: Kubernetes API를 사용하지 않는 애플리케이션에서는 불필요한 ServiceAccount 토큰 마운트를 제한할 수 있다.**

---

### 12. RBAC 장애 대응

**공통 공식: 상태 → 상세 → 로그 → 원인 → 검증**

#### 대표적인 오류

```text
Error from server (Forbidden):
pods is forbidden
```

의미:

인증된 신원에 해당 Kubernetes API 작업을 수행할 권한이 없을 수 있다는 뜻이다.

#### 확인 명령어

```bash
# ① ServiceAccount 확인
kubectl get serviceaccounts -n rbac-lab

# ② Role 확인
kubectl describe role pod-reader -n rbac-lab

# ③ RoleBinding 확인
kubectl describe rolebinding marble-read-binding -n rbac-lab

# ④ Pod에 설정된 ServiceAccount 확인
kubectl get pod marble-rbac-demo \
  -n rbac-lab \
  -o jsonpath='{.spec.serviceAccountName}{"\n"}'

# ⑤ 최종 권한 검증
kubectl auth can-i list pods \
  --as=system:serviceaccount:rbac-lab:marble-reader \
  -n rbac-lab
```

#### 대표 장애 상황

| 증상 | 확인 사항 |
|---|---|
| Pod 조회 Forbidden | Role의 resources와 verbs |
| Role은 있지만 권한 없음 | RoleBinding 연결 여부 |
| 다른 Namespace 접근 실패 | 권한 적용 Namespace |
| Pod 로그 조회 실패 | pods/log의 get 권한 |
| ServiceAccount 연결 오류 | serviceAccountName |
| --as 명령 자체가 거부됨 | 실행 계정의 impersonate 권한 |

**★ 반드시 기억: RBAC 장애는 ServiceAccount → Role → RoleBinding → Namespace → 실제 권한 검증 순서로 확인한다.**

관리자 권한이 필요하다는 이유로 무조건 `cluster-admin`을 부여하는 것은 피해야 한다.

---

### 13. 실습 정리

```bash
# 실습 리소스 확인
kubectl get pods,serviceaccounts,roles,rolebindings \
  -n rbac-lab
```

다른 필요한 리소스가 없는 실습 전용 Namespace라면:

```bash
kubectl delete namespace rbac-lab
```

Namespace 삭제 시 그 안의 Pod, ServiceAccount, Role, RoleBinding도 함께 삭제된다.

---

### 14. 면접 핵심 질문

**Q1. ServiceAccount란 무엇인가요?**

Kubernetes에서 Pod 내부 애플리케이션 등의 프로그램이 API 서버에 인증할 때 사용하는 신원입니다.

**Q2. RBAC란 무엇인가요?**

역할 기반 접근 제어로, Kubernetes 리소스에 대한 사용자나 ServiceAccount의 작업 권한을 관리하는 기능입니다.

**Q3. Role과 RoleBinding의 차이는 무엇인가요?**

Role은 허용할 작업을 정의하고, RoleBinding은 해당 권한을 사용자나 ServiceAccount에 연결합니다.

**Q4. Role과 ClusterRole의 차이는 무엇인가요?**

Role은 Namespace 범위의 권한을 정의하고, ClusterRole은 클러스터 범위의 리소스까지 포함할 수 있는 권한 규칙을 정의합니다.

**Q5. Pod 조회는 허용하고 삭제는 막으려면 어떻게 하나요?**

Role에 get, list 권한만 정의하고 RoleBinding으로 필요한 ServiceAccount에 연결합니다. 다른 경로에서 추가 권한을 받지 않았는지도 확인해야 합니다.

**Q6. RBAC 권한 오류가 발생하면 어떻게 해결하나요?**

ServiceAccount, Role, RoleBinding, Namespace의 연결 상태를 확인한 뒤 `kubectl auth can-i`로 실제 권한을 검증합니다.

**Q7. 최소 권한 원칙이란 무엇인가요?**

프로그램이나 사용자에게 업무 수행에 필요한 최소한의 권한만 부여하는 보안 원칙입니다.

---

## 15. 최종 핵심 암기

| 개념 | 한 줄 암기 |
|---|---|
| ServiceAccount | Kubernetes API에 접근하는 프로그램의 신원 |
| RBAC | 리소스 접근 권한 관리 |
| Role | Namespace 범위 권한 정의 |
| RoleBinding | 권한과 대상 연결 |
| ClusterRole | 클러스터 범위까지 포함할 수 있는 권한 규칙 |
| ClusterRoleBinding | 클러스터 범위 권한 연결 |
| verbs | get, list, create, delete 등 작업 종류 |
| serviceAccountName | Pod가 사용할 ServiceAccount 지정 |
| kubectl auth can-i | 특정 API 작업의 권한 여부 확인 |
| Forbidden | 권한 거부 등으로 요청이 허용되지 않음 |

### ★ 반드시 기억할 5문장

1. **ServiceAccount는 프로그램이 Kubernetes API에 인증할 때 사용하는 신원이다.**
2. **RBAC는 누가 어떤 Kubernetes 리소스에 어떤 작업을 할 수 있는지 관리한다.**
3. **Role은 권한을 정의하고, RoleBinding은 권한을 대상에게 연결한다.**
4. **Role과 RoleBinding으로 특정 Namespace에 한정된 권한을 부여할 수 있다.**
5. **최소 권한 원칙에 따라 프로그램에는 필요한 작업 권한만 부여해야 한다.**

### 이번 교시 핵심 한 문장

> ServiceAccount는 Kubernetes API에 접근하는 프로그램의 신원이고, RBAC는 Role과 RoleBinding 등을 이용해 해당 프로그램이 수행할 수 있는 작업을 제한하는 권한 관리 기능이다.

---

## 학습 결과

- 학습 범위: 6장 3교시 — ServiceAccount · RBAC · 권한 관리
- 객관식 확인문제: **8문항 중 8문항 정답 (100점)**
- 6장 전체 객관식: **24문항 중 23문항 정답 (95.8%)**
- 전체 강의 진도: **15/26교시 (57.7%)**
- 실제 실습 수행 및 성공 여부: 별도 검증 필요
- 다음 수업: **7장 1교시 — PV · PVC · 데이터 영구 저장**

  <img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/8df2a31c-2e8c-4286-a848-7a493fab4f56" />
