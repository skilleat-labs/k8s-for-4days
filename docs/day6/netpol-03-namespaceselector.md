# NetworkPolicy 03 — NamespaceSelector 실습

## 실습 목표

- `namespaceSelector`로 **특정 네임스페이스**에서 오는 트래픽을 허용한다.
- `podSelector`와 `namespaceSelector`를 **AND / OR**로 조합하는 방법을 이해한다.
- 실무 패턴(모니터링 NS 허용, 운영 NS만 허용)을 실습한다.

---

## 개념

### NamespaceSelector

`namespaceSelector`는 **네임스페이스에 붙은 라벨**로 매칭합니다.

```
netpol-demo NS          netpol-client NS        monitoring NS
┌──────────┐            ┌──────────────┐         ┌─────────────┐
│  server  │ ◀─ 차단 ── │  client-b    │         │  prometheus │
│  (80)   │ ◀─ 허용 ── │              │ ← env=  │             │
└──────────┘            └──────────────┘   prod  └─────────────┘
                                                   ↑ env=monitoring
```

### AND vs OR 핵심 차이

```yaml
# 케이스 A: OR (각각 독립적)
from:
  - podSelector:
      matchLabels: {role: frontend}
  - namespaceSelector:
      matchLabels: {env: prod}
# → role=frontend Pod 이거나, env=prod 네임스페이스의 모든 Pod

# 케이스 B: AND (같은 from 항목에 둘 다)
from:
  - podSelector:
      matchLabels: {role: frontend}
    namespaceSelector:
      matchLabels: {env: prod}
# → env=prod 네임스페이스에 있는 role=frontend Pod 만
```

!!! danger "들여쓰기 실수 주의"
    `-`(대시) 위치에 따라 AND/OR가 달라집니다.
    `-`가 하나이면 같은 항목 안에서 AND, `-`가 두 개이면 OR입니다.

---

## 실습 환경 준비

### 네임스페이스에 라벨 추가

NetworkPolicy의 `namespaceSelector`는 **네임스페이스 라벨**을 기준으로 합니다.

```bash
# netpol-demo에 env=demo 라벨 추가
kubectl label namespace netpol-demo env=demo

# netpol-client에 env=prod 라벨 추가
kubectl label namespace netpol-client env=prod

# monitoring 네임스페이스 생성 및 라벨 추가
kubectl create namespace monitoring
kubectl label namespace monitoring purpose=monitoring

# 라벨 확인
kubectl get namespaces --show-labels
```

```
NAME             LABELS
default          ...
monitoring       kubernetes.io/metadata.name=monitoring,purpose=monitoring
netpol-client    env=prod,kubernetes.io/metadata.name=netpol-client
netpol-demo      env=demo,kubernetes.io/metadata.name=netpol-demo
```

### 모니터링용 Pod 추가

```bash
kubectl run prometheus \
  --image=curlimages/curl:latest \
  --namespace=monitoring \
  --labels="app=prometheus" \
  --command -- sleep infinity
```

### Deny-All 기준선 설정

```bash
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: netpol-demo
spec:
  podSelector: {}
  policyTypes:
    - Ingress
EOF
```

---

## Step 1. NamespaceSelector로 특정 NS 허용

`env=prod` 라벨이 붙은 네임스페이스(= netpol-client)에서 오는 트래픽을 허용합니다.

```yaml title="allow-from-prod-ns.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-prod-ns
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:        # (1) env=prod 네임스페이스에서 오는 모든 트래픽
            matchLabels:
              env: prod
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl apply -f allow-from-prod-ns.yaml
```

### 결과 확인

```bash
# netpol-client (env=prod) → 허용 ✅
kubectl exec -n netpol-client client-b -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local | head -3

# netpol-demo 내 client-a (같은 NS) → 차단 🚫
kubectl exec -n netpol-demo client-a -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local

# monitoring (purpose=monitoring) → 차단 🚫
kubectl exec -n monitoring prometheus -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local
```

---

## Step 2. 모니터링 NS도 추가 허용 (OR 조건)

```yaml title="allow-prod-and-monitoring.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prod-and-monitoring
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
    - Ingress
  ingress:
    - from:                          # OR 조건 1: env=prod NS
        - namespaceSelector:
            matchLabels:
              env: prod
      ports:
        - port: 80
    - from:                          # OR 조건 2: purpose=monitoring NS
        - namespaceSelector:
            matchLabels:
              purpose: monitoring
      ports:
        - port: 80
```

