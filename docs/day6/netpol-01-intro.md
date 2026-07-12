# NetworkPolicy 01 — 개념과 기본 Deny-All

## 실습 목표

- NetworkPolicy가 왜 필요한지 이해한다.
- 기본(기본값) 통신 허용 상태를 직접 눈으로 확인한다.
- **Deny-All** 정책을 적용하고 트래픽이 막히는 것을 확인한다.

---

## 개념

### K8s 기본 네트워크 동작

NetworkPolicy를 아무것도 적용하지 않으면 **모든 Pod 간 통신이 허용**됩니다.

```
Pod A ──────────────────▶ Pod B   ✅ 허용 (기본값)
Pod A ──────────────────▶ Pod C   ✅ 허용 (기본값)
```

이 상태에서는 다른 네임스페이스의 Pod도 서로 통신할 수 있습니다.
실무에서는 이 상태가 **보안 위협**이 됩니다.

---

### NetworkPolicy란?

NetworkPolicy는 **Pod 단위로 인바운드(Ingress) / 아웃바운드(Egress) 트래픽을 제어**하는 오브젝트입니다.

```
           NetworkPolicy
                │
     ┌──────────┼──────────┐
     │          │          │
  podSelector  ingress   egress
  (어떤 Pod)  (들어오는) (나가는)
```

| 방향 | 의미 |
|------|------|
| **Ingress** | 선택된 Pod로 *들어오는* 트래픽 |
| **Egress** | 선택된 Pod에서 *나가는* 트래픽 |

!!! warning "CNI 지원 필수"
    NetworkPolicy는 **CNI(Container Network Interface)** 플러그인이 실제로 구현해야 동작합니다.
    Cilium, Calico, Weave Net 등이 지원합니다.
    기본 flannel은 NetworkPolicy 오브젝트를 생성해도 **무시**합니다.

---

### Deny-All 패턴

```yaml
spec:
  podSelector: {}   # 네임스페이스의 모든 Pod 선택
  policyTypes:
    - Ingress       # Ingress 규칙 목록이 없으면 → 전부 차단
```

`policyTypes`에 `Ingress`가 있는데 `ingress:` 항목이 없으면 → **모든 인바운드 차단**입니다.

---

## 실습 환경 준비

### 네임스페이스 생성

```bash
kubectl create namespace netpol-demo
kubectl create namespace netpol-client
```

### 테스트용 Pod 3개 배포

```bash
# 서버 역할 (netpol-demo)
kubectl run server \
  --image=nginx:alpine \
  --namespace=netpol-demo \
  --labels="app=server"

# 클라이언트 A (같은 네임스페이스)
kubectl run client-a \
  --image=curlimages/curl:latest \
  --namespace=netpol-demo \
  --labels="app=client" \
  --command -- sleep infinity

# 클라이언트 B (다른 네임스페이스)
kubectl run client-b \
  --image=curlimages/curl:latest \
  --namespace=netpol-client \
  --labels="app=client" \
  --command -- sleep infinity
```

### 서버 Service 생성

```bash
kubectl expose pod server \
  --port=80 \
  --name=server-svc \
  --namespace=netpol-demo
```

### Pod 상태 확인

```bash
kubectl get pods -n netpol-demo
kubectl get pods -n netpol-client
```

```
# 예상 출력
NAME       READY   STATUS    RESTARTS   AGE
client-a   1/1     Running   0          30s
server     1/1     Running   0          40s
```

---

## Step 1. 기본 상태 — 통신 허용 확인

NetworkPolicy가 없는 상태에서 통신을 확인합니다.

```bash
# 서버 ClusterIP 확인
kubectl get svc server-svc -n netpol-demo
```

```bash
# client-a → server 통신 테스트 (같은 네임스페이스)
kubectl exec -n netpol-demo client-a -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local

# client-b → server 통신 테스트 (다른 네임스페이스)
kubectl exec -n netpol-client client-b -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local
```

두 요청 모두 nginx 환영 페이지 HTML이 반환됩니다. ✅

---

## Step 2. Deny-All Ingress 정책 적용

```yaml title="deny-all-ingress.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: netpol-demo
spec:
  podSelector: {}        # (1) 빈 {} = 이 네임스페이스의 모든 Pod
  policyTypes:
    - Ingress            # (2) Ingress 유형 선언
                         # (3) ingress 규칙 없음 = 전부 차단
```

1. `podSelector: {}` — 네임스페이스 안의 **모든** Pod에 적용
2. `policyTypes: [Ingress]` — Ingress 방향을 이 정책으로 관리하겠다는 선언
3. `ingress:` 항목이 없으므로 → **아무것도 허용하지 않음**

```bash
kubectl apply -f deny-all-ingress.yaml
kubectl get networkpolicy -n netpol-demo
```

---

## Step 3. 차단 확인

```bash
# client-a → server (같은 네임스페이스) — 차단 예상
kubectl exec -n netpol-demo client-a -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local

# client-b → server (다른 네임스페이스) — 차단 예상
kubectl exec -n netpol-client client-b -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local
```

```
curl: (28) Connection timed out after 3001 milliseconds
```

3초 후 타임아웃이 발생합니다. 두 클라이언트 모두 차단됩니다. 🚫

---

## Step 4. Deny-All Egress 정책도 추가

인바운드뿐 아니라 아웃바운드도 막아봅니다.

```yaml title="deny-all-egress.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: netpol-demo
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP    # DNS는 허용 (이름 해석을 막으면 디버깅 불가)
```

!!! tip "DNS 포트(53)는 열어두자"
    Egress를 완전히 막으면 DNS 조회도 안 됩니다.
    `kube-dns`에 대한 UDP/TCP 53 포트는 최소한 허용하는 것이 일반적입니다.

```bash
kubectl apply -f deny-all-egress.yaml
```

```bash
# DNS는 됩니다 (53 허용)
kubectl exec -n netpol-demo client-a -- nslookup server-svc.netpol-demo.svc.cluster.local

# 하지만 HTTP는 안 됩니다
kubectl exec -n netpol-demo client-a -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local
```

---

## Step 5. 정리 — 정책 삭제

다음 실습을 위해 정책을 삭제합니다. (Pod와 Service는 유지)

```bash
kubectl delete networkpolicy deny-all-ingress -n netpol-demo
kubectl delete networkpolicy deny-all-egress  -n netpol-demo

# 삭제 확인
kubectl get networkpolicy -n netpol-demo
```

삭제 후 통신이 다시 허용되는지 확인합니다.

```bash
kubectl exec -n netpol-demo client-a -- \
  curl -s --connect-timeout 3 http://server-svc.netpol-demo.svc.cluster.local | head -5
```

---

## 핵심 요약

| 상태 | 동작 |
|------|------|
| NetworkPolicy 없음 | 모든 Pod 간 통신 허용 |
| `podSelector: {}` + Ingress, 규칙 없음 | 해당 NS의 모든 Pod로 들어오는 트래픽 차단 |
| `podSelector: {}` + Egress, 규칙 없음 | 해당 NS의 모든 Pod에서 나가는 트래픽 차단 |

!!! info "다음 실습"
    다음 실습에서는 **PodSelector**로 특정 Pod만 선택적으로 허용하는 방법을 배웁니다.
