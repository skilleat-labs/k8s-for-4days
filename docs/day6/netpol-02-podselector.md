# NetworkPolicy 02 — PodSelector 실습

## 실습 목표

- `podSelector`로 **특정 Pod만** 통신을 허용한다.
- 라벨을 이용해 허용 범위를 세밀하게 제어한다.
- 같은 네임스페이스 안에서의 Pod 간 통신을 컨트롤한다.

---

## 개념

### PodSelector

NetworkPolicy에서 PodSelector는 **두 곳**에 등장합니다.

```yaml
spec:
  podSelector:          # ① 이 정책이 "적용되는" Pod
    matchLabels:
      app: server
  ingress:
    - from:
        - podSelector:  # ② 트래픽을 "허용할" 소스 Pod
            matchLabels:
              role: frontend
```

| 위치 | 역할 |
|------|------|
| `spec.podSelector` | 이 NetworkPolicy의 대상 Pod (수신자) |
| `ingress[].from[].podSelector` | 트래픽을 보낼 수 있는 Pod (발신자) |

---

## 실습 환경 확인

이전 실습에서 생성한 리소스를 사용합니다. 없다면 아래 명령으로 다시 생성합니다.

```bash
# 네임스페이스
kubectl create namespace netpol-demo 2>/dev/null || true
kubectl create namespace netpol-client 2>/dev/null || true

# Pod
kubectl run server \
  --image=nginx:alpine \
  --namespace=netpol-demo \
  --labels="app=server" 2>/dev/null || true

kubectl run client-a \
  --image=curlimages/curl:latest \
  --namespace=netpol-demo \
  --labels="app=client,role=frontend" \
  --command -- sleep infinity 2>/dev/null || true

kubectl run client-b \
  --image=curlimages/curl:latest \
  --namespace=netpol-client \
  --labels="app=client" \
  --command -- sleep infinity 2>/dev/null || true

# Service
kubectl expose pod server --port=80 --name=server-svc \
  --namespace=netpol-demo 2>/dev/null || true
```

### 라벨 확인

```bash
kubectl get pods -n netpol-demo --show-labels
```

```
NAME       READY   STATUS    LABELS
client-a   1/1     Running   app=client,role=frontend
server     1/1     Running   app=server
```

---

## Step 1. Deny-All 적용 (기준선 설정)

```yaml title="deny-all.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: netpol-demo
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

```bash
kubectl apply -f deny-all.yaml
```

차단 상태 확인:

```bash
kubectl exec -n netpol-demo client-a -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local
```

```
curl: (28) Connection timed out after 3001 milliseconds
```

---

## Step 2. PodSelector로 특정 Pod 허용

`role=frontend` 라벨을 가진 Pod에서만 `server`로 접근을 허용합니다.

```yaml title="allow-frontend.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: netpol-demo
spec:
  podSelector:           # (1) server Pod에 적용
    matchLabels:
      app: server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:   # (2) role=frontend 라벨의 Pod만 허용
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl apply -f allow-frontend.yaml
```

---

## Step 3. 결과 확인

### client-a (role=frontend 있음) — 허용 예상 ✅

```bash
kubectl exec -n netpol-demo client-a -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local | head -3
```

```html
<!DOCTYPE html>
<html>
<head><title>Welcome to nginx!</title></head>
```

### 새 Pod (role=frontend 없음) — 차단 예상 🚫

`role=frontend` 라벨이 없는 Pod를 추가해 테스트합니다.

```bash
kubectl run intruder \
  --image=curlimages/curl:latest \
  --namespace=netpol-demo \
  --labels="app=intruder" \
  --command -- sleep infinity

kubectl exec -n netpol-demo intruder -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local
```

```
curl: (28) Connection timed out after 3001 milliseconds
```

### client-b (다른 네임스페이스) — 차단 예상 🚫

```bash
kubectl exec -n netpol-client client-b -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local
```

!!! info "왜 차단되나?"
    `podSelector`는 같은 네임스페이스 안의 Pod만 매칭합니다.
    `netpol-client` 네임스페이스의 `client-b`는 `podSelector`로 매칭되지 않습니다.
    다른 네임스페이스를 허용하려면 `namespaceSelector`가 필요합니다 (다음 실습).

---

## Step 4. matchExpressions 사용

`matchLabels` 대신 `matchExpressions`로 더 유연하게 선택할 수 있습니다.

```yaml title="allow-multi-role.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-multi-role
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchExpressions:
              - key: role
                operator: In
                values:
                  - frontend
                  - backend     # frontend 또는 backend 라벨이 있는 Pod 허용
      ports:
        - protocol: TCP
          port: 80
```

```bash
# 기존 정책 교체
kubectl delete networkpolicy allow-frontend -n netpol-demo
kubectl apply -f allow-multi-role.yaml

# backend 라벨 Pod 추가 테스트
kubectl run backend-pod \
  --image=curlimages/curl:latest \
  --namespace=netpol-demo \
  --labels="app=backend,role=backend" \
  --command -- sleep infinity

kubectl exec -n netpol-demo backend-pod -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local | head -3
```

---

## Step 5. 여러 ingress 규칙 (OR 조건)

`ingress` 배열의 각 항목은 **OR** 조건입니다.

```yaml title="allow-multiple-sources.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-multiple-sources
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
    - Ingress
  ingress:
    - from:                    # 조건 1: role=frontend
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - port: 80
    - from:                    # 조건 2 (OR): app=monitoring
        - podSelector:
            matchLabels:
              app: monitoring
      ports:
        - port: 80
```

```bash
kubectl apply -f allow-multiple-sources.yaml
```

---

## Step 6. 정리

```bash
kubectl delete networkpolicy --all -n netpol-demo
kubectl delete pod intruder backend-pod -n netpol-demo --ignore-not-found
```

---

## 핵심 요약

| 패턴 | 의미 |
|------|------|
| `podSelector: {}` | 네임스페이스의 **모든** Pod 선택 |
| `podSelector: {matchLabels: {k:v}}` | 특정 라벨을 가진 Pod만 선택 |
| `matchExpressions` + `In` | 여러 값 중 하나를 가진 Pod 선택 |
| `ingress` 배열의 여러 항목 | **OR** 조건 (하나라도 맞으면 허용) |
| `from` 배열의 여러 항목 | **OR** 조건 (하나라도 맞으면 허용) |

!!! warning "from[] 안에 여러 항목 vs ingress[] 여러 항목"
    `from: [{podSelector: A}, {namespaceSelector: B}]` → A **또는** B (OR)

    `from: [{podSelector: A, namespaceSelector: B}]` → A **이고** B (AND)

    이 차이는 다음 실습에서 자세히 다룹니다.

!!! info "다음 실습"
    다음에는 **NamespaceSelector**로 다른 네임스페이스의 Pod를 허용하는 방법을 배웁니다.
