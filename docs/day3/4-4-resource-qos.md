# Resource QoS 실습

!!! info "예상 소요 40분"

---

!!! quote "이대리"
    *"requests랑 limits 차이는 이제 알지? 그 둘의 관계가 QoS Class를 결정해. 노드 자원이 부족할 때 K8s가 어떤 Pod를 먼저 죽이냐 — 그게 QoS야."*

---

## 이론

### QoS Class란?

K8s는 노드 메모리가 부족해지면 일부 Pod를 강제 종료(Eviction)합니다.
이때 **어떤 Pod를 먼저 죽일지** 결정하는 기준이 QoS Class입니다.

requests/limits 설정에 따라 K8s가 자동으로 QoS Class를 부여합니다.

| QoS Class | 조건 | Eviction 우선순위 |
|-----------|------|----------------|
| **BestEffort** | requests, limits 모두 없음 | **가장 먼저** 퇴거 |
| **Burstable** | requests < limits (또는 일부만 설정) | 두 번째 퇴거 |
| **Guaranteed** | requests == limits (모든 리소스) | **마지막** 퇴거 |

### 왜 이 순서인가?

```text title="Eviction 우선순위의 논리"
BestEffort   → "자원을 얼마나 쓸지 아무 약속도 안 했어. 먼저 비워."
Burstable    → "최소한의 약속은 했어. 근데 limits까지 쓸 수 있으니 위험할 수 있어."
Guaranteed   → "requests = limits. 사용량이 예측 가능하고 안정적이야. 마지막에 비워."
```

노드가 메모리 압박을 받으면:

```text title="Eviction 발생 흐름"
노드 메모리 부족 감지
    ↓
1단계: BestEffort Pod 강제 종료
    ↓ (still not enough)
2단계: Burstable Pod 중 가장 많이 쓰는 Pod 강제 종료
    ↓ (still not enough)
3단계: Guaranteed Pod 강제 종료 (최후의 수단)
```

---

## 실습

### Step 1. QoS Class 3종 만들고 비교하기

3가지 QoS Class Pod를 한번에 만들어서 나란히 비교합니다.

```yaml title="pod-besteffort.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-besteffort
spec:
  containers:
    - name: app
      image: nginx:1.25
      # resources 없음 → BestEffort
```

```yaml title="pod-burstable.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-burstable
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 512Mi   # requests < limits → Burstable
```

```yaml title="pod-guaranteed.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-guaranteed
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 200m
          memory: 256Mi
        limits:
          cpu: 200m
          memory: 256Mi   # requests == limits → Guaranteed
```

```bash title="터미널"
kubectl apply -f pod-besteffort.yaml
kubectl apply -f pod-burstable.yaml
kubectl apply -f pod-guaranteed.yaml
```

3개가 뜨면 QoS Class를 한눈에 비교합니다:

```bash title="터미널"
kubectl get pods pod-besteffort pod-burstable pod-guaranteed -o custom-columns=NAME:.metadata.name,QoS:.status.qosClass,CPU-REQ:.spec.containers[0].resources.requests.cpu,CPU-LIM:.spec.containers[0].resources.limits.cpu,MEM-REQ:.spec.containers[0].resources.requests.memory,MEM-LIM:.spec.containers[0].resources.limits.memory
```

```text title="출력 예시"
NAME             QoS          CPU-REQ   CPU-LIM   MEM-REQ   MEM-LIM
pod-besteffort   BestEffort   <none>    <none>    <none>    <none>
pod-burstable    Burstable    100m      500m      128Mi     512Mi
pod-guaranteed   Guaranteed   200m      200m      256Mi     256Mi
```

!!! success "✅ 확인 포인트"
    QoS Class가 자동으로 부여된 것 확인.
    requests/limits 관계가 그대로 Class를 결정함을 확인하세요.

---

### Step 2. Eviction 위험도 — 노드 상태로 확인

실제로 Eviction을 트리거하기는 어렵지만, 노드 상태와 Pod의 실제 사용량을 보면 위험도를 판단할 수 있습니다.

