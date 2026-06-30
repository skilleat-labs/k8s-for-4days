# 디버깅 도전 과제 — 고장난 앱을 고쳐라

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
          image: skilleat/rollout-demo:v999
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
    app: wrong-app
  ports:
    - port: 80
      targetPort: 9999
      nodePort: 30090
```

```bash
kubectl apply -f broken-app.yaml
```

---

## 성공 조건

3개의 버그를 모두 수정하고 아래를 확인하세요.

```bash
# 1. 모든 Pod가 Running 상태
kubectl get pods -l app=myapp

# 2. Endpoints에 Pod IP가 들어와 있음
kubectl get endpoints broken-svc
```

=== "macOS/Linux"
    ```bash
    # 3. 브라우저 또는 curl로 페이지 확인
    curl http://localhost:30090
    ```
=== "Windows PowerShell"
    ```powershell
    # 3. 브라우저 또는 curl로 페이지 확인
    curl.exe http://localhost:30090
    ```

`http://localhost:30090`에서 페이지가 보이면 성공입니다.

---

## 정리

```bash
kubectl delete -f broken-app.yaml
```
