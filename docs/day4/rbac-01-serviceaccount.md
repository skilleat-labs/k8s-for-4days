# RBAC 01 — ServiceAccount 기초

## 실습 목표

- ServiceAccount가 무엇인지, 왜 필요한지 이해한다.
- 기본 SA와 사용자 정의 SA의 차이를 확인한다.
- Pod에 SA를 지정하고, 토큰이 자동 마운트되는 위치를 확인한다.

---

## 개념

### ServiceAccount란?

사람이 kubectl로 클러스터에 접근할 때 **User 계정**을 씁니다.
Pod 안의 앱이 API Server에 접근할 때는 **ServiceAccount(SA)** 를 씁니다.

```
사람       →  User Account  →  kubectl  →  API Server
Pod 안 앱  →  ServiceAccount →  토큰     →  API Server
```

SA를 지정하지 않으면 Namespace의 **default** SA가 자동으로 붙습니다.

### 토큰의 역할

Pod가 생성될 때 SA의 토큰이 컨테이너 안의 고정 경로에 자동으로 마운트됩니다.

```
/var/run/secrets/kubernetes.io/serviceaccount/
  ├── token      ← JWT 토큰 (API Server 인증용)
  ├── ca.crt     ← API Server TLS 인증서
  └── namespace  ← 현재 네임스페이스 이름
```

앱이 이 토큰으로 API Server에 요청을 보냅니다.

---

## Step 1. 기본 SA 확인

```bash
# 현재 네임스페이스의 SA 목록
kubectl get serviceaccounts
kubectl get sa   # 축약형

# default SA 상세 확인
kubectl describe sa default
```

출력 예시:
```
Name:                default
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              <none>
```

```bash
# 다른 네임스페이스의 SA 확인
kubectl get sa -n kube-system
```

!!! info
    모든 네임스페이스에는 `default` SA가 자동으로 존재합니다.
    SA를 지정하지 않은 Pod는 이 `default` SA를 사용합니다.

---

## Step 2. 새 SA 생성

### 방법 A — kubectl 명령어

```bash
kubectl create serviceaccount my-app-sa
kubectl get sa
```

### 방법 B — YAML

`sa.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
```

```bash
kubectl apply -f sa.yaml
kubectl describe sa my-app-sa
```

---

## Step 3. Pod에 SA 지정

SA를 지정하지 않은 Pod와 지정한 Pod를 비교합니다.

`pod-default-sa.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-default-sa
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command: ["sleep", "3600"]
  # serviceAccountName 미지정 → default SA 자동 사용
```

`pod-custom-sa.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-custom-sa
spec:
  serviceAccountName: my-app-sa   # 사용자 정의 SA 지정
  containers:
    - name: app
      image: busybox:1.36-musl
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f pod-default-sa.yaml
kubectl apply -f pod-custom-sa.yaml

kubectl get pods
```

=== "macOS/Linux"
    ```bash
    kubectl describe pod pod-default-sa | grep "Service Account"
    kubectl describe pod pod-custom-sa  | grep "Service Account"
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl describe pod pod-default-sa | Select-String "Service Account"
    kubectl describe pod pod-custom-sa  | Select-String "Service Account"
    ```

출력:
```
Service Account:  default
Service Account:  my-app-sa
```

---

## Step 4. Pod 안에서 마운트된 토큰 확인

```bash
# Pod 안으로 접속
kubectl exec pod-custom-sa -- sh

# 아래를 Pod 안에서 실행
ls /var/run/secrets/kubernetes.io/serviceaccount/
cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
cat /var/run/secrets/kubernetes.io/serviceaccount/token
exit
```

출력 예시:
```
ca.crt  namespace  token
default
eyJhbGciOiJSUzI1NiIsImtpZCI6Ii...  (JWT 토큰)
```

토큰을 API Server 호출에 직접 사용해봅니다:

=== "macOS/Linux"
    ```bash
    kubectl exec pod-custom-sa -- sh -c '
      TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
      CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      curl -s --cacert $CACERT \
        -H "Authorization: Bearer $TOKEN" \
        https://kubernetes.default.svc/api/v1/namespaces/default/pods
    '
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl exec pod-custom-sa -- sh -c 'TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token); CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt; curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/default/pods'
    ```

!!! info "Forbidden 응답이 정상입니다"
    `403 Forbidden`은 **인증은 성공**했지만 **권한이 없다**는 뜻입니다.
    `401 Unauthorized`라면 토큰 자체가 잘못된 것입니다.
    권한 부여는 다음 실습(02)에서 합니다.

---

## 정리

```bash
kubectl delete pod pod-default-sa pod-custom-sa
kubectl delete sa my-app-sa
```

---

## 핵심 정리

| 항목 | 내용 |
|------|------|
| SA 기본값 | 모든 Namespace에 `default` SA 자동 생성 |
| SA 미지정 시 | `default` SA 자동 연결 |
| 토큰 위치 | `/var/run/secrets/kubernetes.io/serviceaccount/token` |
| 토큰 용도 | Pod → API Server 인증 |
| 403 vs 401 | 403=인증OK/권한없음, 401=인증실패 |
