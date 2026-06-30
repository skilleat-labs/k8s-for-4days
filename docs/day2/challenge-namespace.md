# 🔧 도전 과제 — Namespace에 앱 배포하고 관찰하기

**난이도 ★★☆**

> 명령어는 직접 찾아 해결하세요 · 아래 조건만 보고 완료합니다

---

## 조건

### 1 · Namespace 생성

- 이름: `guide-ns`

---

### 2 · Deployment 배포 (replicas: 1)

`guide-ns` 네임스페이스 안에 아래 조건으로 Deployment를 배포하세요.

| 항목 | 값 |
|------|----|
| Deployment 이름 | `guide` |
| 이미지 | `docker/getting-started` |
| 복제본(replicas) | **1** |
| 컨테이너 포트 | **80** |
| 배포 네임스페이스 | `guide-ns` |

---

### 3 · Service 생성 (NodePort)

- `guide-ns` 네임스페이스 안에 NodePort 타입으로 Service를 생성하세요.
- NodePort 번호: **30088**

---

### 4 · 접속 확인 (replicas: 1)

- 브라우저에서 `http://localhost:30088` 접속 → getting-started 페이지 확인

---

### 5 · replicas를 2로 변경

- Deployment의 복제본을 **2**로 변경하세요.
- Pod 2개가 모두 `Running` 상태인지 확인하세요.

---

### 6 · 관찰

- `http://localhost:30088`에서 브라우저를 **여러 번 새로고침**해보세요.
- 무언가 이상한 점을 발견할 수 있습니까?
- `kubectl` 명령어로 두 Pod의 상태를 확인하면서 어떤 일이 벌어지는지 생각해보세요.

---

!!! note "아키텍처 주의사항"
    - Deployment와 Service는 **같은 Namespace** 안에 있어야 서로 연결됩니다.
    - Service의 `selector`는 Deployment의 Pod `labels`와 **정확히 일치**해야 합니다.
    - NodePort는 클러스터 전체 노드에서 동일한 포트로 열립니다. Rancher Desktop / Docker Desktop 환경에서는 노드 IP 대신 `localhost`를 사용하세요.
    - replicas가 2 이상일 때 Service는 요청을 **여러 Pod에 분산**합니다.
