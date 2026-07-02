# RBAC 05 — SA 토큰 관리

## 실습 목표

- 토큰 자동 마운트의 보안 위험을 이해한다.
- SA 레벨과 Pod 레벨에서 자동 마운트를 비활성화한다.
- Projected Volume으로 만료 시간이 있는 토큰을 수동 마운트한다.

---

## 개념

### 토큰 자동 마운트의 보안 위험

Pod가 생성되면 기본적으로 SA 토큰이 **자동으로** 마운트됩니다.

```
Pod 안 컨테이너
  └── /var/run/secrets/kubernetes.io/serviceaccount/token
        ← 이 토큰으로 API Server에 접근 가능
```

앱이 API Server에 접근할 필요가 없어도 토큰이 항상 존재합니다.
컨테이너가 탈취되면 공격자가 이 토큰으로 클러스터를 공격할 수 있습니다.

### 해결책

| 방법 | 설명 |
|------|------|
| SA 레벨 비활성화 | SA에 `automountServiceAccountToken: false` |
| Pod 레벨 비활성화 | Pod spec에 `automountServiceAccountToken: false` |
| Projected Volume | 필요할 때만, 만료 시간 있는 토큰을 수동 마운트 |

Pod 레벨 설정이 SA 레벨보다 우선합니다.

---

## Step 1. 기본 상태 — 토큰 존재 확인

```bash
# 기본 SA로 Pod 배포
kubectl run pod-default --image=busybox:1.36-musl -- sleep 3600
kubectl get pod pod-default
```

```bash
# 토큰 마운트 확인
kubectl exec pod-default -- ls /var/run/secrets/kubernetes.io/serviceaccount/
```

출력:
```
ca.crt  namespace  token
```

=== "macOS/Linux"
    ```bash
    # 마운트 경로 확인
    kubectl describe pod pod-default | grep -A5 "Mounts:"
    ```
=== "Windows PowerShell"
    ```powershell
    # 마운트 경로 확인
    kubectl describe pod pod-default | Select-String -Pattern "Mounts:" -Context 0,5
    ```

출력:
```
Mounts:
  /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-xxxxx (ro)
```

```bash
kubectl delete pod pod-default
```

---

## Step 1-1. 토큰으로 API 서버 직접 호출

토큰이 실제로 API 서버 접근에 사용될 수 있음을 확인합니다.

```bash
# curl이 있는 Pod 생성
kubectl run api-test --image=curlimages/curl:latest --restart=Never --command -- sleep 3600
kubectl get pod api-test
```

Pod 안에서 토큰을 꺼내 API 서버에 직접 요청합니다:

=== "macOS/Linux"
    ```bash
    kubectl exec api-test -- sh -c '
      TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
      CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
      curl -s --cacert $CACERT \
        -H "Authorization: Bearer $TOKEN" \
        https://kubernetes.default.svc/api/v1/namespaces/$NAMESPACE/pods
    '
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl exec api-test -- sh -c 'TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token); CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt; NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace); curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/$NAMESPACE/pods'
    ```

    !!! info "PowerShell single quote 사용 이유"
        PowerShell에서 single quote(`'`)로 감싸면 `$TOKEN` 같은 변수를 PowerShell이 먼저 해석하지 않고 그대로 컨테이너 안의 sh에 전달합니다.

```bash
kubectl delete pod api-test
```

!!! warning "이것이 보안 위협인 이유"
    컨테이너가 탈취되면 공격자가 이 토큰으로 `curl` 한 줄로 클러스터 정보를 가져올 수 있습니다.
    API 접근이 필요 없는 Pod라면 토큰 자동 마운트를 반드시 비활성화해야 합니다.

---

## Step 2. SA 레벨에서 자동 마운트 비활성화

`sa-no-automount.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-token-sa
automountServiceAccountToken: false   # SA 레벨 비활성화
```

```bash
kubectl apply -f sa-no-automount.yaml
```

이 SA를 사용하는 Pod 배포:

`pod-sa-no-token.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-sa-no-token
spec:
  serviceAccountName: no-token-sa
  containers:
    - name: app
      image: busybox:1.36-musl
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f pod-sa-no-token.yaml

# 토큰이 없는지 확인
kubectl exec pod-sa-no-token -- ls /var/run/secrets/kubernetes.io/serviceaccount/ 2>&1
```

출력:
```
ls: /var/run/secrets/kubernetes.io/serviceaccount/: No such file or directory
```

!!! success "확인 포인트"
    디렉토리 자체가 존재하지 않으면 토큰 자동 마운트가 비활성화된 것입니다.

---

## Step 3. Pod 레벨에서 비활성화

SA 설정과 무관하게 **Pod 단위**로도 제어할 수 있습니다.
Pod 레벨 설정이 SA 레벨보다 **우선**합니다.

