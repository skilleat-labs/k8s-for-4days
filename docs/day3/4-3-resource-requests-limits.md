# K8s 2 · Resource requests & limits

!!! info "예상 소요 40분"

---

!!! quote "이대리"
    *"requests랑 limits, 둘 다 '자원 설정'이라고 생각하면 헷갈려. 얘네는 역할이 완전히 달라. requests는 스케줄러한테 하는 말이고, limits는 런타임한테 하는 말이야."*

---

## 이론

### 두 가지 다른 역할

requests와 limits는 이름이 비슷하지만 **동작하는 시점과 대상이 완전히 다릅니다.**

```text title="requests vs limits 역할 분리"
Pod 생성 요청
    ↓
[스케줄러]  requests를 본다
            "이 Pod를 올리려면 CPU 200m, Memory 256Mi가 필요해.
             어느 노드에 이만큼 여유가 있지?"
            → 노드 선택 후 배치
    ↓
[컨테이너 런타임]  limits를 본다
                   "이 컨테이너는 CPU 500m, Memory 512Mi 이상
                    절대 못 쓰게 막아."
                   → 실행 중 상한 강제
```

| 설정 | 누가 사용 | 역할 |
|------|---------|------|
| `requests` | **스케줄러** | Pod 배치 시 노드 선택 기준 |
| `limits` | **컨테이너 런타임** | 실행 중 자원 상한 강제 |

### CPU와 Memory — limits 초과 시 동작이 다릅니다

```text title="초과 시 동작 차이"
CPU limits 초과  →  Throttling (속도 제한, 종료 없음)
Memory limits 초과  →  OOMKilled (즉시 강제 종료)
```

CPU는 "빌려 쓰는" 개념이라 잠시 넘어도 나중에 줄이면 됩니다.
Memory는 이미 점유한 것을 즉시 돌려줄 방법이 없어서 프로세스를 강제 종료합니다.

---

## 실습

### Step 1. requests — 스케줄러의 배치 기준

requests는 **실제 사용량이 아니라 스케줄러에 하는 약속**입니다.
요청한 requests만큼 노드에 여유가 없으면 Pod는 배치되지 않고 `Pending` 상태가 됩니다.

아무 노드도 수용할 수 없는 큰 requests를 설정해봅니다:

```yaml title="pod-pending.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-pending
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: "100"        # 100 core 요청 — 노드가 감당 불가
          memory: 100Gi
```

```bash title="터미널"
kubectl apply -f pod-pending.yaml
kubectl get pod pod-pending -w
```

```text title="출력 예시"
NAME          READY   STATUS    RESTARTS   AGE
pod-pending   0/1     Pending   0          10s   ← 배치될 노드가 없음
```

```bash title="터미널"
kubectl describe pod pod-pending
```

```text title="출력 예시 — Events 섹션"
Events:
  Warning  FailedScheduling  0/1 nodes are available:
           1 Insufficient cpu, 1 Insufficient memory.
```

!!! success "✅ 확인 포인트"
    `Pending` 상태 + `Insufficient cpu` 메시지 확인.
    Pod가 실행조차 안 된 이유가 순전히 **requests** 때문임을 기억하세요.

!!! note "requests ≠ 실제 사용량"
    requests.cpu: 200m을 설정해도 앱이 실제로 200m을 쓰는 게 아닙니다.
    스케줄러가 "이 노드에 200m 여유 있어?"를 확인할 때만 쓰입니다.
    실제 앱이 10m밖에 안 쓰더라도 스케줄러는 200m 예약으로 계산합니다.

정리:

```bash title="터미널"
kubectl delete pod pod-pending
```

---

### 사전 준비 — metrics-server 활성화

!!! warning "Rancher Desktop 필수 설정"
    `kubectl top` 명령은 **metrics-server**가 있어야 동작합니다.
    Rancher Desktop(k3s)은 기본 비활성화 상태입니다. Step 2, 4 전에 한 번만 실행하세요.

    ```bash title="터미널"
    # metrics-server 설치
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

    # k3s는 kubelet TLS 검증 우회 필요 (Mac/Linux/Windows 공통 — 한 줄로 실행)
    kubectl patch deployment metrics-server -n kube-system --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

    # 준비 확인 (1~2분 소요)
    kubectl rollout status deployment/metrics-server -n kube-system
    ```

    ```text title="출력 예시"
    deployment "metrics-server" successfully rolled out
    ```

---

### Step 2. CPU limits → Throttling (느려지지만 죽지 않음)

CPU를 최대한 사용하려는 컨테이너에 limits.cpu를 걸어봅니다.
CPU는 limits를 초과해도 **종료되지 않고 속도만 제한(Throttling)**됩니다.

```yaml title="pod-cpu-limit.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-cpu-limit
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "while true; do :; done"   # CPU를 무한히 점유하는 루프
      resources:
        requests:
          cpu: 100m
        limits:
          cpu: 200m                  # 200m 이상은 못 씀
```

```bash title="터미널"
kubectl apply -f pod-cpu-limit.yaml
kubectl get pod pod-cpu-limit
```

Pod가 `Running` 상태인지 확인 후, 실제 CPU 사용량을 측정합니다:

```bash title="터미널"
kubectl top pod pod-cpu-limit
```

```text title="출력 예시"
NAME            CPU(cores)   MEMORY(bytes)
pod-cpu-limit   200m         0Mi            ← 200m에서 딱 막힘
```

