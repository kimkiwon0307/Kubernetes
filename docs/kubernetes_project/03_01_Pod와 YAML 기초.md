# 3장 1교시 — Pod와 YAML 기초

## 1. Pod란?

**Pod는 Kubernetes에서 생성하고 배포할 수 있는 가장 작은 컴퓨팅 단위**이며, 하나 이상의 컨테이너를 포함한다.

- Docker: 컨테이너를 직접 실행하고 관리할 수 있다.
- Kubernetes: Pod를 배포 단위로 사용하여 컨테이너를 관리한다.
- 같은 Pod의 컨테이너는 IP와 네트워크 공간을 공유한다.
- Pod 삭제 후 자동 재생성하려면 일반적으로 Deployment 같은 상위 관리 리소스가 필요하다.

```text
Worker Node
    │
    └── Pod
         └── Nginx Container
```

**핵심:** Pod는 컨테이너를 포함하는 관리 단위이며, 실제 컨테이너 실행은 containerd가 담당한다.

## 2. Pod 생성 원리

```text
kubectl
   ↓
API Server (요청 처리)
   ↓
etcd (상태 저장)
   ↓
Scheduler (실행할 Node 선택)
   ↓
kubelet (Pod 실행 상태 관리)
   ↓
containerd (컨테이너 실행)
```

※ 실제 구성요소들은 API Server를 통해 상태를 조회하고 변경한다.

**역할 구분**
- Scheduler: 어디에서 실행할지 결정
- kubelet: 원하는 Pod 실행 상태를 관리
- containerd: 실제 컨테이너 실행

## 3. YAML의 기본 구조

Kubernetes YAML은 원하는 리소스 구성을 선언하는 파일이다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx-lab
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 80
```

| 항목 | 의미 |
|---|---|
| apiVersion | 사용할 API 버전 |
| kind | 생성할 리소스 종류 |
| metadata | 리소스 이름과 라벨 |
| spec | 원하는 리소스 구성 |
| containers | 컨테이너 목록 |
| image | 실행할 이미지 |
| containerPort | 컨테이너가 사용하는 포트 정보 |

**주의:** `containerPort: 80`을 선언해도 외부 접근이 자동으로 허용되는 것은 아니다.

**YAML 작성 규칙**
- 들여쓰기는 공백 사용
- `:`는 속성과 값 구분
- `-`는 목록 항목
- `#`는 주석

## 4. Pod 생성 및 검증 명령어

### YAML 작성

```bash
mkdir -p ~/k8s-study/ch03
cd ~/k8s-study/ch03
nano nginx-pod.yaml
```

### Pod 생성

```bash
kubectl apply -f nginx-pod.yaml
```

- `apply`: 리소스 구성 적용
- `-f`: YAML 파일 지정

### Pod 상태 확인

```bash
kubectl get pods
kubectl get pods -o wide
kubectl describe pod nginx-pod
```

- `get pods`: Pod 상태 확인
- `-o wide`: Pod IP 및 실행 Node 확인
- `describe`: 상세 정보와 Events 확인

### Pod 내부 명령어 실행

```bash
kubectl exec nginx-pod -- \
  wget -qO- http://127.0.0.1:80/
```

`exec`는 실행 중인 컨테이너 내부에서 명령어를 실행한다.

### 포트 포워딩

```bash
kubectl port-forward pod/nginx-pod 8080:80
```

다른 터미널에서 확인:

```bash
curl -I http://127.0.0.1:8080
```

정상 응답 예시: `HTTP/1.1 200 OK`

### Pod 로그 확인

```bash
kubectl logs nginx-pod
kubectl logs nginx-pod --tail=20
kubectl logs -f nginx-pod
```

- `--tail=20`: 최근 20줄
- `-f`: 실시간 로그 확인

### Pod 삭제

```bash
kubectl delete pod nginx-pod
```

**중요:** 단독 생성한 Pod를 삭제해도 Kubernetes가 자동으로 같은 Pod를 재생성하지 않는다.

