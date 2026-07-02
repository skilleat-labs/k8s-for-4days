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
- Service로 노출

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
