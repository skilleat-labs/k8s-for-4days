# Probe 2 · Readiness Probe

!!! info "예상 소요 30분"

---

!!! quote "이대리"
    *"Pod가 Running이라고 바로 트래픽 보내면 안 돼. DB 연결이 아직 안 됐을 수도 있잖아. Readiness Probe는 '진짜 준비됐냐'를 확인하는 거야."*

---

## 이론

### Readiness Probe란?

Liveness Probe가 "살아있냐?"를 물어본다면, Readiness Probe는 **"트래픽 받을 준비됐냐?"** 를 물어봅니다.

Pod가 `Running` 상태여도 다음 상황에서는 트래픽을 보내면 안 됩니다:

- 앱이 DB 연결을 아직 맺는 중
- 캐시 워밍업이 진행 중
- 배포 직후 초기화 작업 중

### Liveness와의 결정적 차이

```text
Liveness 실패  →  컨테이너 재시작
Readiness 실패 →  Service Endpoint에서 제거 (재시작 없음)
```

Pod는 살아있지만 Service가 해당 Pod로 트래픽을 보내지 않습니다.
준비가 되면 **자동으로 Endpoint에 다시 추가**됩니다.

### 동작 구조

```text title="Readiness Probe 동작"
[Pod A]  READY 1/1  ──┐
[Pod B]  READY 1/1  ──┼──▶  Service  ──▶  사용자 요청
[Pod C]  READY 0/1  ✗ ┘     (Pod C는 Endpoint 제외)
```

Readiness가 실패한 Pod C는 Service의 Endpoint 목록에서 빠집니다.
준비가 완료되면 자동으로 다시 추가됩니다.

### 롤링 업데이트와 Readiness의 관계

Readiness Probe가 운영에서 가장 빛나는 순간은 **배포(롤링 업데이트)** 중입니다.

```text title="Readiness 있을 때 롤링 업데이트"
[구 Pod A]  READY 1/1  ──▶ 트래픽 처리 중
[구 Pod B]  READY 1/1  ──▶ 트래픽 처리 중
                             ↓ 배포 시작
[신 Pod C]  READY 0/1  (초기화 중 — 트래픽 안 받음)
[구 Pod A]  READY 1/1  ──▶ 여전히 트래픽 처리
[구 Pod B]  READY 1/1  ──▶ 여전히 트래픽 처리
                             ↓ 신 Pod C Ready!
[신 Pod C]  READY 1/1  ──▶ 트래픽 처리 시작
[구 Pod A]  삭제 시작...
```

Readiness가 없다면 초기화 중인 새 Pod에 트래픽이 바로 가서 오류가 발생합니다.

---

## 실습

### Step 1. Deployment + Service 작성 및 배포

파일명 `deploy-readiness.yaml`에 **Deployment와 Service를 하나의 파일에** 작성합니다.
Readiness Probe는 아직 넣지 않습니다.

**Deployment 조건**

| 항목 | 값 |
|------|-----|
| `metadata.name` | `readiness-app` |
| `spec.replicas` | `3` |
| `selector.matchLabels` / `template.labels` | `app: readiness-app` |
| 컨테이너 이름 | `app` |
| 이미지 | `nginx:1.25` |
| `containerPort` | `80` |

**Service 조건**

| 항목 | 값 |
|------|-----|
| `metadata.name` | `readiness-svc` |
| `port` / `targetPort` | `80` |

!!! tip "Deployment와 Service를 한 파일에 쓸 때"
    두 리소스 사이에 `---` (문서 구분자)를 넣으면 됩니다.

??? success "작성 완료 후 정답 확인"
    ```yaml title="deploy-readiness.yaml"
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: readiness-app
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: readiness-app
      template:
        metadata:
          labels:
            app: readiness-app
        spec:
          containers:
            - name: app
              image: nginx:1.25
              ports:
                - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: readiness-svc
    spec:
      selector:
        app: readiness-app
      ports:
        - port: 80
          targetPort: 80
    ```

```bash title="터미널"
kubectl apply -f deploy-readiness.yaml
kubectl get pods -l app=readiness-app
kubectl get endpoints readiness-svc
```

!!! success "✅ 확인 포인트"
    3개 Pod가 `READY 1/1`, Endpoints에 IP 3개가 표시되면 OK.

---

### Step 2. Readiness Probe 추가

Probe 없이 배포하면 Pod가 `Running`이 되는 즉시 Endpoint에 추가됩니다.
초기화 중인 앱에 트래픽이 바로 갈 수 있습니다.

