# Resource Requests & Limits 실습

| 설정 | 누가 사용 | 역할 |
|---|---|---|
| `requests` | 스케줄러 | Pod 배치 시 노드 선택 기준 |
| `limits` | 컨테이너 런타임 | 실행 중 자원 상한 강제 |

!!! info "실습 환경 노드 스펙"
    현재 AKS 클러스터: **2 vCPU / 4GB × 2 노드**
    시스템 예약 후 실제 배치 가능한 용량은 노드당 약 **CPU 1.9 core / Memory 3.3GB** 입니다.

---

## 1) requests — 노드에 자원이 없으면 Pending

노드 용량(2 CPU / 4GB)을 초과하는 requests를 설정해 Pod가 배치되지 못하는 상황을 확인합니다.

```yaml
# pod-pending.yaml
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
          cpu: "4"       # 노드 최대 2 CPU → 배치 불가
          memory: 8Gi    # 노드 최대 4GB → 배치 불가
```

```bash
kubectl apply -f pod-pending.yaml
kubectl get pod pod-pending
kubectl describe pod pod-pending   # Events: Insufficient cpu, Insufficient memory 확인
kubectl delete pod pod-pending
```

---

## 2) limits CPU — Throttling (느려지지만 죽지 않음)

CPU를 무한히 점유하려는 컨테이너에 200m 상한을 걸어둡니다.
CPU는 limits를 초과해도 종료되지 않고 속도만 제한됩니다.

```yaml
# pod-cpu-limit.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-cpu-limit
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command: ["sh", "-c", "while true; do :; done"]
      resources:
        requests:
          cpu: 100m
        limits:
          cpu: 200m      # 2 vCPU 중 10%만 허용
```

```bash
kubectl apply -f pod-cpu-limit.yaml
kubectl get pod pod-cpu-limit      # Running 유지 — CPU 초과해도 종료 안됨
kubectl describe pod pod-cpu-limit
kubectl delete pod pod-cpu-limit
```

---

## 3) limits Memory — OOMKilled (즉시 강제 종료)

4GB 노드지만 limits를 20Mi로 낮게 설정하고 50MB 할당을 시도합니다.
Memory는 limits 초과 시 즉시 강제 종료됩니다.

```yaml
# pod-oom.yaml
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
        - "dd if=/dev/zero of=/dev/shm/waste bs=1M count=50"
      resources:
        requests:
          memory: 10Mi
        limits:
          memory: 20Mi     # 20MB 제한, 50MB 시도 → OOMKilled
```

```bash
kubectl apply -f pod-oom.yaml
kubectl get pod pod-oom -w         # OOMKilled 확인
kubectl describe pod pod-oom       # Reason: OOMKilled, Exit Code: 137
kubectl delete pod pod-oom
```

!!! info "CPU vs Memory limits 초과 동작 차이"
    - CPU 초과 → **Throttling** (속도 제한, 종료 없음)
    - Memory 초과 → **OOMKilled** (즉시 강제 종료)
