# 🔧 도전 과제 — 조건에 맞춰 Deployment 만들기

**난이도 ★★☆**

> 명령어는 직접 찾아 해결하세요 · 아래 조건만 보고 배포부터 접속까지 완료합니다

---

## 1 · 템플릿 뽑기

- Deployment를 손으로 작성하지 말고, 명령어로 YAML 템플릿을 생성해 파일로 저장
- 파일명: `web-deploy.yaml`

---

## 2 · 조건에 맞게 수정

- Deployment 이름: `web-deploy` · 이미지: `docker/getting-started` · 복제본(replicas): **3개**
- 라벨(labels): `app=web`, `tier=frontend`
- selector: 위 라벨과 정확히 일치하도록 작성 (라벨 3곳이 어긋나면 배포 실패)
- 컨테이너 포트: **80**

---

## 3 · 배포 & 확인

- 선언형(YAML 파일)으로 배포
- Pod 3개가 모두 Running인지 확인
- 라벨 셀렉터로 해당 Pod만 골라내기 (`app=web, tier=frontend`)

---

## 4 · 접속 (port-forward 8000)

- 로컬 **8000** 포트 → 컨테이너 **80** 포트로 연결
- 브라우저에서 `http://localhost:8000` 접속 → getting-started 페이지 확인

---

!!! tip "확인 포인트"
    - 템플릿을 명령어로 뽑았는가? (직접 작성 금지)
    - `labels` ↔ `selector` ↔ `template.labels` 세 곳 일치
    - 라벨 셀렉터로 Pod를 정확히 선택했는가?
    - port-forward 터미널은 켠 채로 다른 창에서 접속했는가?
