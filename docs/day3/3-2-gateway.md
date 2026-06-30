# Gateway API 실습

## 실습 환경 — AKS 클러스터 연결

### 0) Azure CLI 설치 (PowerShell)

```powershell
winget install --exact --id Microsoft.AzureCLI
```

설치 후 **PowerShell을 재시작**하고 버전을 확인합니다.

```powershell
az version
```

로그인:

```powershell
az login
```

브라우저가 열리면 Azure 계정으로 로그인합니다. 로그인 완료 후 터미널에 구독 목록이 출력되면 정상입니다.

---

### 1) AKS 클러스터 생성 (강사와 함께)

강사가 아래 명령어로 AKS 클러스터를 생성합니다. 수강생은 화면을 보며 따라갑니다.

```powershell
# 리소스 그룹 생성
az group create --name k8s-4days-rg --location koreacentral

# AKS 클러스터 생성 (약 3~5분 소요)
az aks create `
  --resource-group k8s-4days-rg `
  --name k8s-4days-aks `
  --node-count 2 `
  --node-vm-size Standard_B2s `
  --generate-ssh-keys
```

---

### 2) 개별 작업 — kubeconfig 가져오기

클러스터 생성이 완료되면 각자 아래 명령어를 실행해 kubeconfig를 로컬에 가져옵니다.

```powershell
az aks get-credentials --resource-group k8s-4days-rg --name k8s-4days-aks
```

연결 확인:

```powershell
kubectl config current-context
kubectl get nodes
```

`k8s-4days-aks` 컨텍스트로 노드 목록이 출력되면 정상입니다.

**kubeconfig 파일 위치**

| OS | 경로 |
|----|------|
| macOS / Linux | `~/.kube/config` |
| Windows | `C:\Users\<사용자명>\.kube\config` |

=== "macOS/Linux"
    ```bash
    cat ~/.kube/config
    ```
=== "Windows PowerShell"
    ```powershell
    Get-Content "$env:USERPROFILE\.kube\config"
    ```

!!! warning "컨텍스트 주의"
    이후 실습은 모두 이 AKS 클러스터 위에서 진행됩니다.
    Rancher Desktop 등 로컬 클러스터가 함께 있는 경우 컨텍스트가 섞이지 않도록 주의하세요.

---

## 실습 목표

- Gateway API의 3계층 구조(GatewayClass / Gateway / HTTPRoute)를 이해하고 직접 배포한다.
- 가중치 기반 트래픽 분산(Canary)과 헤더 기반 라우팅을 구현한다.
- Ingress와 Gateway API의 차이를 실습을 통해 체감한다.

---

## Gateway API란?

Ingress는 하나의 리소스에 모든 규칙이 담깁니다. 팀이 커지면 여러 팀이 같은 파일을 건드려야 해서 충돌이 생깁니다. Gateway API는 이 문제를 **역할별 리소스 분리**로 해결합니다.

```
GatewayClass  →  "어떤 종류의 게이트웨이를 쓸 것인가" (클러스터 관리팀)
Gateway       →  "게이트웨이 인스턴스 설정" (플랫폼 팀)
HTTPRoute     →  "이 경로는 이 Service로" (개발팀)
```

세 리소스가 서로 참조하는 구조여서, 각 팀이 자신의 리소스만 수정하면 됩니다.

---

## Helm이란?

**Helm**은 Kubernetes의 패키지 매니저입니다. macOS의 `brew`, Ubuntu의 `apt`처럼 복잡한 애플리케이션을 명령어 하나로 설치·업그레이드·삭제할 수 있게 해줍니다.

Kubernetes에 어떤 애플리케이션을 설치하려면 Deployment, Service, ConfigMap, RBAC 등 수십 개의 YAML을 직접 작성해야 합니다. Helm은 이 YAML 묶음을 **Chart**라는 단위로 패키징해서 배포합니다.

```
helm install <릴리즈 이름> <차트 경로 또는 저장소>
helm uninstall <릴리즈 이름>
helm upgrade <릴리즈 이름> <차트>
```

이번 실습에서는 Envoy Gateway를 Helm Chart로 설치합니다.

---

### 1) Envoy Gateway 설치

> helm이 없다면 먼저 설치합니다.
>
> === "macOS"
>     ```bash
>     brew install helm
>     ```
> === "Windows (PowerShell)"
>     ```powershell
>     winget install Helm.Helm
>     ```

**Controller + CRD 설치 (Helm)**

=== "macOS / Linux"

    ```bash
    helm install eg oci://docker.io/envoyproxy/gateway-helm \
      --version v1.1.0 \
      -n envoy-gateway-system \
      --create-namespace
    ```

=== "Windows (PowerShell)"

    ```powershell
    helm install eg oci://docker.io/envoyproxy/gateway-helm `
      --version v1.1.0 `
      -n envoy-gateway-system `
      --create-namespace
    ```

Pod가 Running이 될 때까지 기다립니다. (1~2분 소요)

```bash
kubectl get pods -n envoy-gateway-system
```

**GatewayClass 생성**

Helm 차트는 Controller와 CRD만 설치합니다. GatewayClass 오브젝트는 별도로 생성해야 합니다.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
EOF
```

