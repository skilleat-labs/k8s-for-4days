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

## 구현 조건

### Namespace

- 이름: `webapp`
- 모든 리소스는 이 namespace 안에서 생성

### StorageClass / PVC

!!! warning "RWX가 필요한 이유"
    Backend Pod 2개가 서로 다른 노드에 배포될 경우 `ReadWriteOnce(RWO)` PVC는 단일 노드에서만 마운트 가능합니다.
    여러 노드에 걸쳐 데이터를 공유하려면 반드시 **ReadWriteMany(RWX)** 를 지원하는 StorageClass가 필요합니다.

| 클라우드 | RWX 지원 StorageClass 예시 |
|---------|--------------------------|
| AKS (Azure) | `azurefile`, `azurefile-csi` |
| GKE (Google) | `filestore` (별도 프로비저너 설치) |
| EKS (AWS) | `efs-sc` (EFS CSI Driver 설치 필요) |

| 항목 | 값 |
|------|---|
| PVC 이름 | `blog-pvc` |
| StorageClassName | 클러스터에서 직접 확인 후 RWX 지원 클래스 기재 |
| accessModes | `ReadWriteMany` |
| 용량 | `1Gi` |

### Backend

| 항목 | 값 |
|------|---|
| Deployment 이름 | `backend` |
| replicas | `2` |
| image | `skilleat/backend:v3-kb5` |
| containerPort | `5000` |
| volumeMount 경로 | `/app/data` — PVC `blog-pvc` 마운트 |
| label | `app=backend` |
| Service 이름 | `backend-service` |
| port → targetPort | `5000 → 5000` |

### Frontend

| 항목 | 값 |
|------|---|
| Deployment 이름 | `frontend` |
| replicas | `1` |
| image | `skilleat/frontend:v3-kb5` |
| containerPort | `80` |
| label | `app=frontend` |
| Service 이름 | `frontend-service` |
| port → targetPort | `80 → 80` |

### GatewayClass

| 항목 | 값 |
|------|---|
| 이름 | `blog-gw-class` |
| controllerName | `gateway.envoyproxy.io/gatewayclass-controller` |

### Gateway

| 항목 | 값 |
|------|---|
| 이름 | `blog-gw` |
| gatewayClassName | `blog-gw-class` |
| Listener 이름 | `http` |
| port | `80` |
| protocol | `HTTP` |

### HTTPRoute

| 항목 | 값 |
|------|---|
| 이름 | `frontend-route` |
| parentRefs | Gateway `blog-gw` (webapp namespace) |
| pathPrefix | `/` |
| backendRef | `frontend-service` port `80` |

---

## 성공 조건

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
