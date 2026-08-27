# 🔧 도전 과제 — 미니 블로그 배포

**난이도 ★★★**

> 명령어와 YAML은 직접 작성하세요 · 아래 조건만 보고 완료합니다

---

## 시작 전 준비

### 1) AKS 로그인

```powershell
az login --use-device-code
```

터미널에 `https://microsoft.com/devicelogin` 주소와 코드가 출력됩니다. 브라우저에서 해당 주소를 열고 코드를 입력해 로그인합니다.

### 2) 내 클러스터에 연결

본인 번호에 맞는 클러스터 이름을 확인하고 credentials를 가져옵니다.

| 수강생 | 리소스 그룹 | 클러스터 이름 |
|--------|------------|--------------|
| 1번 | `user01-rg` | `aks-user01` |
| 2번 | `user02-rg` | `aks-user02` |
| 3번 | `user03-rg` | `aks-user03` |
| 4번 | `user04-rg` | `aks-user04` |

```powershell
az aks get-credentials --resource-group user01-rg --name aks-user01
```

!!! warning "본인 번호로 바꾸세요"
    `user01-rg`와 `aks-user01` 두 곳 모두 본인 번호로 변경하세요. (예: 2번 → `user02-rg`, `aks-user02`)

연결 확인:

```powershell
kubectl config current-context
kubectl get nodes
```

노드 목록이 출력되면 정상입니다.

### 3) Envoy Gateway 설치

Helm이 없으면 먼저 설치합니다.

```powershell
winget install Helm.Helm
```

설치 후 PowerShell을 재시작하고 Envoy Gateway를 설치합니다.

```powershell
helm install eg oci://docker.io/envoyproxy/gateway-helm `
  --version v1.1.0 `
  -n envoy-gateway-system `
  --create-namespace
```

Pod가 Running 상태가 될 때까지 기다립니다. (1~2분)

```powershell
kubectl get pods -n envoy-gateway-system
```

모든 Pod가 `Running`이면 다음으로 넘어갑니다.

### 4) GatewayClass 생성

```powershell
@"
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
"@ | kubectl apply -f -
```

확인:

```powershell
kubectl get gatewayclass
```

`ACCEPTED: True`가 되면 도전 과제를 시작합니다.

---

## 시나리오

미니 블로그 애플리케이션을 AKS 클러스터에 배포합니다.
Frontend와 Backend로 구성되며, Backend 2개가 **동일한 데이터 파일을 공유**해야 합니다.
외부에서 Gateway API를 통해 접속할 수 있어야 합니다.

---

## 조건

### 1 · Namespace

- 이름: `webapp`
- 모든 리소스는 이 Namespace 안에 생성

---

### 2 · 스토리지

- Backend Pod 2개가 **서로 다른 노드**에서 동일한 파일을 공유할 수 있어야 합니다.
- 어떤 StorageClass와 accessMode를 써야 할지 스스로 판단하세요.
- PVC 이름: `blog-pvc`, 용량: `1Gi`

---

### 3 · Backend

- 이미지: `skilleat/backend:v3-kb5`
- 2개 replicas, **서로 다른 노드**에 분산 배포
- `/app/data` 경로에 `blog-pvc` 마운트
- Service 이름: `backend-service`, 포트: `5000` (Frontend 이미지에 하드코딩되어 있으므로 반드시 일치해야 함)

---

### 4 · Frontend

- 이미지: `skilleat/frontend:v3-kb5`
- 1개 replica
- Service 이름: `frontend-service`, 포트: `80`

---

### 5 · Gateway API

GatewayClass → Gateway → HTTPRoute 순서로 구성합니다.

**GatewayClass** — 아래 명령어를 그대로 실행하세요.

```powershell
@"
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
"@ | kubectl apply -f -
```

**Gateway** — 빈 칸을 채워서 `gateway.yaml`을 완성하세요.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ________          # Gateway 이름 (자유롭게)
  namespace: ________     # 앱과 같은 Namespace
spec:
  gatewayClassName: ______ # 위에서 만든 GatewayClass 이름
  listeners:
    - name: http
      protocol: ______    # HTTP 또는 HTTPS
      port: ______        # 외부에서 접속할 포트
```

**HTTPRoute** — 빈 칸을 채워서 `httproute.yaml`을 완성하세요.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ________
  namespace: ________
spec:
  parentRefs:
    - name: ________      # 연결할 Gateway 이름
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: ______  # 모든 경로를 받으려면?
      backendRefs:
        - name: ________   # 트래픽을 보낼 Service 이름
          port: ______     # Service 포트
```

---

## 성공 조건

- [ ] `blog-pvc` STATUS가 `Bound`
- [ ] Backend Pod 2개가 서로 다른 노드에서 Running
- [ ] Gateway에 External IP가 할당됨
- [ ] 브라우저에서 블로그 페이지 접속 성공
- [ ] 게시글 작성 후 두 Backend Pod가 동일한 데이터를 가짐

---

## 정리

```bash
kubectl delete namespace webapp
```


---

## 정답

??? success "정답 보기"

    ### namespace.yaml

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: webapp
    ```

    ### backend.yaml

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: blog-pvc
      namespace: webapp
    spec:
      accessModes:
        - ReadWriteMany
      storageClassName: azurefile-csi
      resources:
        requests:
          storage: 1Gi
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: backend
      namespace: webapp
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: backend
      template:
        metadata:
          labels:
            app: backend
        spec:
          containers:
            - name: backend
              image: skilleat/backend:v3-kb5
              ports:
                - containerPort: 5000
              volumeMounts:
                - name: data
                  mountPath: /app/data
          volumes:
            - name: data
              persistentVolumeClaim:
                claimName: blog-pvc
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: backend-service
      namespace: webapp
    spec:
      selector:
        app: backend
      ports:
        - port: 5000
          targetPort: 5000
    ```

    ### frontend.yaml

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: frontend
      namespace: webapp
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: frontend
      template:
        metadata:
          labels:
            app: frontend
        spec:
          containers:
            - name: frontend
              image: skilleat/frontend:v3-kb5
              ports:
                - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: frontend-service
      namespace: webapp
    spec:
      selector:
        app: frontend
      ports:
        - port: 80
          targetPort: 80
    ```

    ### gateway.yaml

    ```yaml
    apiVersion: gateway.networking.k8s.io/v1
    kind: GatewayClass
    metadata:
      name: blog-gw-class
    spec:
      controllerName: gateway.envoyproxy.io/gatewayclass-controller
    ---
    apiVersion: gateway.networking.k8s.io/v1
    kind: Gateway
    metadata:
      name: blog-gw
      namespace: webapp
    spec:
      gatewayClassName: blog-gw-class
      listeners:
        - name: http
          port: 80
          protocol: HTTP
    ---
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: frontend-route
      namespace: webapp
    spec:
      parentRefs:
        - name: blog-gw
      rules:
        - matches:
            - path:
                type: PathPrefix
                value: /
          backendRefs:
            - name: frontend-service
              port: 80
    ```

    ### 적용 순서

    ```bash
    kubectl apply -f namespace.yaml
    kubectl apply -f backend.yaml
    kubectl apply -f frontend.yaml
    kubectl apply -f gateway.yaml
    ```
