# 도전 과제 — Gateway API + PVC 미니 블로그

## 시나리오

미니 블로그 애플리케이션을 클라우드 매니지드 쿠버네티스 클러스터에 배포합니다.

Backend Replica 2개가 **서로 다른 노드**에 분산 배포되면서도 **동일한 데이터(data.json)**를 공유해야 합니다.
Gateway API를 통해 외부에서 접속할 수 있도록 전체 구조를 직접 설계하고 구현하세요.

---

## 전체 아키텍처

```
사용자
  ↓
Gateway (External IP)
  ↓
HTTPRoute
  ↓
frontend-service → frontend Pods (1개)
  ↓
backend-service → backend Pods (2개, 노드 분산)
  ↓
PVC → PV → StorageClass
     (data.json 공유 저장)
```

---

## 사전 확인

### 클러스터 및 노드 확인

```bash
kubectl get nodes -o wide
```

노드가 2개 이상 존재하는지 확인합니다. 이후 Backend Pod 2개가 각각 다른 노드에 스케줄됐는지 검증에 사용합니다.

### 사용 가능한 StorageClass 확인

```bash
kubectl get storageclass
```

클라우드 환경별 RWX(ReadWriteMany)를 지원하는 StorageClass를 확인합니다.

| 클라우드 | RWX 지원 StorageClass 예시 |
|---------|--------------------------|
| AKS (Azure) | `azurefile`, `azurefile-csi` |
| GKE (Google) | `filestore` (별도 프로비저너 설치) |
| EKS (AWS) | `efs-sc` (EFS CSI Driver 설치 필요) |

!!! warning "RWX가 필요한 이유"
    Backend Pod 2개가 **서로 다른 노드**에 배포될 경우 `ReadWriteOnce(RWO)` PVC는 단일 노드에서만 마운트 가능합니다.
    여러 노드에 걸쳐 데이터를 공유하려면 반드시 **ReadWriteMany(RWX)** 를 지원하는 StorageClass가 필요합니다.

---

## STEP 1. Namespace 생성

**요구 조건**

- Namespace 이름: `webapp`
- 모든 리소스(Deployment, Service, PVC, HTTPRoute 등)는 이 namespace에서 생성

---

## STEP 2. 기본 Namespace 전환

kubectl 명령이 기본적으로 `webapp` namespace를 바라보도록 설정합니다.

**요구 조건**

- 현재 kubectl context가 `webapp`을 기본 namespace로 사용해야 함

**검증**

```bash
kubectl config view --minify | grep namespace:
# namespace: webapp 이 출력되어야 함
```

---

## STEP 3. PVC 생성

**요구 조건**

| 항목 | 값 |
|------|---|
| PVC 이름 | `blog-pvc` |
| StorageClassName | 직접 확인 후 RWX 지원 클래스 기재 |
| accessModes | `ReadWriteMany` |
| 용량 | `1Gi` |

**검증**

```bash
kubectl get pvc blog-pvc -n webapp
# STATUS가 Bound 이어야 함
```

---

## STEP 4. Backend Deployment 구축

**요구 조건**

| 항목 | 값 |
|------|---|
| 이름 | `backend` |
| replicas | `2` |
| image | `skilleat/backend:v3-kb5` |
| containerPort | `5000` |
| volumeMount 경로 | `/app/data` (PVC `blog-pvc` 마운트) |
| label | `app=backend` |

**노드 분산 검증**

```bash
kubectl get pods -n webapp -l app=backend -o wide
# 두 Pod가 서로 다른 NODE에 배치되어 있어야 함
```

---

## STEP 5. Backend Service 생성

**요구 조건**

| 항목 | 값 |
|------|---|
| 이름 | `backend-service` |
| selector | `app=backend` |
| port → targetPort | `5000 → 5000` |

**검증**

```bash
kubectl get endpoints backend-service -n webapp
# 2개의 Pod IP가 Endpoints에 등록되어 있어야 함
```

---

## STEP 6. Frontend Deployment 생성

**요구 조건**

| 항목 | 값 |
|------|---|
| Deployment 이름 | `frontend` |
| replicas | `1` |
| image | `skilleat/frontend:v3-kb5` |
| containerPort | `80` |
| label | `app=frontend` |

---

## STEP 7. Frontend Service 생성

**요구 조건**

| 항목 | 값 |
|------|---|
| 이름 | `frontend-service` |
| selector | `app=frontend` |
| port → targetPort | `80 → 80` |

---

## STEP 8. GatewayClass + Gateway 생성

**GatewayClass 요구 조건**

| 항목 | 값 |
|------|---|
| 이름 | `blog-gw-class` |
| controllerName | `gateway.envoyproxy.io/gatewayclass-controller` |

**Gateway 요구 조건**

| 항목 | 값 |
|------|---|
| 이름 | `blog-gw` |
| gatewayClassName | `blog-gw-class` |
| Listener 이름 | `http` |
| port | `80` |
| protocol | `HTTP` |

**검증**

```bash
kubectl get gateway blog-gw -n webapp
# PROGRAMMED 상태가 True 이어야 함

kubectl get svc -n envoy-gateway-system
# blog-gw 관련 서비스에 EXTERNAL-IP가 할당되어 있어야 함
```

---

## STEP 9. HTTPRoute 구성

**요구 조건**

| 항목 | 값 |
|------|---|
| 이름 | `frontend-route` |
| parentRefs | Gateway `blog-gw` (webapp namespace) |
| pathPrefix | `/` |
| backendRef | `frontend-service` port `80` |

**검증**

```bash
kubectl get httproute -n webapp
```

---

## STEP 10. 접속 확인

클라우드 매니지드 환경에서는 Gateway에 External IP가 자동 할당됩니다.

```bash
# Gateway External IP 확인
kubectl get svc -n envoy-gateway-system

# 브라우저 또는 curl로 접속
curl http://<EXTERNAL-IP>
```

!!! info "External IP 할당 지연"
    클라우드 LoadBalancer는 IP 할당에 1~3분 정도 소요될 수 있습니다.
    `kubectl get svc -n envoy-gateway-system -w`로 실시간 확인하세요.

---

## STEP 11. 최종 검증

### 1. 블로그 게시글 작성 및 공유 확인

1. `http://<EXTERNAL-IP>` 에 접속
2. 새 글 작성
3. 새로고침 후에도 데이터가 유지되는지 확인

### 2. 각 Backend Pod에서 파일 직접 확인

```bash
# Pod 이름 조회
kubectl get pods -n webapp -l app=backend

# 각 Pod에서 data.json 확인
kubectl exec -n webapp <backend-pod-1> -- cat /app/data/data.json
kubectl exec -n webapp <backend-pod-2> -- cat /app/data/data.json
# 두 Pod 모두 동일한 데이터를 가지고 있어야 함
```

### 3. 노드 분산 최종 확인

```bash
kubectl get pods -n webapp -o wide
# backend Pod 2개가 서로 다른 NODE에 배치되어 있어야 함
```

---

## 성공 조건 체크리스트

- [ ] `webapp` namespace에 모든 리소스가 생성됨
- [ ] `blog-pvc` STATUS가 `Bound`
- [ ] Backend Pod 2개가 **서로 다른 노드**에 배포됨
- [ ] `backend-service` Endpoints에 2개 IP 등록됨
- [ ] Gateway External IP가 할당됨
- [ ] 브라우저에서 블로그 페이지 접속 성공
- [ ] 게시글 작성 후 두 Backend Pod 모두 동일한 `data.json` 보유

---

## 정리

```bash
kubectl delete namespace webapp
```

