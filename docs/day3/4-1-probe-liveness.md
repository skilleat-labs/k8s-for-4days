# Probe 1 · Liveness Probe

!!! info "예상 소요 30분"

---

!!! quote "김팀장"
    *"프로세스가 살아있다고 앱이 정상인 건 아니야. 무한 루프에 빠진 앱도 프로세스는 멀쩡하거든. Liveness Probe는 그걸 잡아내는 거야."*

---

## 이론

### Liveness Probe란?

K8s는 컨테이너가 실행 중인지는 알 수 있습니다. 하지만 **실행 중인 앱이 정상적으로 동작하는지**는 모릅니다.

예를 들어:

- 교착 상태(Deadlock)에 빠진 Java 앱 — 프로세스는 살아있지만 요청을 처리 못 함
- 무한 루프에 빠진 Worker — CPU만 먹고 아무 일도 안 함

이런 상황을 감지해서 컨테이너를 **자동 재시작**하는 것이 Liveness Probe입니다.

```
Liveness Probe 실패 → 컨테이너 재시작
```

### 핵심 파라미터

| 파라미터 | 의미 |
|---------|------|
| `initialDelaySeconds` | 컨테이너 시작 후 **첫 Probe까지 대기** (앱 초기화 시간 확보) |
| `periodSeconds` | 이후 **반복 주기** |
| `failureThreshold` | **연속 몇 회 실패**해야 재시작할지 |

### 타이밍 흐름

```text title="Probe 타이밍 예시 (initialDelaySeconds:5, periodSeconds:5)"
t=0s    컨테이너 시작
        ↕ initialDelaySeconds: 5 — 앱 초기화 대기
t=5s    첫 번째 Probe 실행
        ↕ periodSeconds: 5
t=10s   두 번째 Probe 실행
        ↕ periodSeconds: 5
t=15s   세 번째 Probe 실행
        ...5초마다 반복...
```

!!! note "failureThreshold: 3 의 의미"
    한 번 실패해도 바로 재시작하지 않습니다.
    **연속 3회 실패** = 최대 `periodSeconds × failureThreshold = 5 × 3 = 15초` 감지 후 재시작.

!!! tip "initialDelaySeconds가 왜 필요한가?"
    컨테이너가 뜨는 순간 앱은 아직 초기화 중입니다.
    이 시점에 Probe를 바로 실행하면 정상 앱도 실패로 판정되어 불필요하게 재시작됩니다.
    Spring Boot, JVM처럼 기동이 느린 앱은 `initialDelaySeconds: 30` 이상으로 설정하세요.

---

## 실습

### Step 1. 정상 동작 확인

Liveness Probe가 계속 성공하는 Pod를 만듭니다.

```yaml title="pod-liveness-ok.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: liveness-ok
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "touch /tmp/healthy && sleep 3600"
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-liveness-ok.yaml
kubectl get pod liveness-ok -w
```

!!! success "✅ 확인 포인트"
    `RESTARTS`가 0으로 유지되면 OK. `Ctrl+C`로 종료.

---

### Step 2. Liveness 실패 → 재시작 확인

30초 후 `/tmp/healthy` 파일을 삭제해서 Probe를 의도적으로 실패시킵니다.

```yaml title="pod-liveness-fail.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: liveness-fail
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "touch /tmp/healthy && sleep 30 && rm /tmp/healthy && sleep 3600"
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-liveness-fail.yaml
kubectl get pod liveness-fail -w
```

!!! success "✅ 확인 포인트"
    30~60초 후 `RESTARTS` 숫자가 1로 증가하면 OK.

이벤트 로그로 재시작 원인을 확인합니다:

```bash title="터미널"
kubectl describe pod liveness-fail
```

```text title="출력 예시 — Events 섹션"
Events:
  Warning  Unhealthy  Liveness probe failed: ...
  Normal   Killing    Stopping container app
  Normal   Started    Started container app
```

---

### Step 3. httpGet 방식 — 실무에서 가장 많이 쓰는 형태

Step 1~2는 파일 존재 여부로 체크하는 `exec` 방식이었습니다.
실제 운영에서는 **HTTP 헬스체크 엔드포인트**(`/health`, `/healthz`)를 호출하는 `httpGet` 방식을 주로 사용합니다.

!!! note "왜 httpGet이 더 실무적인가?"
    `exec`는 파일 시스템 상태만 확인합니다.
    `httpGet`은 앱이 실제로 HTTP 요청을 처리할 수 있는 상태인지 확인합니다.
    FastAPI, Spring Boot, Node.js 등 웹 앱은 대부분 `/health` 엔드포인트를 제공합니다.

nginx의 `/` 경로를 헬스체크 엔드포인트로 활용하는 예시입니다:

```yaml title="pod-liveness-http.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: /          # 헬스체크 경로 (실무: /health 또는 /healthz)
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-liveness-http.yaml
kubectl get pod liveness-http -w
```

