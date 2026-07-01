# 3-1-1. Ingress TLS & Redirect 실습

## 실습 목표

- 자체 서명(Self-signed) 인증서로 HTTPS를 활성화한다.
- HTTP → HTTPS 자동 리다이렉트를 설정한다.
- Nginx Ingress Controller와 Cilium Ingress Controller 두 가지 케이스를 비교한다.

---

## 사전 준비 — 테스트용 인증서 생성

실습에서는 자체 서명(Self-signed) 인증서를 사용합니다.

### openssl이란?

**openssl**은 암호화·인증서·TLS 통신을 다루는 오픈소스 라이브러리이자 CLI 도구입니다.

주요 역할:

| 역할 | 설명 |
|------|------|
| 인증서 생성 | 개인키(`.key`)와 공개 인증서(`.crt`) 생성 |
| 인증서 서명 | CSR을 CA가 서명하거나 self-signed로 직접 서명 |
| 인증서 검증 | 인증서 유효기간·CN·발급자 확인 |
| TLS 테스트 | 서버와 TLS 핸드셰이크 테스트 |

HTTPS가 동작하려면 서버에 **개인키 + 인증서** 한 쌍이 필요합니다. openssl은 이 키 쌍을 만들어주는 도구입니다. 실제 운영에서는 Let's Encrypt 같은 공인 CA가 인증서를 발급하지만, 실습에서는 openssl로 직접 만들어 씁니다.

---

### openssl 준비

=== "macOS/Linux"
    openssl이 기본 내장되어 있습니다. 별도 설치 불필요합니다.

=== "Windows PowerShell"
    PowerShell에는 openssl이 없으므로 **Git Bash**를 사용합니다.

    **Git 설치 (없는 경우)**

    브라우저에서 `https://git-scm.com/download/win` 접속 → **64-bit Git for Windows Setup** 다운로드 → 설치 (기본 옵션으로 Next 계속)

    또는 winget:
    ```powershell
    winget install --id Git.Git
    ```

    설치 후 시작 메뉴에서 **Git Bash** 실행 — 이후 인증서 생성은 Git Bash에서 진행합니다.

### 인증서 생성

=== "macOS/Linux"
    ```bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout tls.key \
      -out tls.crt \
      -subj "/CN=lab.local/O=local"
    ```
=== "Windows (Git Bash)"
    Git Bash는 `/CN=` 앞의 `/`를 Windows 경로로 해석하므로 `//`와 `\`를 사용합니다.
    ```bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "//CN=lab.local\O=local"
    ```

??? info "명령어 옵션 설명"
    **`req -x509`**

    - `req` — 인증서 서명 요청(CSR)을 처리하는 명령어
    - `-x509` — CA에 보내지 않고 **자기 자신이 서명**한 인증서를 바로 생성 (Self-signed)

    실제 운영에서는 Let's Encrypt, DigiCert 같은 CA에 CSR을 보내서 서명을 받지만, 실습에서는 `-x509`로 직접 서명합니다.

    **나머지 옵션**

    | 옵션 | 의미 |
    |------|------|
    | `-nodes` | 개인키에 암호를 걸지 않음. kubectl이 자동으로 읽을 수 있어야 하므로 필수 |
    | `-days 365` | 인증서 유효기간 365일 |
    | `-newkey rsa:2048` | 2048비트 RSA 키를 새로 생성 |
    | `-keyout tls.key` | 생성된 개인키 저장 파일명 |
    | `-out tls.crt` | 생성된 인증서 저장 파일명 |

    **`-subj "/CN=lab.local/O=local"`**

    | 필드 | 풀네임 | 의미 |
    |------|--------|------|
    | `CN` | Common Name | 인증서가 적용될 **도메인 이름**. 브라우저가 접속 주소와 CN을 비교해서 일치 여부 확인 |
    | `O` | Organization | 조직명. 실습이라 `local`로 지정했지만 실제론 회사명 |

    `CN=lab.local`로 설정했기 때문에 `https://localhost`로 접속할 때만 유효합니다.
    실습에서 `curl -k` 옵션을 쓰는 이유가 바로 Self-signed 인증서의 CN 검증을 무시하기 위해서입니다.

TLS Secret으로 등록 (PowerShell로 돌아와서 실행):

=== "macOS/Linux"
    ```bash
    kubectl create secret tls my-tls-secret \
      --cert=tls.crt \
      --key=tls.key
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl create secret tls my-tls-secret `
      --cert=tls.crt `
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
        - lab.local
      secretName: my-tls-secret
  rules:
    - host: lab.local
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
    curl -k https://lab.local

    # HTTP → HTTPS 리다이렉트 확인
    curl -v http://lab.local 2>&1 | grep -E "< HTTP|Location"
    ```
=== "Windows PowerShell"
    ```powershell
    # HTTPS 접속 (자체 서명 인증서이므로 -k 옵션 사용)
    curl.exe -k https://lab.local

    # HTTP → HTTPS 리다이렉트 확인
    curl.exe -v http://lab.local 2>&1 | Select-String "< HTTP|Location"
    ```

예상 출력:

```
< HTTP/1.1 308 Permanent Redirect
< Location: https://lab.local/
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

## Case 2 — Cilium Ingress Controller (AKS 전용)

!!! warning "AKS 환경에서만 진행"
    Rancher Desktop의 기본 CNI는 Flannel이며 Cilium이 설치되어 있지 않습니다.
    이 섹션은 **3일차 AKS 클러스터**에 접속한 상태에서 진행하세요.

### Cilium Ingress Controller 활성화 확인

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium
kubectl get ingressclass
```

IngressClass 목록에 `cilium`이 있으면 사용 가능합니다. 없는 경우 아래 명령어로 활성화합니다.

```bash
az aks update \
  --resource-group k8s-4days-rg \
  --name k8s-4days-aks \
  --enable-cilium-dataplane
```

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
        - lab.local
      secretName: my-tls-secret
  rules:
    - host: lab.local
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
    curl -k https://lab.local

    # HTTP → HTTPS 리다이렉트 확인
    curl -v http://lab.local 2>&1 | grep -E "< HTTP|Location"
    ```
=== "Windows PowerShell"
    ```powershell
    # HTTPS 접속
    curl.exe -k https://lab.local

    # HTTP → HTTPS 리다이렉트 확인
    curl.exe -v http://lab.local 2>&1 | Select-String "< HTTP|Location"
    ```

예상 출력:

```
< HTTP/1.1 301 Moved Permanently
< Location: https://lab.local/
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
| 실습 환경 | Rancher Desktop | AKS |

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