`deploy-readiness.yaml`의 Deployment에 아래 조건으로 `readinessProbe`를 추가하세요.

| 항목 | 값 |
|------|-----|
| Probe 방식 | `httpGet` — 경로 `/`, 포트 `80` |
| `initialDelaySeconds` | `5` |
| `periodSeconds` | `5` |
| `failureThreshold` | `3` |

??? success "작성 완료 후 정답 확인"
    ```yaml title="deploy-readiness.yaml (readinessProbe 추가)"
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: readiness-app
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: readiness-app
      template:
        metadata:
          labels:
            app: readiness-app
        spec:
          containers:
            - name: app
              image: nginx:1.25
              ports:
                - containerPort: 80
              readinessProbe:
                httpGet:
                  path: /
                  port: 80
                initialDelaySeconds: 5
                periodSeconds: 5
                failureThreshold: 3
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: readiness-svc
    spec:
      selector:
        app: readiness-app
      ports:
        - port: 80
          targetPort: 80
    ```

```bash title="터미널"
kubectl apply -f deploy-readiness.yaml
kubectl get pods -l app=readiness-app -w
```

```text title="출력 예시"
NAME                            READY   STATUS    RESTARTS   AGE
readiness-app-xxx               0/1     Running   0          3s    ← 초기화 중
readiness-app-xxx               1/1     Running   0          8s    ← Probe 통과 후 트래픽 받음
```

!!! success "✅ 확인 포인트"
    Pod가 `Running`이 된 직후 바로 `READY 1/1`이 아니라, `initialDelaySeconds: 5` 이후에 `READY 1/1`로 바뀌는 것을 확인하세요.
    `Ctrl+C`로 종료.

---

### Step 3. Readiness 실패 → Endpoint 제외 확인

Pod 하나의 nginx를 강제로 중단해서 Readiness를 실패시킵니다.

!!! warning "관측 윈도우 약 5초"
    `nginx -s stop`을 실행하면 nginx 프로세스가 종료되고 **컨테이너가 즉시 재시작**됩니다.
    재시작 후 `initialDelaySeconds: 5` 동안은 `READY 0/1` 상태입니다.
    이 5초 안에 아래 두 명령을 빠르게 실행하세요.

```bash title="터미널"
# Pod 이름 확인
kubectl get pods -l app=readiness-app

# 첫 번째 Pod의 nginx 중단 (실행 직후 빠르게 아래 명령 실행)
kubectl exec <pod이름> -- nginx -s stop
kubectl get pods -l app=readiness-app
kubectl get endpoints readiness-svc
```

!!! success "✅ 확인 포인트"
    해당 Pod가 `Running`이지만 `READY 0/1` 상태이고, `kubectl get endpoints`에서 해당 Pod의 IP가 빠져 있으면 OK.
    5초 후에는 자동으로 `READY 1/1`로 복귀합니다 (`RESTARTS: 1` 표시).

!!! note "Liveness와 비교 / 왜 컨테이너가 재시작되나?"
    nginx를 중단하면 **컨테이너의 메인 프로세스가 종료**되어 K8s가 컨테이너를 재시작합니다.
    이 실습에서 READY 0/1이 보이는 이유는 Readiness Probe 실패가 아니라, **새 컨테이너의 초기화 대기(initialDelaySeconds: 5)** 때문입니다.

    순수하게 "재시작 없이 Readiness만 실패시키는" 실습은 바로 아래 Step 3에서 합니다.

---

### Step 4. Readiness 회복 → Endpoint 자동 복귀

실패와 회복을 직접 제어하기 위해 **파일 기반 exec Probe** Pod를 사용합니다.
파일을 지우면 실패, 다시 만들면 회복 — 이렇게 Readiness 상태를 자유롭게 조작할 수 있습니다.

```yaml title="pod-readiness-exec.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: readiness-exec
  labels:
    app: readiness-exec
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "touch /tmp/ready && sleep 3600"
      readinessProbe:
        exec:
          command:
            - cat
            - /tmp/ready
        initialDelaySeconds: 3
        periodSeconds: 3
        failureThreshold: 2
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-exec-svc
spec:
  selector:
    app: readiness-exec
  ports:
    - port: 80
      targetPort: 80
```

```bash title="터미널"
kubectl apply -f pod-readiness-exec.yaml
kubectl get pod readiness-exec
kubectl get endpoints readiness-exec-svc
```