!!! success "✅ 확인 포인트"
    `RESTARTS` 0 유지 확인. `Ctrl+C`로 종료.

이번엔 nginx를 강제로 중단해서 HTTP Probe 실패를 유도합니다:

```bash title="터미널"
kubectl exec liveness-http -- nginx -s stop
kubectl get pod liveness-http -w
```

```text title="출력 예시"
NAME             READY   STATUS    RESTARTS   AGE
liveness-http    1/1     Running   0          30s
liveness-http    1/1     Running   1          60s   ← HTTP Probe 실패 → 재시작
```

!!! success "✅ 확인 포인트"
    nginx 중단 후 `RESTARTS`가 1로 올라가면 OK.

#### exec vs httpGet vs tcpSocket 비교

| 방식 | 체크 내용 | 언제 사용 |
|------|---------|---------|
| `exec` | 명령어 종료 코드(0=성공) | 파일 존재, 스크립트 실행 결과 |
| `httpGet` | HTTP 응답 코드(200~399=성공) | 웹 앱, REST API (가장 일반적) |
| `tcpSocket` | 포트 연결 가능 여부 | HTTP 없는 TCP 서버 (DB, gRPC 등) |

---

### Step 4. CrashLoopBackOff — 운영에서 꼭 마주치는 상황

Liveness Probe가 계속 실패하면 Pod는 계속 재시작됩니다.
K8s는 재시작이 반복될수록 **대기 시간을 지수적으로 늘립니다** (10s → 20s → 40s → 최대 5분).
이 상태가 `CrashLoopBackOff`입니다.

```text title="CrashLoopBackOff 진행 과정"
재시작 1회  →  10초 대기
재시작 2회  →  20초 대기
재시작 3회  →  40초 대기
재시작 4회  →  80초 대기
재시작 5회~ →  최대 300초(5분) 대기  ← CrashLoopBackOff 상태 표시
```

!!! warning "CrashLoopBackOff ≠ 앱 종료"
    Pod가 CrashLoopBackOff 상태여도 K8s는 재시작을 계속 시도합니다.
    앱이 영구적으로 죽은 게 아니라 "재시작 대기 중"인 상태입니다.

처음부터 Probe를 실패시키는 Pod를 만들어봅니다:

```yaml title="pod-crashloop.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: crashloop-app
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "sleep 3600"
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/nonexistent    # 존재하지 않는 파일 → 항상 실패
        initialDelaySeconds: 3
        periodSeconds: 3
        failureThreshold: 2
```

```bash title="터미널"
kubectl apply -f pod-crashloop.yaml
kubectl get pod crashloop-app -w
```

```text title="출력 예시"
NAME            READY   STATUS             RESTARTS   AGE
crashloop-app   1/1     Running            0          5s
crashloop-app   0/1     CrashLoopBackOff   1          20s
crashloop-app   1/1     Running            2          45s
crashloop-app   0/1     CrashLoopBackOff   3          90s
```

CrashLoopBackOff 상태가 보이면 `Ctrl+C`로 watch를 종료하고, 아래 3단계 명령으로 원인을 파악합니다:

```bash title="터미널 — 원인 파악 3단계"
# 1. 이벤트에서 실패 원인 확인
kubectl describe pod crashloop-app

# 2. 현재 컨테이너 로그 확인 (이 실습은 sleep 명령이라 비어 있음)
kubectl logs crashloop-app

# 3. 이전 재시작 컨테이너 로그 확인 (RESTARTS 가 1 이상일 때만 동작)
kubectl logs crashloop-app --previous
```

!!! note "`--previous` 는 재시작 후에만 동작합니다"
    `RESTARTS` 숫자가 1 이상인 상태에서 실행하세요.
    Pod를 방금 생성했거나 아직 재시작이 없으면 `previous terminated container not found` 오류가 납니다.

!!! success "✅ 확인 포인트"
    `kubectl describe pod`의 Events 섹션에서 `Liveness probe failed` 메시지를 찾으면 OK.

---

### 정리

```bash title="터미널"
kubectl delete pod liveness-ok liveness-fail liveness-http crashloop-app
```

### 트러블슈팅

| 증상 | 확인 |
|------|------|
| Pod가 계속 재시작됨 | `kubectl describe pod` → Events에서 Liveness 실패 원인 확인 |
| 정상 앱인데 재시작됨 | `initialDelaySeconds` 늘리거나 Startup Probe 추가 |
| CrashLoopBackOff 상태 | `kubectl logs <pod> --previous`로 직전 재시작 로그 확인 |
| httpGet Probe 실패 | 앱이 해당 경로에서 200~399 응답을 반환하는지 확인 |

---

## 다음 단계

[:material-arrow-left: 환경 세팅](environment-setup.md){ .md-button }
[Probe 2 · Readiness :material-arrow-right:](probe-readiness.md){ .md-button .md-button--primary }
