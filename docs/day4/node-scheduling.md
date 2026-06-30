# 노드 스케줄링 — 배치 제어 실습

## 실습 목표

- `nodeSelector` / `affinity`로 특정 노드에 Pod를 유도한다.
- `taints & tolerations`으로 노드를 전용 격리한다.
- `cordon / drain`으로 노드 점검 시 Pod를 안전하게 비운다.

---

## 사전 확인

```bash
kubectl get nodes -o wide
kubectl get nodes --show-labels
```

이 실습은 **worker-1, worker-2** 두 노드가 있는 환경을 기준으로 합니다.

---

## Part 1 — nodeSelector

### 개념

가장 단순한 노드 지정 방법입니다.
노드에 **label을 붙이고**, Pod에 `nodeSelector`로 그 label을 지정하면
해당 label을 가진 노드에만 배포됩니다.

```
[worker-1]  label: role=web    ← Pod가 여기 배포됨
[worker-2]  label: role=db
```

### 1-1. 노드에 label 추가

```bash
kubectl label node worker-1 role=web
kubectl label node worker-2 role=db
```

=== "macOS/Linux"
    ```bash
    kubectl get nodes --show-labels | grep role
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl get nodes --show-labels | Select-String "role"
    ```

### 1-2. nodeSelector로 특정 노드에 배포

`pod-nodeselector.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-web
spec:
  containers:
    - name: app
      image: nginx:1.25
  nodeSelector:
    role: web
```

```bash
kubectl apply -f pod-nodeselector.yaml
kubectl get pod pod-web -o wide
```

!!! success "확인 포인트"
    `NODE` 컬럼이 `worker-1`인지 확인하세요.

### 1-3. label이 없는 노드에 배포 시도 (실패 확인)

`pod-nodeselector-fail.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-fail
spec:
  containers:
    - name: app
      image: nginx:1.25
  nodeSelector:
    role: gpu     # 어떤 노드에도 없는 label
```

```bash
kubectl apply -f pod-nodeselector-fail.yaml
kubectl get pod pod-fail
kubectl describe pod pod-fail
```

Events에서 아래 메시지를 확인합니다:

```
Warning  FailedScheduling  0/2 nodes are available: node(s) didn't match Pod's node affinity/selector
```

```bash
kubectl delete pod pod-web pod-fail
```

---

## Part 2 — Node Affinity

### 개념

`nodeSelector`보다 세밀한 규칙을 설정할 수 있습니다.

| 타입 | 동작 |
|------|------|
| `requiredDuringSchedulingIgnoredDuringExecution` | **반드시** 해당 노드여야 함 (nodeSelector와 동일한 강도) |
| `preferredDuringSchedulingIgnoredDuringExecution` | **가능하면** 해당 노드 우선, 없으면 다른 노드도 허용 |

### 2-1. required — 반드시 worker-1에 배포

`pod-affinity-required.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-required
spec:
  containers:
    - name: app
      image: nginx:1.25
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: role
                operator: In
                values:
                  - web
```

```bash
kubectl apply -f pod-affinity-required.yaml
kubectl get pod pod-affinity-required -o wide
# NODE가 worker-1인지 확인
```

### 2-2. preferred — worker-2 우선, 없으면 다른 곳도 허용

`pod-affinity-preferred.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-preferred
spec:
  containers:
    - name: app
      image: nginx:1.25
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: role
                operator: In
                values:
                  - db
```

```bash
kubectl apply -f pod-affinity-preferred.yaml
kubectl get pod pod-affinity-preferred -o wide
# worker-2에 배포되는 것을 확인
# worker-2가 꽉 찼거나 불가능한 상황이면 worker-1에도 배포될 수 있음
```

### 2-3. operator 종류

| operator | 의미 |
|----------|------|
| `In` | label 값이 목록 중 하나와 일치 |
| `NotIn` | label 값이 목록에 없음 |
| `Exists` | key가 존재하면 (값 무관) |
| `DoesNotExist` | key가 존재하지 않으면 |

### 정리

```bash
kubectl delete pod pod-affinity-required pod-affinity-preferred
```