!!! success "✅ 확인 포인트"
    `READY 1/1`, Endpoint에 IP 1개 확인.

**Readiness 실패시키기 — 파일 삭제**

```bash title="터미널"
kubectl exec readiness-exec -- rm /tmp/ready
kubectl get pod readiness-exec -w
```

```text title="출력 예시"
NAME              READY   STATUS    RESTARTS   AGE
readiness-exec    1/1     Running   0          30s
readiness-exec    0/1     Running   0          36s   ← Readiness 실패
```

```bash title="터미널"
kubectl get endpoints readiness-exec-svc
```

```text title="출력 예시 — Endpoint에서 제거됨"
NAME                  ENDPOINTS   AGE
readiness-exec-svc    <none>      40s
```

**Readiness 회복시키기 — 파일 복구**

```bash title="터미널"
kubectl exec readiness-exec -- touch /tmp/ready
kubectl get pod readiness-exec -w
```

```text title="출력 예시"
NAME              READY   STATUS    RESTARTS   AGE
readiness-exec    0/1     Running   0          50s
readiness-exec    1/1     Running   0          53s   ← 자동 회복!
```

```bash title="터미널"
kubectl get endpoints readiness-exec-svc
```

!!! success "✅ 확인 포인트"
    Endpoint에 IP가 다시 나타나면 OK.
    **수동 개입 없이 자동으로 복귀**되는 것이 핵심입니다.

---

### Step 5. 롤링 업데이트 보호 체험

Readiness Probe가 있을 때 배포 중 트래픽이 끊기지 않는지 직접 확인합니다.

**이미지 업데이트 실행 (터미널 2개 준비)**

```bash title="터미널 1 — Pod 상태 감시"
kubectl get pods -l app=readiness-app -w
```

```bash title="터미널 2 — 이미지 버전 업데이트"
kubectl set image deployment/readiness-app app=nginx:1.24
```

```text title="출력 예시 — 터미널 1"
NAME                            READY   STATUS              RESTARTS
readiness-app-old-xxx           1/1     Running             0
readiness-app-old-yyy           1/1     Running             0
readiness-app-old-zzz           1/1     Running             0
readiness-app-new-aaa           0/1     ContainerCreating   0   ← 새 Pod 생성
readiness-app-new-aaa           0/1     Running             0   ← 초기화 중, 아직 트래픽 안 받음
readiness-app-new-aaa           1/1     Running             0   ← Ready! 이제 트래픽 받음
readiness-app-old-xxx           1/1     Terminating         0   ← 그 다음 구 Pod 종료
```

!!! success "✅ 확인 포인트"
    새 Pod가 `READY 1/1`이 된 **이후**에 구 Pod가 `Terminating`으로 바뀝니다.
    Readiness Probe 덕분에 배포 중에도 항상 최소 2개 Pod가 트래픽을 처리합니다.

롤아웃 상태도 확인합니다:

```bash title="터미널"
kubectl rollout status deployment/readiness-app
```

```text title="출력 예시"
Waiting for deployment "readiness-app" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "readiness-app" rollout to finish: 2 out of 3 new replicas have been updated...
deployment "readiness-app" successfully rolled out
```

!!! tip "Readiness가 없으면 어떻게 될까?"
    새 Pod가 `Running`이 되는 즉시 Service Endpoint에 추가됩니다.
    초기화 중인 Pod에 트래픽이 가서 500 에러가 발생할 수 있습니다.
    **Readiness Probe는 무중단 배포의 핵심 조건입니다.**

---

### 정리

```bash title="터미널"
kubectl delete -f deploy-readiness.yaml
kubectl delete -f pod-readiness-exec.yaml
```

### 트러블슈팅

| 증상 | 확인 |
|------|------|
| Pod는 Running인데 READY=0/1 | Readiness 실패 — `kubectl describe pod` → Events 확인 |
| Endpoint에 Pod IP가 없음 | `kubectl get endpoints <svc이름>`으로 Readiness 상태 확인 |
| 롤링 업데이트가 멈춤 | 새 Pod의 Readiness 실패 → `kubectl rollout status`로 상황 확인 후 `kubectl rollout undo` |

---

# Probe 3 · Startup Probe

!!! quote "김팀장"
    *"Spring Boot 뜨는 데 40초 걸리는 앱에 Liveness를 초반에 돌리면 어떻게 되겠어? 멀쩡한 앱을 계속 죽이는 거야. Startup Probe는 그걸 막는 안전장치야."*