`pod-no-token.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-no-token
spec:
  serviceAccountName: default        # default SA 사용
  automountServiceAccountToken: false   # Pod 레벨 비활성화
  containers:
    - name: app
      image: busybox:1.36-musl
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f pod-no-token.yaml

kubectl exec pod-no-token -- ls /var/run/secrets/ 2>&1
# 디렉토리 없음 확인
```

우선순위 확인:

```bash
# SA는 automount: false, Pod는 true로 설정 → Pod 설정이 이김
```

`pod-override-token.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-override-token
spec:
  serviceAccountName: no-token-sa      # automount: false인 SA
  automountServiceAccountToken: true   # Pod에서 true로 덮어쓰기
  containers:
    - name: app
      image: busybox:1.36-musl
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f pod-override-token.yaml

kubectl exec pod-override-token -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# token이 존재 — Pod 레벨 설정이 SA 레벨을 덮어씀
```

---

## Step 4. Projected Volume으로 수동 마운트

토큰이 필요하긴 하지만 만료 시간을 설정해서 보안을 높이고 싶을 때 사용합니다.

`pod-projected-token.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-projected-token
spec:
  serviceAccountName: default
  automountServiceAccountToken: false   # 자동 마운트 비활성화
  containers:
    - name: app
      image: busybox:1.36-musl
      command: ["sleep", "3600"]
      volumeMounts:
        - name: token-vol
          mountPath: /var/run/secrets/tokens
          readOnly: true
  volumes:
    - name: token-vol
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600   # 1시간 후 만료 (최소 600초)
              audience: api             # 토큰 수신자 지정
          - configMap:
              name: kube-root-ca.crt
              items:
                - key: ca.crt
                  path: ca.crt
```

```bash
kubectl apply -f pod-projected-token.yaml

# 마운트 위치 확인
kubectl exec pod-projected-token -- ls /var/run/secrets/tokens/
```

출력:
```
ca.crt  token
```

```bash
# 토큰 내용 확인 (JWT)
kubectl exec pod-projected-token -- cat /var/run/secrets/tokens/token
```

!!! info "Projected Token의 장점"
    - `expirationSeconds`로 수명 제한 (기본 자동 마운트 토큰은 무기한)
    - `audience`로 토큰 사용처 제한
    - 쿠버네티스가 자동으로 갱신 (만료 80% 시점)
    - 경로를 원하는 곳으로 지정 가능

---

## Step 5. 토큰 내용 디코딩 (확인용)

=== "macOS/Linux"
    ```bash
    # 토큰 추출
    TOKEN=$(kubectl exec pod-projected-token -- cat /var/run/secrets/tokens/token)

    # JWT 구조 확인 (중간 payload 부분 디코딩)
    echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
    ```
=== "Windows PowerShell"
    ```powershell
    # 토큰 추출
    $env:TOKEN = kubectl exec pod-projected-token -- cat /var/run/secrets/tokens/token

    # JWT 구조 확인 (중간 payload 부분 디코딩)
    $payload = $env:TOKEN.Split('.')[1]
    # base64 패딩 보정 후 디코딩
    $padded = $payload + ('=' * ((4 - $payload.Length % 4) % 4))
    [System.Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($padded)) | python3 -m json.tool
    ```

출력 예시:
```json
{
  "aud": ["api"],
  "exp": 1735000000,        ← 만료 시간 (Unix timestamp)
  "iat": 1734996400,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "kubernetes.io": {
    "namespace": "default",
    "pod": { "name": "pod-projected-token", ... },
    "serviceaccount": { "name": "default", ... }
  }
}
```

---

## 정리

```bash
kubectl delete pod pod-sa-no-token pod-no-token pod-override-token pod-projected-token
kubectl delete sa no-token-sa
```

---

## SA 토큰 설정 우선순위

```
Pod spec.automountServiceAccountToken
  ↓ (Pod 설정이 있으면 이게 우선)
SA spec.automountServiceAccountToken
  ↓ (SA 설정이 없으면)
기본값: true (자동 마운트)
```

---

## 보안 설계 가이드

| 상황 | 권장 설정 |
|------|---------|
| API Server 접근 불필요한 앱 | SA 또는 Pod에 `automountServiceAccountToken: false` |
| API Server 접근 필요한 앱 | Projected Volume으로 `expirationSeconds` 설정 |
| 다수의 Pod에 일괄 적용 | SA 레벨에서 `false` 설정 후 필요한 Pod에서 `true` 덮어쓰기 |
| 프로덕션 환경 | 최소 권한 Role + Projected Volume 조합 |

---

## 트러블슈팅

| 증상 | 확인 사항 |
|------|---------|
| API 호출 시 `401 Unauthorized` | 토큰이 마운트됐는지, 경로가 맞는지 확인 |
| `403 Forbidden` | 토큰은 유효하나 RBAC 권한 없음 → Role/RoleBinding 확인 |
| Projected Token이 갱신 안됨 | `expirationSeconds` 최솟값은 600초 |
| `No such file or directory` | `automountServiceAccountToken: false` 설정 확인 |