=== "macOS/Linux"
    ```bash title="터미널"
    kubectl describe node | grep -A 10 "Allocated resources"
    ```
=== "Windows PowerShell"
    ```powershell title="터미널"
    kubectl describe node | Select-String -Pattern "Allocated resources" -Context 0,10
    ```

```text title="출력 예시"
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                530m (26%)  700m (35%)
  memory             394Mi (10%) 768Mi (20%)
```

!!! note "Overcommit — 약속 초과"
    Requests 합계가 노드 총 자원을 넘으면 **Overcommit** 상태입니다.
    평소엔 문제없지만, 여러 Pod가 동시에 limits까지 치솟으면 노드가 압박받습니다.
    이때 BestEffort → Burstable 순서로 Eviction이 발생합니다.

현재 Pod들의 실제 메모리 사용량도 확인합니다:

```bash title="터미널"
kubectl top pod
```

!!! tip "Eviction 위험도를 판단하는 기준"
    실제 사용량이 requests에 근접하거나 넘으면 Eviction 후보가 됩니다.

    ```text
    pod-burstable:
      requests.memory: 128Mi
      실제 사용: 120Mi  ← requests에 근접 → Eviction 위험
    ```

---

Step 1 Pod를 정리하고 다음 실습을 진행합니다:

```bash title="터미널"
kubectl delete pod pod-besteffort pod-burstable pod-guaranteed
```

---

### Step 3. ResourceQuota — Quota 초과 시 Pod 생성 거부

ResourceQuota는 Namespace 전체의 자원 총량을 제한합니다.
Quota를 초과하면 **Pod 생성 자체가 거부**됩니다.

먼저 작은 Quota를 설정합니다:

```yaml title="resourcequota.yaml"
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab-quota
spec:
  hard:
    requests.cpu: "500m"
    requests.memory: "256Mi"
    limits.cpu: "1"
    limits.memory: "512Mi"
    count/pods: "5"
```

```bash title="터미널"
kubectl apply -f resourcequota.yaml
kubectl describe resourcequota lab-quota
```

```text title="출력 예시"
Name:            lab-quota
Namespace:       default
Resource         Used    Hard
--------         ----    ----
count/pods       0       5        ← 현재 Pod 없음, 최대 5개
limits.cpu       0       1
limits.memory    0       512Mi
requests.cpu     0       500m
requests.memory  0       256Mi
```

**Quota 초과 Pod 생성 시도:**

```yaml title="pod-quota-exceed.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-quota-exceed
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 200m
          memory: 200Mi
        limits:
          cpu: 500m
          memory: 400Mi
```

```bash title="터미널"
kubectl apply -f pod-quota-exceed.yaml
```

```text title="출력 예시 — 생성 거부"
Error from server (Forbidden): error when creating "pod-quota-exceed.yaml":
pods "pod-quota-exceed" is forbidden:
exceeded quota: lab-quota,
requested: limits.memory=400Mi,
used: limits.memory=768Mi,
limited: limits.memory=512Mi
```

!!! success "✅ 확인 포인트"
    `exceeded quota` 에러와 함께 Pod 생성이 거부되면 OK.
    `requested / used / limited` 세 수치를 비교하면 어디서 초과됐는지 바로 파악됩니다.

---

### Step 4. LimitRange — 기본값 자동 주입 확인

ResourceQuota가 있는 Namespace에서 requests/limits 없이 Pod를 만들면 생성이 거부됩니다.
LimitRange는 이 상황을 막기 위해 **기본값을 자동으로 채워줍니다**.

먼저 LimitRange 없이 requests 없는 Pod를 시도합니다:

```bash title="터미널"
kubectl run no-resources --image=nginx:1.25
```

```text title="출력 예시 — ResourceQuota가 있을 때"
Error from server (Forbidden): pods "no-resources" is forbidden:
failed quota: lab-quota:
must specify limits.cpu for: no-resources;
must specify limits.memory for: no-resources;
must specify requests.cpu for: no-resources;
must specify requests.memory for: no-resources
```