---

## 이론

### Startup Probe가 필요한 이유

Liveness Probe는 주기적으로 체크합니다. 그런데 초기화가 긴 앱은 **뜨는 동안 Liveness가 실패해서** 계속 재시작될 수 있습니다.

```text title="Startup Probe 없을 때 — 초기화 40초 앱"
t=0s    컨테이너 시작, 앱 초기화 중...
t=5s    Liveness Probe 실행 → 실패 (아직 초기화 중)
t=10s   Liveness Probe 실행 → 실패
t=15s   Liveness Probe 실행 → 실패  (failureThreshold: 3 초과)
        → 컨테이너 재시작! (앱이 뜨기도 전에)
t=0s    다시 시작... 무한 반복 ❌
```

### Startup Probe의 역할

**Startup Probe가 성공하기 전까지 Liveness와 Readiness를 잠가둡니다.**

```text title="Startup Probe 있을 때"
t=0s    컨테이너 시작
        Liveness/Readiness ← 비활성화 (Startup이 성공할 때까지)
        Startup Probe만 실행 중...

t=40s   앱 초기화 완료 → Startup Probe 성공
        Liveness/Readiness ← 활성화

t=45s   Liveness Probe 시작
        Readiness Probe 시작 → READY 1/1
```

### 최대 대기 시간 설정

Startup Probe는 `failureThreshold × periodSeconds`로 최대 대기 시간을 설정합니다:

```yaml
startupProbe:
  failureThreshold: 30   # 30회
  periodSeconds: 10      # × 10초 = 최대 300초(5분) 대기
```

이 시간 안에 Startup Probe가 성공하지 못하면 컨테이너를 재시작합니다.

---

## 실습

### Step 1. Startup Probe 동작 확인

`sleep 20`으로 초기화가 20초 걸리는 앱을 시뮬레이션합니다.

```yaml title="pod-startup.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: startup-app
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "sleep 20 && touch /tmp/started && sleep 3600"
      startupProbe:
        exec:
          command:
            - cat
            - /tmp/started
        failureThreshold: 30   # 30 × 10s = 최대 300초 대기
        periodSeconds: 10
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/started
        periodSeconds: 10
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-startup.yaml
kubectl get pod startup-app -w
```

!!! success "✅ 확인 포인트"
    처음 20초간 `READY 0/1`이지만 `RESTARTS`는 0으로 유지됩니다.
    20초 후 `READY 1/1`로 전환되면 OK.

---

### Step 2. Startup Probe 없었다면? (비교 실습)

`initialDelaySeconds` 없이 Liveness만 설정하면 어떻게 되는지 확인합니다.

```yaml title="pod-no-startup.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: no-startup-app
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "sleep 20 && touch /tmp/started && sleep 3600"
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/started
        initialDelaySeconds: 0   # 즉시 시작
        periodSeconds: 5
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-no-startup.yaml
kubectl get pod no-startup-app -w
```

!!! success "✅ 확인 포인트"
    약 10초 후(`t=0` 첫 실패 → `t=5` 두 번째 실패 → `t=10` 세 번째 실패 → kill) `RESTARTS`가 올라가기 시작합니다.
    Startup Probe의 필요성이 보이면 OK.

---

### Step 3. Startup 시간 초과 → 재시작

`failureThreshold × periodSeconds`를 실제 초기화 시간보다 **짧게** 설정하면 어떻게 될까요?
이 설정 실수가 운영에서 의외로 자주 발생합니다.

앱은 20초가 필요한데, Startup Probe의 최대 허용 시간을 6초(3 × 2)로 설정합니다:

```yaml title="pod-startup-timeout.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: startup-timeout
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "sleep 20 && touch /tmp/started && sleep 3600"
      startupProbe:
        exec:
          command:
            - cat
            - /tmp/started
        failureThreshold: 2    # 2회
        periodSeconds: 3       # × 3초 = 최대 6초만 허용 (앱 초기화 20초 필요)
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/started
        periodSeconds: 10
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-startup-timeout.yaml
kubectl get pod startup-timeout -w
```

```text title="출력 예시"
NAME               READY   STATUS    RESTARTS   AGE
startup-timeout    0/1     Running   0          5s
startup-timeout    0/1     Running   1          15s   ← 6초 안에 초기화 못 해서 재시작
startup-timeout    0/1     Running   2          30s   ← 또 재시작 → CrashLoopBackOff로 진행
```