확인:

```bash
kubectl get gatewayclass
```

```
NAME   CONTROLLER                                             ACCEPTED
eg     gateway.envoyproxy.io/gatewayclass-controller          True
```

`ACCEPTED: True`가 되면 다음 단계로 진행합니다.

---

### 2) 테스트용 앱 배포

Gateway API 실습용 앱을 `gateway-ns` 네임스페이스에 배포합니다.

```bash
kubectl create namespace gateway-ns
```

`gw-apps.yaml` 파일을 만듭니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue-app
  namespace: gateway-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blue-app
  template:
    metadata:
      labels:
        app: blue-app
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo:latest
          args:
            - "-text=Blue App v1"
            - "-listen=:5678"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: blue-svc
  namespace: gateway-ns
spec:
  selector:
    app: blue-app
  ports:
    - port: 80
      targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: green-app
  namespace: gateway-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: green-app
  template:
    metadata:
      labels:
        app: green-app
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo:latest
          args:
            - "-text=Green App v2"
            - "-listen=:5678"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: green-svc
  namespace: gateway-ns
spec:
  selector:
    app: green-app
  ports:
    - port: 80
      targetPort: 5678
```

```bash
kubectl apply -f gw-apps.yaml
kubectl get pods -n gateway-ns
kubectl get svc -n gateway-ns
```

---

### 3) Gateway 생성

`Gateway`는 실제 리스닝 포트와 프로토콜을 정의합니다. 플랫폼 팀이 관리하는 리소스입니다.

`gateway.yaml` 파일을 만듭니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: lab-gateway
  namespace: gateway-ns
spec:
  gatewayClassName: eg          # GatewayClass 이름 참조
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same            # 같은 네임스페이스의 HTTPRoute만 허용
```

```bash
kubectl apply -f gateway.yaml
kubectl get gateway -n gateway-ns
```

```
NAME          CLASS   ADDRESS        PROGRAMMED   AGE
lab-gateway   eg      <IP>           True         30s
```

`PROGRAMMED: True`가 되면 Gateway가 정상적으로 준비된 것입니다.

```bash
kubectl describe gateway lab-gateway -n gateway-ns
```

확인 항목:

- `Listeners`: 포트·프로토콜 설정
- `Addresses`: 할당된 IP
- `Conditions`: Ready 여부

!!! warning "PROGRAMMED: False가 지속되는 경우"
    이전 실습(3-1)에서 Nginx Ingress Controller를 삭제하지 않았으면 포트 80이 충돌해 Gateway에 주소가 할당되지 않습니다.

    ```bash
    kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
    ```

    삭제 후 1분 정도 기다렸다가 `kubectl get gateway -n gateway-ns`를 다시 확인합니다.

---

### 4) HTTPRoute 생성 — 경로 기반 라우팅

`HTTPRoute`는 "어떤 경로를 어느 Service로 보낼 것인가"를 정의합니다. 개발팀이 직접 관리하는 리소스입니다.

`httproute-path.yaml` 파일을 만듭니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: lab-route
  namespace: gateway-ns
spec:
  parentRefs:
    - name: lab-gateway         # 어느 Gateway에 붙을지 참조
  hostnames:
    - "gw.lab.local"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /blue
      backendRefs:
        - name: blue-svc
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /green
      backendRefs:
        - name: green-svc
          port: 80
```

```bash
kubectl apply -f httproute-path.yaml
kubectl get httproute -n gateway-ns
kubectl describe httproute lab-route -n gateway-ns
```

`describe` 출력에서 확인:

- `ParentRefs`: 연결된 Gateway
- `Rules`: path → backendRefs Service 매핑

---

### 5) /etc/hosts 등록 및 접속 확인

`/etc/hosts`에 등록합니다.

=== "macOS / Linux"

    ```bash
    sudo sh -c 'echo "127.0.0.1 gw.lab.local" >> /etc/hosts'
    ```

=== "Windows (PowerShell — 관리자 권한)"

    ```powershell
    Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1 gw.lab.local"
    ```

접속 확인:

=== "macOS / Linux"

    ```bash
    curl http://gw.lab.local/blue
    # Blue App v1

    curl http://gw.lab.local/green
    # Green App v2
    ```

=== "Windows (PowerShell)"

    ```powershell
    Invoke-WebRequest -Uri http://gw.lab.local/blue -UseBasicParsing | Select-Object -ExpandProperty Content
    Invoke-WebRequest -Uri http://gw.lab.local/green -UseBasicParsing | Select-Object -ExpandProperty Content
    ```

---

### 6) HTTPRoute — 가중치 기반 트래픽 분산 (Canary)

Gateway API의 강력한 기능 중 하나입니다. Ingress에서는 annotation으로 억지로 구현하던 것을 `weight` 필드 하나로 표준화해서 지원합니다.

`httproute-canary.yaml` 파일을 만듭니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-route
  namespace: gateway-ns
spec:
  parentRefs:
    - name: lab-gateway
  hostnames:
    - "gw.lab.local"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /app
      backendRefs:
        - name: blue-svc    # 기존 버전 (v1)
          port: 80
          weight: 80        # 트래픽의 80%
        - name: green-svc   # 신규 버전 (v2)
          port: 80
          weight: 20        # 트래픽의 20%
```

