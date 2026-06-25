# 🔧 디버깅 도전 과제 — 고장난 앱을 고쳐라

## 시나리오

당신은 오늘 온콜 당번입니다.

팀원이 퇴근 직전에 `broken-app`을 배포하고 갔는데, 앱이 정상적으로 동작하지 않습니다.
`http://localhost:30090`에 접속하면 아무 응답이 없는 상태입니다.

**YAML 파일 어딘가에 버그가 3개 있습니다. 모두 찾아서 고쳐야 페이지가 보입니다.**

---

## 실습 목표

- `kubectl get`, `kubectl describe`, `kubectl logs`로 문제를 스스로 추적한다.
- Deployment / Service 의 관계를 이해하고 버그를 수정한다.
- 3개의 버그를 모두 고쳐 `http://localhost:30090`에서 정상 페이지를 확인한다.

---

## 준비 — 고장난 앱 배포

아래 내용을 `broken-app.yaml`로 저장하고 배포합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: skilleat/rollout-demo:v999   # 🐛
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: broken-svc
spec:
  type: NodePort
  selector:
    app: wrong-app   # 🐛
  ports:
    - port: 80
      targetPort: 9999   # 🐛
      nodePort: 30090
```

```bash
kubectl apply -f broken-app.yaml
```

---

## 지금 상태 확인

```bash
kubectl get pods
kubectl get svc broken-svc
kubectl get endpoints broken-svc
```

`http://localhost:30090`에 접속해보세요. 응답이 없습니다.

---

## 🔍 단서 — 어디서부터 볼까?

아래 명령어들로 문제를 추적해보세요.

```bash
# Pod 상태 확인
kubectl get pods

# Pod 상세 이벤트 확인
kubectl describe pod <pod이름>

# Service 상세 확인
kubectl describe svc broken-svc

# Endpoints 확인 (비어있으면 Service가 Pod를 못 찾는 것)
kubectl get endpoints broken-svc
```

---

## ✅ 성공 조건

3개의 버그를 모두 수정하고 아래를 확인하세요.

```bash
# 1. 모든 Pod가 Running 상태
kubectl get pods -l app=myapp

# 2. Endpoints에 Pod IP가 들어와 있음
kubectl get endpoints broken-svc

# 3. 브라우저 또는 curl로 페이지 확인
curl http://localhost:30090
```

`http://localhost:30090`에서 페이지가 보이면 성공입니다.

---

## 정리

```bash
kubectl delete -f broken-app.yaml
```

---

??? hint "힌트 1 — Bug 1 위치"
    `kubectl get pods` 실행 후 STATUS 컬럼을 확인하세요.
    `ImagePullBackOff` 또는 `ErrImagePull`이 보인다면 이미지 이름이나 태그에 문제가 있습니다.
    `kubectl describe pod <pod이름>` 의 Events 섹션에서 정확한 오류를 확인하세요.

??? hint "힌트 2 — Bug 2 위치"
    `kubectl get endpoints broken-svc` 결과가 `<none>`이면
    Service의 `selector`가 Pod의 `labels`와 일치하지 않는 것입니다.
    `kubectl get pods --show-labels`로 실제 Pod label을 확인하고 Service selector와 비교하세요.

??? hint "힌트 3 — Bug 3 위치"
    Endpoints에 Pod IP가 들어왔는데도 `curl http://localhost:30090`이 실패한다면
    Service의 `targetPort`가 컨테이너가 실제로 열고 있는 포트와 다른 것입니다.
    `kubectl describe pod <pod이름>`에서 `Containers > Ports` 항목을 확인하세요.

??? hint "정답"
    **Bug 1**: `image: skilleat/rollout-demo:v999` → `skilleat/rollout-demo:v1.0.0`

    **Bug 2**: Service `selector: app: wrong-app` → `app: myapp`

    **Bug 3**: Service `targetPort: 9999` → `8080`