```bash title="터미널"
kubectl describe pod startup-timeout
```

!!! success "✅ 확인 포인트"
    `RESTARTS`가 계속 올라가면 OK. Startup Probe 시간 설정 실수가 어떤 결과를 낳는지 확인.

!!! danger "실무 실수 패턴"
    **초기화 시간을 과소 추정**하는 것이 가장 흔한 실수입니다.
    운영 중 배포할 때 갑자기 CrashLoopBackOff가 나타난다면 Startup Probe의
    `failureThreshold × periodSeconds` 값을 먼저 점검하세요.

    ```
    잘못된 설정:  failureThreshold: 5 × periodSeconds: 5  = 25초  (앱 초기화 30초 필요)
    올바른 설정:  failureThreshold: 10 × periodSeconds: 5 = 50초  (여유분 포함)
    ```

---

### Step 4. 3 Probe 조합 — 활성화 순서 직접 확인

Startup → Readiness → Liveness가 **순서대로** 작동하는 것을 한 Pod에서 확인합니다.

```yaml title="pod-all-probes.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: all-probes
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "sleep 15 && touch /tmp/started && sleep 3600"
      startupProbe:
        exec:
          command: [cat, /tmp/started]
        failureThreshold: 10
        periodSeconds: 3       # 최대 30초 대기
      readinessProbe:
        exec:
          command: [cat, /tmp/started]
        periodSeconds: 3
        failureThreshold: 2
      livenessProbe:
        exec:
          command: [cat, /tmp/started]
        periodSeconds: 10
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-all-probes.yaml
kubectl get pod all-probes -w
```

```text title="출력 예시 — 시간 흐름"
NAME         READY   STATUS    RESTARTS   AGE
all-probes   0/1     Running   0          3s    ← Startup 실행 중, Liveness/Readiness 잠김
all-probes   0/1     Running   0          6s    ← Startup 계속 실패 중 (RESTARTS는 0 유지!)
all-probes   0/1     Running   0          12s   ← 아직 초기화 중
all-probes   1/1     Running   0          18s   ← Startup 성공 → Readiness 즉시 통과 → READY!
```

!!! success "✅ 확인 포인트"
    15초간 `RESTARTS`가 0으로 유지되면 Startup Probe가 Liveness를 잘 막고 있는 것.
    15초 후 `READY 1/1`로 한번에 전환되면 OK.

```bash title="터미널"
kubectl describe pod all-probes
```

### 3 Probe 활성화 타임라인 정리

```text title="3 Probe 타임라인"
t=0s   컨테이너 시작
       ┌─────────────────────────────────────────┐
       │  Startup Probe 실행 중                   │  Liveness ← 잠김
       │  (3초마다 /tmp/started 체크)             │  Readiness ← 잠김
       └─────────────────────────────────────────┘
t=15s  /tmp/started 생성 → Startup 성공!
       Startup Probe 종료
       ┌──────────────────┐  ┌──────────────────┐
       │  Readiness 활성화 │  │  Liveness 활성화  │
       │  READY 1/1       │  │  10초마다 체크    │
       └──────────────────┘  └──────────────────┘
```

---

### 정리

```bash title="터미널"
kubectl delete pod startup-app no-startup-app startup-timeout all-probes
```

### 3가지 Probe 한눈에 비교

| Probe | 실패 시 동작 | 언제 사용 |
|-------|------------|---------|
| **Liveness** | 컨테이너 재시작 | 교착 상태·무한 루프 감지 |
| **Readiness** | Service Endpoint에서 제거 | DB 연결 전, 초기화 중 트래픽 차단 |
| **Startup** | Liveness/Readiness 보호 | 초기화가 긴 앱 (JVM, Spring Boot 등) |

### 실무 설정 가이드

| Probe | 권장 설정 |
|-------|---------|
| Startup | `failureThreshold × periodSeconds` ≥ 최대 초기화 시간 |
| Readiness | `periodSeconds` 5~10s |
| Liveness | `initialDelaySeconds` 충분히, `failureThreshold` 3~5 |

### 트러블슈팅

| 증상 | 확인 |
|------|------|
| 초기화 중 계속 재시작 | Startup Probe 추가 또는 `initialDelaySeconds` 늘리기 |
| Startup Probe도 실패 | `failureThreshold × periodSeconds` 값이 초기화 시간보다 짧은지 확인 |
