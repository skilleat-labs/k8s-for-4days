# Resource Requests & Limits 실습

| 설정 | 누가 사용 | 역할 |
|---|---|---|
| `requests` | 스케줄러 | Pod 배치 시 노드 선택 기준 |
| `limits` | 컨테이너 런타임 | 실행 중 자원 상한 강제 |

---

## 1) requests — 노드에 자원이 없으면 Pending

감당할 수 없는 requests를 설정해 Pod가 배치되지 못하는 상황을 확인합니다.

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
          cpu: "100"      # 100 core — 노드가 감당 불가
          memory: 100Gi
```

```bash
kubectl apply -f pod-pending.yaml
kubectl get pod pod-pending
kubectl describe pod pod-pending   # Events: Insufficient cpu 확인
kubectl delete pod pod-pending
```

---

## 2) limits CPU — Throttling (느려지지만 죽지 않음)

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
          cpu: 200m
```

```bash
kubectl apply -f pod-cpu-limit.yaml
kubectl get pod pod-cpu-limit      # Running 유지 — CPU 초과해도 종료 안됨
kubectl describe pod pod-cpu-limit
kubectl delete pod pod-cpu-limit
```

---

## 3) limits Memory — OOMKilled (즉시 강제 종료)

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