---

## Part 3 — Taints & Tolerations

### 개념

Affinity는 Pod가 특정 노드를 "선호"하는 방식이지만,
**Taint**는 노드가 특정 Pod를 "거부"하는 방식입니다.

```
[worker-2]  Taint: dedicated=gpu:NoSchedule
  → Toleration 없는 Pod는 이 노드에 배포 불가
  → Toleration 있는 Pod만 배포 허용
```

**Taint Effect 종류:**

| Effect | 동작 |
|--------|------|
| `NoSchedule` | Toleration 없는 새 Pod는 스케줄하지 않음 (기존 Pod는 유지) |
| `PreferNoSchedule` | 가능하면 스케줄 안 함, 다른 노드 없으면 허용 |
| `NoExecute` | 기존에 실행 중인 Pod도 즉시 퇴출 |

### 3-1. worker-2에 Taint 설정

```bash
kubectl taint node worker-2 dedicated=gpu:NoSchedule
```

=== "macOS/Linux"
    ```bash
    # 확인
    kubectl describe node worker-2 | grep Taints
    ```
=== "Windows PowerShell"
    ```powershell
    # 확인
    kubectl describe node worker-2 | Select-String "Taints"
    ```

출력:
```
Taints: dedicated=gpu:NoSchedule
```

### 3-2. Toleration 없는 Pod → worker-2 배포 불가 확인

```bash
kubectl create deployment test-taint --image=nginx:1.25 --replicas=4
kubectl get pods -o wide
```

!!! success "확인 포인트"
    4개 Pod가 모두 **worker-1**에만 배포되는 것을 확인하세요.
    worker-2는 Taint 때문에 거부합니다.

```bash
kubectl delete deployment test-taint
```

### 3-3. Toleration 있는 Pod → worker-2에도 배포 가능

`pod-toleration.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-toleration
spec:
  containers:
    - name: app
      image: nginx:1.25
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
```

```bash
kubectl apply -f pod-toleration.yaml
kubectl get pod pod-toleration -o wide
# worker-2에도 배포 가능해짐 (worker-1에 배포될 수도 있음)
```

!!! info "Toleration은 '허용'이지 '강제'가 아닙니다"
    Toleration이 있어도 스케줄러가 worker-1에 배포할 수 있습니다.
    worker-2에 **반드시** 배포하려면 `nodeSelector` 또는 `affinity`와 함께 사용하세요.

### 3-4. Taint + Affinity 조합 — worker-2 전용 격리

`pod-gpu-dedicated.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-gpu-dedicated
spec:
  containers:
    - name: app
      image: nginx:1.25
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: role
                operator: In
                values:
                  - db
```

```bash
kubectl apply -f pod-gpu-dedicated.yaml
kubectl get pod pod-gpu-dedicated -o wide
# worker-2에만 배포됨
```

### 3-5. Taint 제거

```bash
kubectl taint node worker-2 dedicated=gpu:NoSchedule-
# 마지막에 - 를 붙이면 제거
```

=== "macOS/Linux"
    ```bash
    kubectl describe node worker-2 | grep Taints
    # Taints: <none> 확인
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl describe node worker-2 | Select-String "Taints"
    # Taints: <none> 확인
    ```

### 정리

```bash
kubectl delete pod pod-toleration pod-gpu-dedicated
```

---

## Part 4 — cordon / drain

### 개념

노드 점검(재부팅, 업그레이드 등) 전에 **트래픽과 Pod를 안전하게 비우는** 절차입니다.

```
cordon  →  노드를 스케줄 불가(SchedulingDisabled) 상태로 만듦
           기존 Pod는 그대로 유지, 새 Pod만 배포 안됨

drain   →  기존 Pod를 다른 노드로 모두 이동 + cordon 자동 적용
           노드를 완전히 비움

uncordon →  점검 후 노드를 다시 정상 상태로 복귀
```

### 4-1. 테스트용 Deployment 배포

```bash
kubectl create deployment drain-test --image=nginx:1.25 --replicas=4
kubectl get pods -o wide
# worker-1, worker-2에 분산 배포 확인
```

