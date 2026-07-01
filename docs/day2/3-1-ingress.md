# 3-1. Ingress 실습

## 실습 목표

- Nginx Ingress Controller를 설치하고 경로·호스트 기반 라우팅을 구현한다.
- 혼자서 요구사항을 읽고 Ingress 리소스를 작성한다.

---

## 사전 준비 — Rancher Desktop 사용자

!!! warning "Traefik 비활성화 필수"
    Rancher Desktop(k3s)은 기본적으로 **Traefik**이라는 Ingress Controller가 포트 80을 이미 점유하고 있습니다.
    이 상태에서 Nginx Ingress Controller를 설치하면 포트 충돌로 `EXTERNAL-IP`가 `<pending>`에 머물고 라우팅이 동작하지 않습니다.
    **반드시 아래 순서대로 Traefik을 먼저 비활성화한 뒤 실습을 시작하세요.**

**비활성화 방법 (GUI)**

1. Rancher Desktop 앱 실행
2. 왼쪽 메뉴 → **Kubernetes Settings** (또는 **Preferences → Kubernetes**)
3. **Enable Traefik** 체크박스 **해제**
4. **Apply** 클릭 → k3s 자동 재시작 (1~2분 소요)

**비활성화 확인**

=== "macOS/Linux"
    ```bash
    kubectl get pods -n kube-system | grep traefik
    # 아무것도 출력되지 않으면 정상
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl get pods -n kube-system | Select-String "traefik"
    # 아무것도 출력되지 않으면 정상
    ```

---

### 1) Nginx Ingress Controller 설치

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```

설치 확인 (1~2분 소요):

```bash
kubectl get pods -n ingress-nginx -w
```

`ingress-nginx-controller-*` Pod가 `Running`이 되면 `Ctrl+C` 종료.

```bash
kubectl get svc -n ingress-nginx
```

```
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)
ingress-nginx-controller   LoadBalancer   10.96.10.100   localhost     80:31234/TCP,443:31235/TCP
```

> Docker Desktop / Rancher Desktop에서 `EXTERNAL-IP`가 `localhost`로 표시됩니다.

---

### 2) 테스트용 앱 2개 배포

배포할 리소스 구조는 다음과 같습니다.

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌──────────────────────────┐  ┌──────────────────────────┐ │
│  │       api-app            │  │       web-app            │ │
│  │                          │  │                          │ │
│  │  ┌────────────────────┐  │  │  ┌────────────────────┐  │ │
│  │  │  Service: api-svc  │  │  │  │  Service: web-svc  │  │ │
│  │  │  ClusterIP :80     │  │  │  │  ClusterIP :80     │  │ │
│  │  └────────┬───────────┘  │  │  └────────┬───────────┘  │ │
│  │           │ targetPort   │  │           │ targetPort   │ │
│  │           │ 5678         │  │           │ 5678         │ │
│  │  ┌────────▼───────────┐  │  │  ┌────────▼───────────┐  │ │
│  │  │  Pod: api-app      │  │  │  │  Pod: web-app      │  │ │
│  │  │  http-echo :5678   │  │  │  │  http-echo :5678   │  │ │
│  │  │  "Hello from API"  │  │  │  │  "Hello from Web"  │  │ │
│  │  └────────────────────┘  │  │  └────────────────────┘  │ │
│  └──────────────────────────┘  └──────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

위 다이어그램을 보고 `api-app.yaml`, `web-app.yaml` 두 파일을 직접 작성해보세요.

---

**`args`란?**

`args`는 컨테이너 실행 시 이미지의 기본 커맨드에 전달하는 **명령줄 인수**입니다. 도커에서 `CMD`에 해당합니다.

```yaml
containers:
  - name: app
    image: some-image
    args:
      - "--flag=value"
      - "--another=value"
```

`hashicorp/http-echo`는 `-text` 옵션으로 응답할 텍스트를 지정하지 않으면 아무것도 반환하지 않습니다. **이 이미지는 반드시 `args`를 써야 정상 동작합니다.**

---

**힌트**

| 항목 | api-app | web-app |
|------|---------|---------|
| 이미지 | `hashicorp/http-echo:latest` | `hashicorp/http-echo:latest` |
| `args` `-text` | `Hello from API service` | `Hello from Web service` |
| `ports.containerPort` | `5678` | `5678` |

> `http-echo`의 기본 리슨 포트는 `:5678`입니다. `-listen` 옵션은 생략 가능합니다.

작성이 끝나면 아래 명령으로 배포하고 확인합니다.

```bash
kubectl apply -f api-app.yaml
kubectl apply -f web-app.yaml
kubectl get pods
kubectl get svc
```

---

### 3) Ingress 리소스 생성 — 경로 기반 라우팅

`ingress-path.yaml` 파일을 만듭니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: lab.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-path.yaml
kubectl get ingress
kubectl describe ingress lab-ingress
```

