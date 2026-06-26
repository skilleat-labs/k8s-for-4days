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

## 다음 단계

[:material-arrow-left: Probe 1 · Liveness](probe-liveness.md){ .md-button }
[Probe 3 · Startup :material-arrow-right:](probe-startup.md){ .md-button .md-button--primary }
