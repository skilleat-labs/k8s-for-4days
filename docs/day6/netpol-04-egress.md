# NetworkPolicy 04 — Egress & ipBlock 실습

## 실습 목표

- **Egress** 정책으로 Pod에서 나가는 트래픽을 제어한다.
- **ipBlock**으로 외부 IP 대역(CIDR)을 허용/차단한다.
- Ingress + Egress를 함께 사용하는 실무 패턴을 이해한다.

---

## 개념

### Egress 정책

Egress는 Pod에서 **나가는** 트래픽을 제어합니다.

```
외부 DB (10.0.0.5) ──────────────▶ ❌ (차단)
                                        │
Pod(frontend) ──▶ kube-dns:53    ──▶ ✅
              ──▶ backend:8080   ──▶ ✅
              ──▶ 외부 API       ──▶ ❌ (차단)
```

### ipBlock

`ipBlock`은 **IP CIDR 범위**를 기준으로 허용/차단합니다.
Pod 라벨이 없는 외부 서비스(데이터베이스, 외부 API)를 제어할 때 사용합니다.

```yaml
ipBlock:
  cidr: 10.0.0.0/24     # 이 대역은 허용
  except:
    - 10.0.0.5/32        # 그 중 이 IP는 제외(차단)
```

---

## 실습 환경 준비

```bash
# 네임스페이스 확인
kubectl get namespaces netpol-demo netpol-client

# frontend Pod 생성 (Egress 정책 대상)
kubectl run frontend \
  --image=curlimages/curl:latest \
  --namespace=netpol-demo \
  --labels="app=frontend" \
  --command -- sleep infinity

# backend Pod 생성 (Egress 허용 대상)
kubectl run backend \
  --image=nginx:alpine \
  --namespace=netpol-demo \
  --labels="app=backend"

kubectl expose pod backend \
  --port=8080 \
  --target-port=80 \
  --name=backend-svc \
  --namespace=netpol-demo

# 상태 확인
kubectl get pods,svc -n netpol-demo
```

---

## Step 1. 기본 상태 확인 (Egress 제한 없음)

```bash
# frontend → backend 통신 (내부)
kubectl exec -n netpol-demo frontend -- \
  curl -s --connect-timeout 3 http://backend-svc.netpol-demo.svc.cluster.local:8080 | head -3

# frontend → 외부 인터넷 (허용 상태)
kubectl exec -n netpol-demo frontend -- \
  curl -s --connect-timeout 5 http://httpbin.org/ip
```

---

## Step 2. Egress Deny-All 적용

```yaml title="egress-deny-all.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-deny-all
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: frontend     # frontend Pod에만 적용
  policyTypes:
    - Egress            # Egress 방향만 제어
```

```bash
kubectl apply -f egress-deny-all.yaml

# DNS도 안 됩니다
kubectl exec -n netpol-demo frontend -- \
  nslookup backend-svc.netpol-demo.svc.cluster.local

# HTTP도 안 됩니다
kubectl exec -n netpol-demo frontend -- \
  curl -s --connect-timeout 3 http://backend-svc.netpol-demo.svc.cluster.local:8080
```

---

## Step 3. DNS + backend만 Egress 허용

```yaml title="frontend-egress.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - ports:                     # (1) DNS 허용
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    - to:                        # (2) backend Pod로만 허용
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 8080
```

```bash
kubectl delete networkpolicy egress-deny-all -n netpol-demo
kubectl apply -f frontend-egress.yaml

# DNS 이름 해석 ✅
kubectl exec -n netpol-demo frontend -- \
  nslookup backend-svc.netpol-demo.svc.cluster.local

# backend 통신 ✅
kubectl exec -n netpol-demo frontend -- \
  curl -s --connect-timeout 3 http://backend-svc.netpol-demo.svc.cluster.local:8080 | head -3

# 외부 인터넷 차단 🚫
kubectl exec -n netpol-demo frontend -- \
  curl -s --connect-timeout 5 http://httpbin.org/ip
```