이제 LimitRange를 적용합니다:

```yaml title="limitrange.yaml"
apiVersion: v1
kind: LimitRange
metadata:
  name: lab-limitrange
spec:
  limits:
    - type: Container
      default:           # limits 기본값
        cpu: 200m
        memory: 256Mi
      defaultRequest:    # requests 기본값
        cpu: 100m
        memory: 128Mi
      max:               # 최대 상한 (이 이상 설정 불가)
        cpu: "1"
        memory: 1Gi
      min:               # 최소 하한 (이 이하 설정 불가)
        cpu: 50m
        memory: 64Mi
```

```bash title="터미널"
kubectl apply -f limitrange.yaml
```

LimitRange 적용 후 다시 resources 없는 Pod를 만들어봅니다:

=== "macOS/Linux"
    ```bash title="터미널"
    kubectl run no-resources --image=nginx:1.25
    kubectl get pod no-resources -o yaml | grep -A 12 "resources:"
    ```
=== "Windows PowerShell"
    ```powershell title="터미널"
    kubectl run no-resources --image=nginx:1.25
    kubectl get pod no-resources -o yaml | Select-String -Pattern "resources:" -Context 0,12
    ```

```text title="출력 예시 — 기본값이 자동 주입됨"
    resources:
      limits:
        cpu: 200m        ← LimitRange default에서 자동 주입
        memory: 256Mi    ← LimitRange default에서 자동 주입
      requests:
        cpu: 100m        ← LimitRange defaultRequest에서 자동 주입
        memory: 128Mi    ← LimitRange defaultRequest에서 자동 주입
```

!!! success "✅ 확인 포인트"
    YAML에 resources를 명시하지 않았는데 기본값이 자동으로 들어간 것 확인.

**max 초과 시 거부 확인:**

```yaml title="pod-over-max.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-over-max
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 2000m     # LimitRange max(1 core)를 초과
          memory: 128Mi
        limits:
          cpu: 2000m
          memory: 256Mi
```

```bash title="터미널"
kubectl apply -f pod-over-max.yaml
```

```text title="출력 예시"
Error from server (Forbidden):
max cpu usage per Container is 1, but limit is 2.
```

!!! success "✅ 확인 포인트"
    LimitRange의 `max`를 초과한 Pod가 거부되면 OK.
    ResourceQuota가 전체 합계를 제한한다면, LimitRange는 개별 컨테이너의 상한/하한을 제한합니다.

---

### 정리 및 리소스 삭제

```bash title="터미널"
kubectl delete pod no-resources pod-quota-exceed pod-over-max --ignore-not-found
kubectl delete resourcequota lab-quota
kubectl delete limitrange lab-limitrange
```

### ResourceQuota vs LimitRange 비교

| | ResourceQuota | LimitRange |
|---|---|---|
| 적용 단위 | **Namespace 전체** 합계 | **컨테이너 개별** |
| 역할 | Namespace 총 자원 상한 | 컨테이너별 기본값·상한·하한 |
| 주요 효과 | 초과 시 Pod 생성 거부 | 기본값 자동 주입, 범위 밖 거부 |
| 함께 쓰면 | Namespace 총량 제어 | 개별 과점유 방지 + 기본값 보장 |

### 트러블슈팅

| 증상 | 확인 |
|------|------|
| Pod 생성 시 `exceeded quota` | `kubectl describe resourcequota` → used/hard 비교 |
| resources 없는 Pod 생성 거부 | LimitRange 미설정 → `kubectl apply -f limitrange.yaml` |
| `max cpu usage per Container is X` | LimitRange max 초과 → requests/limits 값 줄이기 |

---

## 다음 단계

[:material-arrow-left: K8s 2 · requests & limits](resource-requests-limits.md){ .md-button }
[K8s 3 · HPA :material-arrow-right:](hpa.md){ .md-button .md-button--primary }