!!! success "✅ 확인 포인트"
    CPU 사용량이 `200m` 언저리에서 멈추면 OK.
    앱은 100% 사용하려 하지만 limits가 200m에서 잘라냅니다.

!!! tip "Throttling이 언제 문제가 되나요?"
    limits.cpu가 너무 낮으면 앱이 느려집니다.
    API 응답 시간이 갑자기 길어진다면 CPU Throttling이 원인일 수 있습니다.
    `kubectl top pod`로 limits에 딱 붙어있는지 확인하세요.

정리:

```bash title="터미널"
kubectl delete pod pod-cpu-limit
```

---

### Step 3. Memory limits → OOMKilled (즉시 강제 종료)

Memory는 CPU와 다르게 limits를 초과하면 **즉시 컨테이너가 강제 종료**됩니다.

limits.memory: 20Mi인데 50MB를 쓰려는 컨테이너를 만들어봅니다:

```yaml title="pod-oom.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-oom
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - |
          echo "메모리 50MB 할당 시도..."
          dd if=/dev/zero of=/dev/shm/waste bs=1M count=50
          echo "완료 (여기까지 오면 안 됨)"
          sleep 3600
      resources:
        requests:
          memory: 10Mi
        limits:
          memory: 20Mi     # 20MB 제한, 50MB 시도 → OOMKilled
```

```bash title="터미널"
kubectl apply -f pod-oom.yaml
kubectl get pod pod-oom -w
```

```text title="출력 예시"
NAME      READY   STATUS      RESTARTS   AGE
pod-oom   0/1     OOMKilled   0          3s    ← 즉시 강제 종료
pod-oom   0/1     CrashLoopBackOff  1    8s
```

```bash title="터미널"
kubectl describe pod pod-oom
```

```text title="출력 예시 — Containers 섹션"
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       OOMKilled         ← 메모리 초과로 강제 종료
      Exit Code:    137
```

!!! success "✅ 확인 포인트"
    `OOMKilled`, Exit Code `137` 확인. CPU Throttling과 달리 즉시 종료됩니다.

!!! warning "OOMKilled vs CrashLoopBackOff"
    OOMKilled로 종료되면 K8s가 재시작을 시도합니다.
    매번 OOMKilled가 반복되면 `CrashLoopBackOff` 상태가 됩니다.
    이 경우 `limits.memory`를 늘리거나 앱의 메모리 누수를 확인하세요.

정리:

```bash title="터미널"
kubectl delete pod pod-oom
```

---

### Step 4. limits 없을 때의 위험

limits를 설정하지 않으면 어떻게 될까요?
스케줄러는 requests 기준으로 배치하지만, 런타임 제한이 없어서 **실제 사용량이 무제한**이 됩니다.

```yaml title="pod-no-limits.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-no-limits
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "while true; do :; done"
      resources:
        requests:
          cpu: 100m          # 스케줄러: "100m짜리 Pod구나"
          memory: 64Mi
        # limits 없음 → 런타임 제한 없음
```

```bash title="터미널"
kubectl apply -f pod-no-limits.yaml
kubectl top pod pod-no-limits
```

```text title="출력 예시"
NAME            CPU(cores)   MEMORY(bytes)
pod-no-limits   1200m        1Mi    ← requests는 100m이지만 실제 1200m 사용
```

!!! success "✅ 확인 포인트"
    requests(100m)보다 훨씬 많은 CPU를 사용하는 것을 확인.

!!! danger "limits 미설정이 위험한 이유"
    노드의 다른 Pod들이 같은 노드에 있다면:

    ```text
    [노드 총 CPU: 2 core]
    pod-no-limits  → 1200m 사용 (limits 없음, 무제한)
    order-api      → CPU 부족으로 throttling 발생
    payment-api    → CPU 부족으로 응답 지연
    ```

    **한 Pod가 노드 자원을 독식해서 다른 Pod에 장애를 일으킬 수 있습니다.**
    운영 환경에서는 반드시 limits를 설정하세요.

정리:

```bash title="터미널"
kubectl delete pod pod-no-limits
```

---

## 핵심 정리

```text title="requests / limits 설계 공식"
requests = 앱의 평균 사용량   (스케줄러 배치 기준)
limits   = 앱의 피크 사용량 + 20~30% 여유   (런타임 상한)

예시:
  평균 CPU  150m → requests.cpu: 150m
  피크 CPU  400m → limits.cpu:   500m

  평균 Memory 200Mi → requests.memory: 200Mi
  피크 Memory 400Mi → limits.memory:   512Mi
```

| 상황 | 원인 | 해결 |
|------|------|------|
| Pod `Pending` | requests가 노드 여유보다 큼 | requests 줄이거나 노드 추가 |
| 앱 응답이 느려짐 | CPU Throttling | `kubectl top pod`로 limits.cpu 여유 확인 |
| Pod 재시작, `OOMKilled` | 메모리 limits 초과 | limits.memory 증가 또는 메모리 누수 확인 |
| 노드 자원 고갈 | limits 미설정 Pod | 반드시 limits 설정 |

---

## 다음 단계

[:material-arrow-left: Probe 3 · Startup](probe-startup.md){ .md-button }
[K8s 2-2 · QoS & ResourceQuota :material-arrow-right:](resource-qos.md){ .md-button .md-button--primary }
