# 3-1-1. Ingress TLS & Redirect 실습

## 실습 목표

- 자체 서명(Self-signed) 인증서로 HTTPS를 활성화한다.
- HTTP → HTTPS 자동 리다이렉트를 설정한다.
- Nginx Ingress Controller와 Cilium Ingress Controller 두 가지 케이스를 비교한다.

---

## 사전 준비 — 테스트용 인증서 생성

실습에서는 자체 서명(Self-signed) 인증서를 사용합니다.

```bash
# 개인키 + 인증서 생성 (localhost 용)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=localhost/O=local"
```

TLS Secret으로 등록:

```bash
kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

확인:

```bash
kubectl get secret my-tls-secret
kubectl describe secret my-tls-secret
```

---

## 테스트 앱 배포

```yaml
# echo-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
        - name: echo
          image: hashicorp/http-echo:0.2.3
          args:
            - "-text=Hello from TLS!"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: echo-svc
spec:
  selector:
    app: echo
  ports:
    - port: 80
      targetPort: 5678
```

```bash
kubectl apply -f echo-app.yaml
kubectl get pods,svc
```

---

## Case 1 — Nginx Ingress Controller

### Ingress Controller 설치 확인

```bash
kubectl get pods -n ingress-nginx
```

설치되어 있지 않으면:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### TLS Ingress 생성

`ingress-nginx-tls.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress-nginx
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - localhost
      secretName: my-tls-secret
  rules:
    - host: localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echo-svc
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-nginx-tls.yaml
kubectl get ingress echo-ingress-nginx
```

### 동작 확인

=== "macOS/Linux"
    ```bash
    # HTTPS 접속 (자체 서명 인증서이므로 -k 옵션 사용)
    curl -k https://localhost

    # HTTP → HTTPS 리다이렉트 확인
    curl -v http://localhost 2>&1 | grep -E "< HTTP|Location"
    ```
=== "Windows PowerShell"
    ```powershell
    # HTTPS 접속 (자체 서명 인증서이므로 -k 옵션 사용)
    curl.exe -k https://localhost

    # HTTP → HTTPS 리다이렉트 확인
    curl.exe -v http://localhost 2>&1 | Select-String "< HTTP|Location"
    ```

예상 출력:

```
< HTTP/1.1 308 Permanent Redirect
< Location: https://localhost/
```

!!! info "308 vs 301"
    Nginx Ingress는 기본적으로 `308 Permanent Redirect`를 사용합니다.
    `301`로 변경하려면 annotation에 `nginx.ingress.kubernetes.io/use-regex: "true"`를 추가합니다.

### Redirect 경로 별도 제어

HTTP는 허용하고 HTTPS도 받으려면 `ssl-redirect`를 끄면 됩니다:

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "false"
```

특정 경로만 리다이렉트하려면 `force-ssl-redirect`를 사용합니다:

```yaml
annotations:
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

---

## Case 2 — Cilium Ingress Controller

### Ingress Controller 활성화 확인

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium
kubectl get ingressclass
```

Cilium이 설치된 경우 IngressClass `cilium`이 존재합니다.
Rancher Desktop에서는 기본적으로 Cilium을 사용합니다.

### TLS Ingress 생성

`ingress-cilium-tls.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress-cilium
  annotations:
    ingress.cilium.io/force-https: "enabled"
spec:
  ingressClassName: cilium
  tls:
    - hosts:
        - localhost
      secretName: my-tls-secret
  rules:
    - host: localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echo-svc
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-cilium-tls.yaml
kubectl get ingress echo-ingress-cilium
```

### 동작 확인

=== "macOS/Linux"
    ```bash
    # HTTPS 접속
    curl -k https://localhost

    # HTTP → HTTPS 리다이렉트 확인
    curl -v http://localhost 2>&1 | grep -E "< HTTP|Location"
    ```
=== "Windows PowerShell"
    ```powershell
    # HTTPS 접속
    curl.exe -k https://localhost

    # HTTP → HTTPS 리다이렉트 확인
    curl.exe -v http://localhost 2>&1 | Select-String "< HTTP|Location"
    ```

예상 출력:

```
< HTTP/1.1 301 Moved Permanently
< Location: https://localhost/
```

!!! info "Cilium의 force-https"
    `ingress.cilium.io/force-https: "enabled"` annotation을 설정하면
    HTTP 요청이 들어올 때 자동으로 `301 Redirect`를 반환합니다.

---

## Nginx vs Cilium 비교

| 항목 | Nginx Ingress | Cilium Ingress |
|------|--------------|----------------|
| TLS 설정 | `spec.tls` + `secretName` | `spec.tls` + `secretName` |
| HTTP→HTTPS 리다이렉트 | `nginx.ingress.kubernetes.io/ssl-redirect: "true"` | `ingress.cilium.io/force-https: "enabled"` |
| 기본 리다이렉트 코드 | `308` | `301` |
| IngressClass 이름 | `nginx` | `cilium` |
| Rancher Desktop 기본 제공 | X | O |

---

## 정리 (리소스 삭제)

```bash
kubectl delete -f ingress-nginx-tls.yaml
kubectl delete -f ingress-cilium-tls.yaml
kubectl delete -f echo-app.yaml
kubectl delete secret my-tls-secret
rm tls.crt tls.key
```

---

## 트러블슈팅

| 증상 | 확인 사항 |
|------|---------|
| `curl: (35) SSL connect error` | TLS Secret이 올바르게 생성됐는지 확인 (`kubectl describe secret`) |
| 리다이렉트가 안됨 | annotation 이름 오타 여부 확인, IngressClass가 올바른지 확인 |
| `404 Not Found` | Ingress의 `host`와 요청 호스트 일치 여부 확인 |
| IngressClass 없음 | `kubectl get ingressclass`로 사용 가능한 클래스 확인 |