`describe` 출력에서 확인:

- `Rules`: host → path → backend Service 매핑
- `Default backend`: 규칙에 없는 경로의 처리

### 4) /etc/hosts 설정

#### hosts 파일이란?

브라우저가 `lab.local` 같은 도메인에 접속하려면 먼저 DNS 서버에 IP 주소를 물어봅니다. **hosts 파일**은 DNS 서버에 묻기 전에 운영체제가 가장 먼저 확인하는 로컬 주소록입니다.

```
도메인 요청 → hosts 파일 확인 → (없으면) DNS 서버 조회
```

실습에서 `lab.local`은 실제 존재하는 도메인이 아니므로, hosts 파일에 직접 `127.0.0.1 lab.local`을 등록해서 로컬 클러스터로 연결되도록 설정합니다.

**파일 위치**

| OS | 경로 |
|----|------|
| macOS / Linux | `/etc/hosts` |
| Windows | `C:\Windows\System32\drivers\etc\hosts` |

#### Windows에서 수정하는 방법

!!! warning "관리자 권한 필요"
    hosts 파일은 시스템 파일이므로 반드시 **관리자 권한**으로 실행해야 합니다.

**방법 1 — PowerShell (관리자 권한)**

시작 메뉴에서 `PowerShell` 검색 → 우클릭 → **관리자 권한으로 실행** 후 아래 명령어 실행:

```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1 lab.local"
```

등록 확인:

```powershell
Get-Content "C:\Windows\System32\drivers\etc\hosts" | Select-String "lab.local"
```

**방법 2 — 메모장으로 직접 편집**

1. 시작 메뉴에서 `메모장` 검색 → 우클릭 → **관리자 권한으로 실행**
2. 메모장에서 `파일 → 열기` → 경로에 `C:\Windows\System32\drivers\etc\hosts` 입력
3. 파일 형식을 **모든 파일 (\*.\*)** 로 변경 후 `hosts` 파일 선택
4. 맨 아래에 추가:
    ```
    127.0.0.1 lab.local
    ```
5. 저장 (`Ctrl+S`)

---

=== "macOS / Linux"

    ```bash
    sudo sh -c 'echo "127.0.0.1 lab.local" >> /etc/hosts'
    ```

=== "Windows (PowerShell — 관리자 권한)"

    ```powershell
    Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1 lab.local"
    ```

### 5) 접속 확인

=== "macOS / Linux"

    ```bash
    curl http://lab.local/api
    # Hello from API service

    curl http://lab.local/web
    # Hello from Web service
    ```

=== "Windows (PowerShell)"

    ```powershell
    curl http://lab.local/api
    # 또는
    Invoke-WebRequest -Uri http://lab.local/api -UseBasicParsing | Select-Object -ExpandProperty Content
    ```

    > PowerShell의 `curl`은 `Invoke-WebRequest`의 별칭입니다.
    > `-UseBasicParsing`을 붙이면 본문만 깔끔하게 출력됩니다.

### 6) Ingress Controller 로그 확인

=== "macOS / Linux"

    ```bash
    kubectl logs -n ingress-nginx deployment/ingress-nginx-controller | tail -20
    ```

=== "Windows (PowerShell)"

    ```powershell
    kubectl logs -n ingress-nginx deployment/ingress-nginx-controller | Select-Object -Last 20
    ```

접속할 때마다 어떤 경로가 어느 Service로 라우팅됐는지 로그로 확인합니다.

---

### 7) 혼자 해보기 🧩

지금까지 가이드를 따라 했다면, 이제 스스로 만들어봅니다.
**아래 요구사항을 읽고 직접 YAML을 작성한 뒤 동작을 확인하세요.**

---

#### 미션 1 — 서비스 추가 + 경로 라우팅

새 앱을 하나 더 추가하고 `/shop` 경로로 연결하세요.

**요구사항:**

- Deployment 이름: `shop-app`, 이미지: `hashicorp/http-echo:latest`
- 응답 메시지: `Hello from Shop service`
- Service 이름: `shop-svc`, port: 80 → targetPort: 5678
- Ingress 규칙: `lab.local/shop` → `shop-svc`

**성공 조건:**

=== "macOS/Linux"
    ```bash
    curl http://lab.local/shop
    # Hello from Shop service
    ```
=== "Windows PowerShell"
    ```powershell
    curl.exe http://lab.local/shop
    # Hello from Shop service
    ```

---

#### 미션 2 — 호스트 기반 라우팅

경로가 아닌 **도메인 이름**으로 라우팅하는 Ingress를 만드세요.

**요구사항:**

