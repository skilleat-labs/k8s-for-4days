# NetworkPolicy 05 — Cilium NetworkPolicy (L7 제어)

## 실습 목표

- **CiliumNetworkPolicy(CNP)**로 HTTP 경로(path)와 메서드(method) 단위로 트래픽을 제어한다.
- 표준 K8s NetworkPolicy와 CiliumNetworkPolicy의 차이를 이해한다.
- 실습 환경에 Cilium이 설치되어 있는지 확인하고, 없으면 kind + Cilium으로 구성한다.

---

## Cilium vs 표준 NetworkPolicy

| 기능 | 표준 NetworkPolicy | CiliumNetworkPolicy |
|------|-------------------|---------------------|
| L3/L4 (IP, 포트) | ✅ | ✅ |
| L7 HTTP (경로, 메서드) | ❌ | ✅ |
| L7 gRPC | ❌ | ✅ |
| DNS 기반 필터링 | ❌ | ✅ (`toFQDNs`) |
| 클러스터 전체 범위 | ❌ | ✅ (`CiliumClusterwideNetworkPolicy`) |
| Identity 기반 (mTLS) | ❌ | ✅ |

---

## 실습 환경 확인

```bash
# Cilium이 설치되어 있는지 확인
kubectl get pods -n kube-system -l k8s-app=cilium

# Cilium CLI 설치 여부 확인
cilium version 2>/dev/null || echo "cilium CLI 미설치"
```

### Cilium이 없는 경우 — kind 클러스터 + Cilium 설치

!!! note "kind + Cilium 설치 (약 5분 소요)"
    기존 클러스터에 Cilium이 없다면 아래 방법으로 kind 클러스터를 새로 만들어 실습합니다.

```bash
# kind 설치 (macOS)
brew install kind

# CNI 없이 kind 클러스터 생성 (Cilium을 직접 설치하기 위해)
cat <<'EOF' > kind-cilium.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: true    # 기본 CNI 비활성화
  kubeProxyMode: none        # kube-proxy 비활성화 (Cilium이 대체)
EOF

kind create cluster --name cilium-lab --config kind-cilium.yaml

# kubectl context 전환
kubectl cluster-info --context kind-cilium-lab
```

```bash
# Cilium CLI 설치
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --remote-name-all \
  "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-darwin-amd64.tar.gz"
tar xzvf cilium-darwin-amd64.tar.gz
sudo mv cilium /usr/local/bin/

# Cilium 설치 (Helm 방식)
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
  --version 1.16.0 \
  --namespace kube-system \
  --set image.pullPolicy=IfNotPresent \
  --set ipam.mode=kubernetes

# 설치 완료 대기
cilium status --wait
```

### 설치 확인

```bash
cilium status
kubectl get pods -n kube-system -l k8s-app=cilium
```

```
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/        ClusterMesh:        disabled
```

---

## 실습 환경 구성

```bash
# 네임스페이스 생성
kubectl create namespace l7-demo

# 백엔드 API 서버 (간단한 HTTP 서버)
kubectl apply -n l7-demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
      volumes:
        - name: nginx-config
          configMap:
            name: api-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  default.conf: |
    server {
      listen 80;
      location /api/public {
        return 200 '{"status":"ok","path":"/api/public"}';
        add_header Content-Type application/json;
      }
      location /api/admin {
        return 200 '{"status":"ok","path":"/api/admin"}';
        add_header Content-Type application/json;
      }
      location /health {
        return 200 '{"status":"healthy"}';
        add_header Content-Type application/json;
      }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
spec:
  selector:
    app: api-server
  ports:
    - port: 80
EOF

# 클라이언트 Pod (두 종류)
kubectl run frontend \
  --image=curlimages/curl:latest \
  --namespace=l7-demo \
  --labels="app=frontend,role=user" \
  --command -- sleep infinity

kubectl run admin \
  --image=curlimages/curl:latest \
  --namespace=l7-demo \
  --labels="app=admin,role=admin" \
  --command -- sleep infinity

kubectl get pods -n l7-demo
```

---

## Step 1. L4만 제어하는 기본 NetworkPolicy 한계 확인

표준 NetworkPolicy로는 HTTP 경로 구분이 불가능합니다.

```bash
# 모든 경로 접근 가능한 현재 상태 확인
kubectl exec -n l7-demo frontend -- curl -s http://api-svc/api/public
kubectl exec -n l7-demo frontend -- curl -s http://api-svc/api/admin    # admin도 접근 가능!
kubectl exec -n l7-demo frontend -- curl -s http://api-svc/health
```

---

## Step 2. CiliumNetworkPolicy — HTTP 경로 제어

