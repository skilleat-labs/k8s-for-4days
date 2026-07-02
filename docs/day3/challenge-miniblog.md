# 🔧 도전 과제 — 미니 블로그 배포

**난이도 ★★★**

> 명령어와 YAML은 직접 작성하세요 · 아래 조건만 보고 완료합니다

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

- Envoy Gateway 설치 방법은 [Gateway API 실습](3-2-gateway.md) 참고
- GatewayClass → Gateway → HTTPRoute 순서로 구성
- `/` 경로로 들어오는 요청이 Frontend로 라우팅되어야 함

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