- `api.lab.local` → `api-svc`
- `web.lab.local` → `web-svc`
- `/etc/hosts`에 두 도메인 모두 등록
- `/api` 경로 없이 루트(`/`)로 접속해도 응답

**`/etc/hosts` 등록** (테스트 전에 먼저 추가하세요)

=== "macOS / Linux"

    ```bash
    sudo sh -c 'echo "127.0.0.1 api.lab.local" >> /etc/hosts'
    sudo sh -c 'echo "127.0.0.1 web.lab.local" >> /etc/hosts'
    ```

=== "Windows (PowerShell — 관리자 권한)"

    ```powershell
    Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1 api.lab.local"
    Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1 web.lab.local"
    ```

**성공 조건:**

=== "macOS/Linux"
    ```bash
    curl http://api.lab.local
    # Hello from API service

    curl http://web.lab.local
    # Hello from Web service
    ```
=== "Windows PowerShell"
    ```powershell
    curl.exe http://api.lab.local
    # Hello from API service

    curl.exe http://web.lab.local
    # Hello from Web service
    ```

> Kubernetes 공식 문서에서 호스트 기반 라우팅 방법을 찾아 직접 구현해보세요.
> 🔗 [Ingress — kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/ingress/#name-based-virtual-hosting)

---

### 정리 (Ingress 파트)

```bash
kubectl delete -f ingress-path.yaml
kubectl delete -f web-app.yaml
kubectl delete -f api-app.yaml
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```

> Nginx Ingress Controller를 삭제하지 않으면 다음 실습(Gateway API)에서 포트 80 충돌로 Gateway가 정상 기동되지 않습니다.

**`/etc/hosts` 원복**

실습 중 추가한 도메인을 삭제합니다.

=== "macOS / Linux"

    ```bash
    sudo sed -i '' '/lab\.local/d' /etc/hosts

    # 확인
    cat /etc/hosts | grep lab.local
    ```

=== "Windows (PowerShell — 관리자 권한)"

    ```powershell
    $hosts = "C:\Windows\System32\drivers\etc\hosts"
    (Get-Content $hosts) | Where-Object { $_ -notmatch "lab\.local" } | Set-Content $hosts

    # 확인
    Get-Content $hosts | Select-String "lab.local"
    ```

---

## 트러블슈팅

| 증상 | 확인 사항 |
|---|---|
| `curl: (6) Could not resolve host` | `/etc/hosts`에 도메인 등록 확인 |
| Ingress `404 Not Found` | `ingressClassName: nginx` 설정 확인 |

---

## 정답

??? note "정답 보기 (먼저 직접 작성해보세요!)"

    **api-app.yaml**

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: api-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: api-app
      template:
        metadata:
          labels:
            app: api-app
        spec:
          containers:
            - name: api-app
              image: hashicorp/http-echo:latest
              args:
                - "-text=Hello from API service"
              ports:
                - containerPort: 5678
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: api-svc
    spec:
      selector:
        app: api-app
      ports:
        - port: 80
          targetPort: 5678
    ```

    **web-app.yaml**

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: web-app
      template:
        metadata:
          labels:
            app: web-app
        spec:
          containers:
            - name: web-app
              image: hashicorp/http-echo:latest
              args:
                - "-text=Hello from Web service"
              ports:
                - containerPort: 5678
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: web-svc
    spec:
      selector:
        app: web-app
      ports:
        - port: 80
          targetPort: 5678
    ```

??? note "미션 1 정답 — 서비스 추가 + 경로 라우팅"

    **shop-app.yaml**

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: shop-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: shop-app
      template:
        metadata:
          labels:
            app: shop-app
        spec:
          containers:
            - name: shop-app
              image: hashicorp/http-echo:latest
              args:
                - "-text=Hello from Shop service"
              ports:
                - containerPort: 5678
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: shop-svc
    spec:
      selector:
        app: shop-app
      ports:
        - port: 80
          targetPort: 5678
    ```

    **ingress-path.yaml** (`/shop` 경로 추가)

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ingress-path
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
      ingressClassName: nginx
      rules:
        - host: lab.local
          http:
            paths:
              - path: /api
                pathType: Prefix
                backend:
                  service:
                    name: api-svc
                    port:
                      number: 80
              - path: /web
                pathType: Prefix
                backend:
                  service:
                    name: web-svc
                    port:
                      number: 80
              - path: /shop
                pathType: Prefix
                backend:
                  service:
                    name: shop-svc
                    port:
                      number: 80
    ```

??? note "미션 2 정답 — 호스트 기반 라우팅"

    **ingress-host.yaml**

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ingress-host
    spec:
      ingressClassName: nginx
      rules:
        - host: api.lab.local
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: api-svc
                    port:
                      number: 80
        - host: web.lab.local
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: web-svc
                    port:
                      number: 80
    ```
