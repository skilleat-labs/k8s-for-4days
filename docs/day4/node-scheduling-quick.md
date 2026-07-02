# 노드 스케줄링 실습

## 사전 확인 — 노드 이름 파악

AKS 노드 이름은 `aks-nodepool1-xxxxxxxx-vmss000000` 형태입니다.
실습 전에 실제 노드 이름을 먼저 확인합니다.

```bash
kubectl get nodes
```

출력 예시:

```
NAME                                STATUS   ROLES   AGE
aks-nodepool1-12345678-vmss000000   Ready    agent   1d
aks-nodepool1-12345678-vmss000001   Ready    agent   1d
```

이후 실습에서 `<노드1>`, `<노드2>` 자리에 실제 노드 이름을 넣으세요.

---

## 1) nodeSelector — 특정 노드에만 배포

### 노드에 label 추가

```bash
kubectl label node <노드1> role=web
kubectl label node <노드2> role=db
kubectl get nodes --show-labels | Select-String "role"   # PowerShell
kubectl get nodes --show-labels | grep role              # macOS/Linux
```

### label이 있는 노드에만 배포

```yaml
# pod-nodeselector.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-web
spec:
  nodeSelector:
    role: web
  containers:
    - name: app
      image: nginx:1.25
```

```bash
kubectl apply -f pod-nodeselector.yaml
kubectl get pod pod-web -o wide    # NODE 컬럼이 노드1인지 확인
kubectl delete pod pod-web
```

### label 제거

```bash
kubectl label node <노드1> role-
kubectl label node <노드2> role-
```

---

## 2) Taint & Toleration — 노드 격리

Taint는 노드가 특정 Pod를 거부하는 방식입니다.
Toleration이 없는 Pod는 Taint가 걸린 노드에 배포되지 않습니다.

### 노드2에 Taint 설정

```bash
kubectl taint node <노드2> dedicated=gpu:NoSchedule
```

### Toleration 없는 Pod → 노드2 배포 불가

```bash
kubectl create deployment test-taint --image=nginx:1.25 --replicas=4
kubectl get pods -o wide    # 모든 Pod가 노드1에만 배포됨
kubectl delete deployment test-taint
```

### Toleration 있는 Pod → 노드2 접근 가능

```yaml
# pod-toleration.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-toleration
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
  containers:
    - name: app
      image: nginx:1.25
```

```bash
kubectl apply -f pod-toleration.yaml
kubectl get pod pod-toleration -o wide    # 노드2에도 배포 가능
kubectl delete pod pod-toleration
```

### Taint 제거

```bash
kubectl taint node <노드2> dedicated=gpu:NoSchedule-
```

---

## 3) cordon / drain — 노드 점검

노드 점검 전 Pod를 안전하게 비우는 절차입니다.

```bash
# 테스트용 Deployment
kubectl create deployment drain-test --image=nginx:1.25 --replicas=4
kubectl get pods -o wide    # 두 노드에 분산 확인
```

### cordon — 새 Pod 배포 차단

```bash
kubectl cordon <노드2>
kubectl get nodes    # 노드2: Ready,SchedulingDisabled

kubectl scale deployment drain-test --replicas=6
kubectl get pods -o wide    # 새 Pod 2개가 노드1에만 배포됨
```

### drain — 기존 Pod까지 모두 이동

```bash
kubectl drain <노드2> --ignore-daemonsets --delete-emptydir-data
kubectl get pods -o wide    # 노드2의 Pod가 모두 노드1로 이동
```

### uncordon — 노드 복귀

```bash
kubectl uncordon <노드2>
kubectl get nodes    # 노드2: Ready

kubectl scale deployment drain-test --replicas=8
kubectl get pods -o wide    # 노드2에도 다시 배포됨
```

```bash
kubectl delete deployment drain-test
```

---

## 정리

| 기능 | 명령어 | 용도 |
|---|---|---|
| nodeSelector | `kubectl label node` + `spec.nodeSelector` | 특정 노드에 Pod 고정 |
| Taint | `kubectl taint node` | 노드에 Pod 접근 차단 |
| Toleration | `spec.tolerations` | Taint 있는 노드 접근 허용 |
| cordon | `kubectl cordon` | 새 Pod 배포 차단 |
| drain | `kubectl drain` | 기존 Pod 모두 이동 |
| uncordon | `kubectl uncordon` | 노드 정상 복귀 |