## 5. Pod 장애 대응

| 상태·증상 | 의미 또는 가능한 원인 | 점검 |
|---|---|---|
| Pending | 스케줄링 대기, 자원 부족 등 | describe |
| ImagePullBackOff | 이미지 다운로드 실패 후 재시도 지연 | describe, 이미지 설정 |
| CrashLoopBackOff | 컨테이너 반복 종료 | logs --previous |
| Running인데 접속 실패 | 애플리케이션·포트·네트워크 문제 | logs, exec, curl |

### ImagePullBackOff 장애 대응

```bash
# 1. Pod 상태 확인
kubectl get pods -o wide

# 2. 상세 이벤트 확인
kubectl describe pod nginx-pod

# 3. 이미지 이름 확인
kubectl get pod nginx-pod \
  -o jsonpath='{.spec.containers[*].image}'
```

**점검 항목**
1. 이미지 이름이 정확한가?
2. 이미지 태그가 존재하는가?
3. Worker Node에서 레지스트리에 접근 가능한가?
4. DNS에 문제가 없는가?
5. Private Registry 인증정보가 올바른가?

이미지를 다운로드하지 못해 컨테이너가 실행되지 않았다면 `kubectl logs`보다 `kubectl describe pod`의 Events가 더 중요하다.

## 6. 장애 대응 명령어 구분

| 명령어 | 목적 |
|---|---|
| kubectl get pods | Pod 상태 조회 |
| kubectl describe pod | Pod 상세 정보·이벤트 분석 |
| kubectl logs | 컨테이너 로그 분석 |
| kubectl exec | 컨테이너 내부 점검 |
| kubectl get nodes | Node 상태 확인 |

**암기할 핵심**

- **get:** 현재 상태 확인
- **describe:** 상세 상태와 장애 단서 확인
- **logs:** 애플리케이션 로그 분석
- **exec:** 컨테이너 내부 점검

## 7. 확인문제 핵심 답변

**Q1. Pod란 무엇인가?**

Kubernetes에서 하나 이상의 컨테이너를 포함하는 최소 배포 단위이다.

**Q2. Pod와 컨테이너의 차이는?**

컨테이너는 애플리케이션 실행 환경이며, Pod는 하나 이상의 컨테이너를 포함하는 Kubernetes의 배포·관리 단위이다.

**Q3. ImagePullBackOff가 발생하면?**

`kubectl describe pod`로 이벤트를 확인하고 이미지 이름, 태그, 네트워크, 레지스트리 인증정보를 점검한다.

**Q4. Scheduler, kubelet, containerd의 차이는?**

Scheduler는 실행할 Node를 선택하고, kubelet은 Pod의 실행을 관리하며, containerd는 실제 컨테이너를 실행한다.

**Q5. 단독 Pod를 삭제하면 자동 복구되는가?**

아니다. 단독 Pod는 삭제 후 자동으로 재생성되지 않는다. 일반적으로 Deployment 같은 컨트롤러를 이용해 원하는 Pod 수를 유지한다.

---

## 8. 최종 핵심 요약

1. **Pod:** Kubernetes의 최소 배포 단위
2. **YAML:** 원하는 리소스 구성 선언
3. **kubectl apply:** YAML 적용
4. **kubectl get:** 상태 조회
5. **kubectl describe:** 이벤트 및 상세 상태 확인
6. **kubectl logs:** 컨테이너 로그 분석
7. **kubectl exec:** 컨테이너 내부 명령 실행
8. **ImagePullBackOff:** 이미지 다운로드 실패
9. **Deployment:** Pod 복제 및 재생성 관리에 사용

**이번 교시의 핵심 한 문장**

> Kubernetes는 Pod를 단위로 컨테이너를 관리하며, YAML로 원하는 상태를 선언하고 kubectl로 배포·조회·검증한다.

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/c4fa2ecd-222b-4d41-9a09-6c6f8440c2e8" />