---

## Step 4. ipBlock으로 외부 IP 대역 허용

특정 외부 CIDR 대역(예: 회사 내부망)만 허용하는 패턴입니다.

```bash
# 현재 클러스터의 노드 IP 확인 (테스트용)
kubectl get nodes -o wide
```

```yaml title="egress-with-ipblock.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-with-ipblock
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - ports:                      # DNS 항상 허용
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    - to:
        - podSelector:            # 내부 backend Pod
            matchLabels:
              app: backend
      ports:
        - port: 8080
    - to:
        - ipBlock:                # 외부 특정 IP 대역만 허용
            cidr: 0.0.0.0/0      # 전체 허용 (단, except 목록 제외)
            except:
              - 10.96.0.0/12     # kube 내부 CIDR (이미 위에서 처리)
              - 192.168.0.0/16   # 내부망
      ports:
        - port: 443              # HTTPS만 허용
        - port: 80               # HTTP 허용
```

```bash
kubectl delete networkpolicy frontend-egress -n netpol-demo
kubectl apply -f egress-with-ipblock.yaml
```

!!! tip "ipBlock 실무 활용"
    - 온프레미스 DB 서버 IP를 명시적으로 허용
    - 외부 API 게이트웨이 IP만 허용
    - 내부 메타데이터 서버(`169.254.169.254`) 접근 차단 (클라우드 보안)

---

## Step 5. Ingress + Egress 동시 적용 (실무 패턴)

실무에서는 Ingress와 Egress를 하나의 NetworkPolicy에 함께 정의하거나
별도 파일로 관리합니다.

```yaml title="frontend-full-policy.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-full-policy
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:        # prod 네임스페이스에서만 인바운드 허용
            matchLabels:
              env: prod
      ports:
        - port: 3000               # frontend 앱 포트
  egress:
    - ports:                       # DNS 항상 허용
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    - to:
        - podSelector:             # backend만 허용
            matchLabels:
              app: backend
      ports:
        - port: 8080
```

```bash
kubectl delete networkpolicy egress-with-ipblock -n netpol-demo
kubectl apply -f frontend-full-policy.yaml

kubectl describe networkpolicy frontend-full-policy -n netpol-demo
```

---

## Step 6. 정책 목록 확인 및 진단

```bash
# 네임스페이스의 모든 NetworkPolicy 확인
kubectl get networkpolicy -n netpol-demo

# 특정 정책 상세 확인
kubectl describe networkpolicy frontend-full-policy -n netpol-demo
```

```
Name:         frontend-full-policy
Namespace:    netpol-demo
Created on:   ...
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app=frontend
  Allowing ingress traffic:
    To Port: 3000/TCP
    From:
      NamespaceSelector: env=prod
  Allowing egress traffic:
    To Port: 53/UDP
    To Port: 53/TCP
    To Port: 8080/TCP
    To:
      PodSelector: app=backend
  Policy Types: Ingress, Egress
```

---

## Step 7. 정리

```bash
kubectl delete networkpolicy --all -n netpol-demo
kubectl delete pod frontend backend -n netpol-demo --ignore-not-found
kubectl delete svc backend-svc -n netpol-demo --ignore-not-found
```

---

## 핵심 요약

| 항목 | 설명 |
|------|------|
| `policyTypes: [Egress]` | Egress 방향을 이 정책으로 관리 |
| `egress.to` | 트래픽을 보낼 수 있는 목적지 |
| `egress.ports` | 허용할 포트 (to 없이 ports만 → 모든 목적지의 해당 포트) |
| `ipBlock.cidr` | 허용할 IP CIDR 범위 |
| `ipBlock.except` | cidr 내에서 제외할 IP 범위 |
| DNS 53 허용 | Egress deny-all 시 반드시 열어야 이름 해석 가능 |

!!! info "다음 실습"
    다음에는 **Cilium NetworkPolicy**로 L7(HTTP 경로, gRPC 메서드)까지 제어하는 방법을 배웁니다.