`frontend` Pod는 `/api/public`과 `/health`만 접근하고, `/api/admin`은 차단합니다.

```yaml title="l7-http-policy.yaml"
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: l7-http-policy
  namespace: l7-demo
spec:
  endpointSelector:            # (1) 이 정책이 적용되는 Pod
    matchLabels:
      app: api-server
  ingress:
    - fromEndpoints:           # (2) frontend Pod에서 오는 요청
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:              # (3) HTTP L7 규칙
              - method: GET
                path: /api/public
              - method: GET
                path: /health
    - fromEndpoints:           # (4) admin Pod는 모든 경로 허용
        - matchLabels:
            app: admin
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
```

```bash
kubectl apply -f l7-http-policy.yaml
kubectl get cnp -n l7-demo    # CiliumNetworkPolicy 목록
```

---

## Step 3. L7 정책 검증

```bash
# frontend: /api/public → 허용 ✅
kubectl exec -n l7-demo frontend -- \
  curl -s -o /dev/null -w "%{http_code}" http://api-svc/api/public

# frontend: /health → 허용 ✅
kubectl exec -n l7-demo frontend -- \
  curl -s -o /dev/null -w "%{http_code}" http://api-svc/health

# frontend: /api/admin → 차단 🚫 (403 Access Denied)
kubectl exec -n l7-demo frontend -- \
  curl -s -o /dev/null -w "%{http_code}" http://api-svc/api/admin

# admin: /api/admin → 허용 ✅
kubectl exec -n l7-demo admin -- \
  curl -s -o /dev/null -w "%{http_code}" http://api-svc/api/admin
```

```
frontend → /api/public : 200
frontend → /health     : 200
frontend → /api/admin  : 403
admin    → /api/admin  : 200
```

!!! success "L7 필터링 성공"
    Cilium이 HTTP 레이어에서 경로별로 트래픽을 차단합니다.
    표준 NetworkPolicy로는 불가능한 기능입니다.

---

## Step 4. toFQDNs — 도메인 이름으로 Egress 제어

Cilium은 IP가 아닌 도메인 이름(FQDN)으로 외부 트래픽을 제어할 수 있습니다.

```yaml title="egress-fqdn.yaml"
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: egress-to-specific-domain
  namespace: l7-demo
spec:
  endpointSelector:
    matchLabels:
      app: frontend
  egress:
    - toFQDNs:                   # FQDN 기반 허용
        - matchName: "httpbin.org"
        - matchPattern: "*.example.com"   # 와일드카드 지원
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
    - toEndpoints:               # 내부 DNS 허용
        - matchLabels:
            "k8s:io.kubernetes.serviceaccount.name": "coredns"
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
```

```bash
kubectl apply -f egress-fqdn.yaml
```

---

## Step 5. Hubble로 트래픽 가시성 확인 (Cilium 관찰)

Cilium에는 **Hubble**이라는 네트워크 관찰 도구가 내장되어 있습니다.

```bash
# Hubble 활성화
cilium hubble enable

# Hubble CLI 포트포워딩
cilium hubble port-forward &

# 실시간 트래픽 관찰
hubble observe --namespace l7-demo --follow

# 다른 터미널에서 트래픽 발생
kubectl exec -n l7-demo frontend -- curl -s http://api-svc/api/public
kubectl exec -n l7-demo frontend -- curl -s http://api-svc/api/admin
```

```
TIMESTAMP   SOURCE                      DESTINATION               TYPE      VERDICT   SUMMARY
...         l7-demo/frontend            l7-demo/api-server:80     L7        FORWARDED GET /api/public
...         l7-demo/frontend            l7-demo/api-server:80     L7        DROPPED   GET /api/admin
```

---

## Step 6. 정리

```bash
kubectl delete namespace l7-demo
```

---

## 핵심 요약

| CiliumNetworkPolicy 필드 | 표준 NetworkPolicy 대응 | 추가 기능 |
|--------------------------|------------------------|-----------|
| `endpointSelector` | `podSelector` | 동일 |
| `fromEndpoints` | `from.podSelector` | 동일 |
| `fromEntities` | - | `cluster`, `world`, `host` 등 특수 대상 |
| `toPorts.rules.http` | ❌ | **L7 HTTP 경로/메서드 필터링** |
| `toFQDNs` | ❌ | **도메인 이름으로 Egress 제어** |
| `fromRequires` | ❌ | **AND 조건 (추가 제약)** |

!!! info "다음 실습"
    다음에는 **Cilium Mutual TLS(mTLS)**로 Pod 간 통신을 암호화하고 인증하는 방법을 배웁니다.