### 4-2. cordon — 새 Pod 배포 차단

```bash
kubectl cordon worker-2

kubectl get nodes
```

출력:
```
NAME            STATUS                     ROLES           AGE
control-plane   Ready                      control-plane   1h
worker-1        Ready                      <none>          1h
worker-2        Ready,SchedulingDisabled   <none>          1h
```

```bash
# 새 Pod 추가 → worker-1에만 배포되는지 확인
kubectl scale deployment drain-test --replicas=6
kubectl get pods -o wide
# 새로 생긴 2개 Pod가 worker-1에만 배포됨
```

!!! info
    cordon은 기존에 worker-2에서 실행 중인 Pod는 그대로 둡니다.
    단지 새로운 Pod가 이 노드에 배포되지 않도록 막는 것입니다.

### 4-3. drain — 기존 Pod까지 모두 이동

```bash
kubectl drain worker-2 --ignore-daemonsets --delete-emptydir-data
```

```bash
kubectl get pods -o wide
# worker-2의 Pod가 모두 worker-1로 이동
kubectl get nodes
# worker-2: Ready,SchedulingDisabled 유지
```

!!! warning "옵션 설명"
    - `--ignore-daemonsets`: DaemonSet Pod는 drain 대상에서 제외 (노드에 항상 있어야 하는 Pod)
    - `--delete-emptydir-data`: emptyDir 데이터 삭제 허용 (없으면 drain 거부됨)

!!! info "실제 노드 점검 흐름"
    ```
    kubectl drain worker-2 ...   # Pod 이동
    → (노드 재부팅 / 업그레이드 / 점검)
    kubectl uncordon worker-2    # 복귀
    ```

### 4-4. uncordon — 노드 복귀

점검이 끝났다고 가정하고 노드를 다시 활성화합니다.

```bash
kubectl uncordon worker-2

kubectl get nodes
# worker-2: Ready 상태로 복귀
```

```bash
# 새 Pod 배포 → worker-2에도 다시 배포되는지 확인
kubectl scale deployment drain-test --replicas=8
kubectl get pods -o wide
```

!!! success "확인 포인트"
    uncordon 후 새 Pod가 worker-2에도 배포되면 정상 복귀된 것입니다.
    단, drain으로 이동한 기존 Pod는 자동으로 돌아오지 않습니다.

### 정리

```bash
kubectl delete deployment drain-test
kubectl label node worker-1 role-
kubectl label node worker-2 role-
```

---

## 전체 비교 정리

| 기능 | 누가 제어 | 방향 | 주요 용도 |
|------|---------|------|---------|
| `nodeSelector` | Pod | Pod → 노드 선택 | 단순 노드 지정 |
| `nodeAffinity` | Pod | Pod → 노드 선택 | 복잡한 조건, 우선순위 |
| `taint` | 노드 | 노드 → Pod 거부 | 전용 노드 격리 |
| `toleration` | Pod | Pod → taint 허용 | 격리 노드 접근 허가 |
| `cordon` | 운영자 | 새 Pod 차단 | 점검 전 준비 |
| `drain` | 운영자 | 기존 Pod 이동 | 노드 완전히 비우기 |
| `uncordon` | 운영자 | 노드 복귀 | 점검 후 정상화 |

---

## 트러블슈팅

| 증상 | 확인 사항 |
|------|---------|
| Pod가 Pending 상태 | `kubectl describe pod`로 스케줄 실패 이유 확인 |
| nodeSelector 지정했는데 다른 노드에 배포됨 | label 이름/값 오타 확인 (`kubectl get nodes --show-labels`) |
| drain이 멈춤 | PodDisruptionBudget 또는 emptyDir 데이터 문제 → `--delete-emptydir-data` 옵션 확인 |
| drain 후 Pod가 worker-2로 안 돌아옴 | 정상 동작. uncordon 후 새 배포부터 분산됨 |
| taint 제거가 안됨 | 명령어 끝에 `-` 붙였는지 확인 (`key=value:Effect-`) |