```bash
kubectl apply -f httproute-canary.yaml
```

실제로 분산되는지 확인합니다. 10번 요청해서 비율을 봅니다.

=== "macOS / Linux"

    ```bash
    for i in $(seq 1 10); do curl -s http://gw.lab.local/app; echo; done
    ```

=== "Windows (PowerShell)"

    ```powershell
    1..10 | ForEach-Object {
        Invoke-WebRequest -Uri http://gw.lab.local/app -UseBasicParsing | Select-Object -ExpandProperty Content
    }
    ```

```
Blue App v1
Blue App v1
Green App v2
Blue App v1
Blue App v1
Blue App v1
Blue App v1
Green App v2
Blue App v1
Blue App v1
```

약 8:2 비율로 응답이 분산되는 것을 확인합니다.

> **실무 활용**: 신버전 배포 시 처음에는 `weight: 5`(5%)만 신버전으로 보내다가 문제없으면 `weight: 50` → `weight: 100`으로 점진적으로 올립니다. Ingress에서는 이 작업이 Controller별로 구현 방식이 달랐지만, Gateway API에서는 표준 YAML 필드로 통일됩니다.

---

### 7) 정리

**HTTPRoute / Gateway / 앱 삭제**

```bash
kubectl delete -f httproute-canary.yaml
kubectl delete -f httproute-path.yaml
kubectl delete -f gateway.yaml
kubectl delete -f gw-apps.yaml
kubectl delete namespace gateway-ns
```

**Envoy Gateway (Helm) 삭제**

=== "macOS / Linux"

    ```bash
    helm uninstall eg -n envoy-gateway-system
    ```

=== "Windows (PowerShell)"

    ```powershell
    helm uninstall eg -n envoy-gateway-system
    ```

**Gateway API CRD 삭제**

Helm uninstall 후에도 CRD는 클러스터에 남습니다. 완전히 제거하려면 아래 명령어를 실행합니다.

=== "macOS / Linux"

    ```bash
    kubectl get crd | grep gateway.networking.k8s.io | awk '{print $1}' | xargs kubectl delete crd

    # 확인
    kubectl get crd | grep gateway
    ```

=== "Windows (PowerShell)"

    ```powershell
    kubectl get crd -o name | Select-String "gateway.networking.k8s.io" | ForEach-Object {
        kubectl delete $_.ToString().Trim()
    }

    # 확인
    kubectl get crd | Select-String "gateway"
    ```

> **왜 CRD를 따로 지워야 하나?**
> Helm은 기본적으로 CRD를 **설치는 하지만 삭제는 하지 않습니다.** CRD를 자동 삭제하면 데이터 손실 위험이 있기 때문입니다. 다음 실습에서 버전 충돌이 생길 수 있으니 명시적으로 삭제합니다.

**`/etc/hosts` 원복**

=== "macOS / Linux"

    ```bash
    sudo sed -i '' '/gw\.lab\.local/d' /etc/hosts

    # 확인
    cat /etc/hosts | grep gw.lab.local
    ```

=== "Windows (PowerShell — 관리자 권한)"

    ```powershell
    $hosts = "C:\Windows\System32\drivers\etc\hosts"
    (Get-Content $hosts) | Where-Object { $_ -notmatch "gw\.lab\.local" } | Set-Content $hosts

    # 확인
    Get-Content $hosts | Select-String "gw.lab.local"
    ```

---

## Ingress vs Gateway API 비교

| 항목 | Ingress | Gateway API |
|---|---|---|
| 라우팅 단위 | 단일 리소스 | GatewayClass / Gateway / HTTPRoute 분리 |
| 팀 역할 분리 | 불가 | 가능 (RBAC 연동) |
| 가중치 트래픽 분산 | Controller별 annotation (비표준) | `weight` 필드 (표준) |
| 헤더 기반 라우팅 | Controller별 annotation | `headers` 필드 (표준) |
| 도입 복잡도 | 낮음 | 상대적으로 높음 |
| 권장 상황 | 소규모·단순 라우팅 | 복잡한 트래픽 제어·대규모 팀 |

---

## 트러블슈팅

| 증상 | 확인 사항 |
|---|---|
| `curl: (6) Could not resolve host` | `/etc/hosts`에 도메인 등록 확인 |
| Gateway `PROGRAMMED: False` | Nginx Ingress Controller가 포트 80을 점유 중 — `kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml` 후 재확인 |
| HTTPRoute 적용 안됨 | `parentRefs.name`이 Gateway 이름과 일치하는지 확인 |
| Canary 비율이 맞지 않음 | `weight` 합계가 100일 필요는 없음, 비율로만 계산됨 |
