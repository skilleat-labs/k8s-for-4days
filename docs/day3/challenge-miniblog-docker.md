# 🔧 도전 과제 — 미니 블로그 배포 (Docker Desktop)

**난이도 ★★★**

> 명령어와 YAML은 직접 작성하세요 · 아래 조건만 보고 완료합니다

---

## 시작 전 확인

Envoy Gateway와 GatewayClass는 이미 설치되어 있습니다. 확인만 하고 넘어갑니다.

```bash
kubectl get pods -n envoy-gateway-system
kubectl get gatewayclass
```

- Envoy Gateway Pod가 `Running` 상태인지 확인
- GatewayClass `eg`의 `ACCEPTED`가 `True`인지 확인

---

## 시나리오

미니 블로그 애플리케이션을 Docker Desktop 클러스터에 배포합니다.
Frontend와 Backend로 구성되며, Gateway API를 통해 `http://localhost`로 접속할 수 있어야 합니다.

---

## 조건

### 1 · Namespace

- 이름: `webapp`
- 모든 리소스는 이 Namespace 안에 생성

---

### 2 · Backend

- 이미지: `skilleat/backend:v3-kb5`
- replicas: **1**
- 컨테이너 포트: `5000`
- Service 이름: `backend-service`, port: `5000`, targetPort: `5000`

---

### 3 · Frontend

- 이미지: `skilleat/frontend:v3-kb5`
- replicas: **1**
- 컨테이너 포트: `80`
- Service 이름: `frontend-service`, port: `80`, targetPort: `80`

---

### 4 · Gateway API

GatewayClass(`eg`)는 이미 생성되어 있습니다. Gateway와 HTTPRoute만 작성하세요.


---

## 성공 조건

- [ ] `webapp` Namespace 안에 모든 리소스가 생성됨
- [ ] Backend, Frontend Pod가 `Running` 상태
- [ ] Gateway에 `localhost`가 EXTERNAL-IP로 할당됨
- [ ] 브라우저에서 `http://localhost` 접속 → 블로그 페이지 확인
- [ ] 게시글 작성 성공

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
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: backend
      namespace: webapp
    spec:
      replicas: 1
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
    kind: Gateway
    metadata:
      name: blog-gw
      namespace: webapp
    spec:
      gatewayClassName: eg
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