```bash
kubectl delete networkpolicy allow-from-prod-ns -n netpol-demo
kubectl apply -f allow-prod-and-monitoring.yaml
```

```bash
# monitoring → 이제 허용 ✅
kubectl exec -n monitoring prometheus -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local | head -3

# env=prod → 여전히 허용 ✅
kubectl exec -n netpol-client client-b -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local | head -3
```

---

## Step 3. AND 조건 — 특정 NS의 특정 Pod만 허용

`env=prod` 네임스페이스이면서 동시에 `app=client` 라벨이 있는 Pod만 허용합니다.

```yaml title="allow-prod-client.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prod-client
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:          # (AND 조건)
            matchLabels:
              env: prod
          podSelector:                # 같은 `-` 항목 → AND
            matchLabels:
              app: client
      ports:
        - port: 80
```

!!! tip "YAML 들여쓰기 확인"
    `namespaceSelector`와 `podSelector`가 같은 리스트 항목(같은 `-` 아래)에 있어야 AND 조건입니다.
    각자 별도의 `-`로 시작하면 OR 조건이 됩니다.

```bash
kubectl delete networkpolicy allow-prod-and-monitoring -n netpol-demo
kubectl apply -f allow-prod-client.yaml
```

### 검증: netpol-client 내 Pod 라벨 확인

```bash
kubectl get pods -n netpol-client --show-labels
```

```
NAME       LABELS
client-b   app=client
```

`client-b`는 `env=prod` NS이고 `app=client` 라벨이 있으므로 → **허용** ✅

```bash
# client-b → 허용 ✅
kubectl exec -n netpol-client client-b -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local | head -3

# 다른 NS의 Pod (app=client 라벨 있어도, NS 라벨이 다르면 차단)
kubectl run client-c \
  --image=curlimages/curl:latest \
  --namespace=monitoring \
  --labels="app=client" \
  --command -- sleep infinity

kubectl exec -n monitoring client-c -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local
```

```
curl: (28) Connection timed out after 3001 milliseconds
```

`monitoring` 네임스페이스에 `app=client` 라벨의 Pod가 있어도, NS 라벨(`env=prod`)이 없으므로 차단됩니다. 🚫

---

## Step 4. kubernetes.io/metadata.name 활용 (특정 NS 이름으로 허용)

라벨을 추가하지 않고 네임스페이스 이름으로 직접 허용할 수도 있습니다.
K8s 1.21+에서는 모든 네임스페이스에 `kubernetes.io/metadata.name` 라벨이 자동으로 추가됩니다.

```yaml title="allow-by-ns-name.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-by-ns-name
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: netpol-client   # NS 이름으로 직접 지정
      ports:
        - port: 80
```

```bash
kubectl delete networkpolicy allow-prod-client -n netpol-demo
kubectl apply -f allow-by-ns-name.yaml

kubectl exec -n netpol-client client-b -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local | head -3
```

---

## Step 5. 정리

```bash
kubectl delete networkpolicy --all -n netpol-demo
kubectl delete pod client-c -n monitoring --ignore-not-found
```

---

## 정리 — AND vs OR 비교표

```yaml
# OR: 두 조건 중 하나라도 맞으면 허용
from:
  - namespaceSelector:       ← 별도 항목 (-)
      matchLabels: {env: prod}
  - podSelector:             ← 별도 항목 (-)
      matchLabels: {role: frontend}

# AND: 두 조건 모두 만족해야 허용
from:
  - namespaceSelector:       ← 같은 항목
      matchLabels: {env: prod}
    podSelector:             ← 같은 항목 (- 없음)
      matchLabels: {role: frontend}
```

| 패턴 | 의미 |
|------|------|
| `namespaceSelector` 단독 | 해당 NS의 **모든** Pod 허용 |
| `namespaceSelector` + `podSelector` (AND) | 해당 NS 안의 **특정 Pod**만 허용 |
| 여러 `from` 항목 (OR) | 조건 중 하나라도 맞으면 허용 |
| `kubernetes.io/metadata.name` | 라벨 없이 NS 이름으로 직접 지정 |

!!! info "다음 실습"
    다음에는 **Egress 규칙**과 **ipBlock**으로 외부 IP 범위를 제어하는 방법을 배웁니다.
