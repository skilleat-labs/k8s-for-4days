# 2-3. ConfigMap & Secret 실습

## 실습 목표

- 환경변수를 코드에 직접 박는 방식의 문제를 체감한다.
- ConfigMap으로 비민감 설정을 분리 관리한다.
- Secret으로 민감 정보를 안전하게 주입한다.
- `envFrom`, `env.valueFrom`, 볼륨 마운트 3가지 주입 방식을 모두 실습한다.
- ConfigMap 변경이 실행 중인 Pod에 어떻게 반영되는지 확인한다.

---

## 왜 ConfigMap과 Secret이 필요한가?

### 환경변수를 YAML에 직접 넣는 방식

`pod-hardcoded.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hardcoded
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo DB=$DB_HOST PASS=$DB_PASSWORD && sleep 3600"]
      env:
        - name: DB_HOST
          value: "mysql-svc"
        - name: DB_PASSWORD
          value: "supersecret123"
  restartPolicy: Never
```

```bash
kubectl apply -f pod-hardcoded.yaml
kubectl logs pod-hardcoded
```

출력:

```
DB=mysql-svc PASS=supersecret123
```

**이 방식의 문제점:**

- 비밀번호가 YAML 파일에 평문으로 노출됨
- 환경(개발/운영)이 바뀔 때마다 YAML을 수정하고 재배포해야 함
- Git에 올리면 비밀번호가 히스토리에 영구히 남음

```bash
kubectl delete pod pod-hardcoded
```

---

## 1) ConfigMap — 비민감 설정 분리

DB 접속 주소, 포트, 환경 이름처럼 **노출돼도 괜찮은 설정값**을 별도 리소스로 분리합니다.

### 방법 A — kubectl 명령어로 생성

```bash
kubectl create configmap app-config \
  --from-literal=DB_HOST=mysql-svc \
  --from-literal=DB_PORT=3306 \
  --from-literal=APP_ENV=development
```

```bash
kubectl get configmaps
kubectl describe configmap app-config
```

### 방법 B — YAML 파일로 생성

`configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: mysql-svc
  DB_PORT: "3306"
  APP_ENV: development
  app.properties: |
    log.level=INFO
    max.connections=100
    timeout.seconds=30
```

```bash
kubectl apply -f configmap.yaml
kubectl get configmap app-config -o yaml
```

---

## 2) Secret — 민감 정보 분리

비밀번호, API 키, 인증서처럼 **노출되면 안 되는 값**을 저장합니다. 내부적으로 base64 인코딩되어 저장됩니다.

!!! warning "base64는 암호화가 아닙니다"
    누구나 디코딩할 수 있습니다. Secret의 보안은 RBAC(접근 권한 제어)으로 확보합니다.

### base64 인코딩 이해

=== "macOS/Linux"
    ```bash
    echo -n "supersecret123" | base64           # 인코딩: c3VwZXJzZWNyZXQxMjM=
    echo -n "c3VwZXJzZWNyZXQxMjM=" | base64 -d # 디코딩: supersecret123
    ```
=== "Windows PowerShell"
    ```powershell
    # 인코딩
    [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("supersecret123"))

    # 디코딩
    [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String("c3VwZXJzZWNyZXQxMjM="))
    ```

### 방법 A — kubectl 명령어 (자동 base64 인코딩)

```bash
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=supersecret123 \
  --from-literal=API_KEY=abc-def-ghi
```

```bash
kubectl get secrets
kubectl describe secret app-secret    # Values는 보이지 않음
```

### 방법 B — YAML 파일로 생성

`secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQxMjM=
  API_KEY: YWJjLWRlZi1naGk=
```

### 방법 C — stringData 사용 (평문 작성, 자동 인코딩)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_PASSWORD: supersecret123
  API_KEY: abc-def-ghi
```

### Secret 값 확인

=== "macOS/Linux"
    ```bash
    kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
    ```
=== "Windows PowerShell"
    ```powershell
    $encoded = kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}'
    [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($encoded))
    ```

---

## 3) 주입 방식 1 — envFrom (전체 주입)

ConfigMap이나 Secret의 **모든 키를 한번에** 환경변수로 주입합니다.

`pod-envfrom.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-envfrom
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env | sort && sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secret
  restartPolicy: Never
```

```bash
kubectl apply -f pod-envfrom.yaml
kubectl logs pod-envfrom
```

=== "macOS/Linux"
    ```bash
    kubectl exec pod-envfrom -- env | grep -E "DB_|APP_|API_"
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl exec pod-envfrom -- env | Select-String "DB_|APP_|API_"
    ```

!!! warning "환경변수 방식의 보안 한계"
    Secret을 환경변수로 주입하면 `kubectl exec`로 접속한 누구든 `env` 명령어로 값을 평문으로 읽을 수 있습니다.
    민감한 값은 볼륨 마운트 방식이나 외부 시크릿 관리 도구(Vault, Azure Key Vault) 사용을 권장합니다.

---

## 4) 주입 방식 2 — env.valueFrom (선택 주입)

여러 ConfigMap·Secret 중 **특정 키만 골라서** 원하는 환경변수 이름으로 주입합니다.

`pod-valuefrom.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-valuefrom
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo HOST=$DB_HOST PORT=$DB_PORT PASS=$DB_PASSWORD && sleep 3600"]
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_PORT
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
  restartPolicy: Never
```

```bash
kubectl apply -f pod-valuefrom.yaml
kubectl logs pod-valuefrom
```

출력:

```
HOST=mysql-svc PORT=3306 PASS=supersecret123
```

---

## 5) 주입 방식 3 — volumeMount (파일로 마운트)

설정 파일 전체를 컨테이너 안의 경로에 **파일로 마운트**합니다.

`pod-volume.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-volume
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "cat /config/app.properties && sleep 3600"]
      volumeMounts:
        - name: config-volume
          mountPath: /config
  volumes:
    - name: config-volume
      configMap:
        name: app-config
        items:
          - key: app.properties
            path: app.properties
  restartPolicy: Never
```

```bash
kubectl apply -f pod-volume.yaml
kubectl logs pod-volume
```

출력:

```
log.level=INFO
max.connections=100
timeout.seconds=30
```

```bash
kubectl exec pod-volume -- ls /config
kubectl exec pod-volume -- cat /config/app.properties
```

---

## 6) ConfigMap 변경 — 반영 방식 차이

```bash
kubectl edit configmap app-config
# APP_ENV: development → APP_ENV: production 으로 변경 후 저장 (:wq)
```

### 환경변수 방식 — Pod 재시작 전까지 반영 안됨

=== "macOS/Linux"
    ```bash
    kubectl exec pod-envfrom -- env | grep APP_ENV
    # APP_ENV=development  (변경 전 값 그대로)
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl exec pod-envfrom -- env | Select-String "APP_ENV"
    # APP_ENV=development  (변경 전 값 그대로)
    ```

```bash
kubectl delete pod pod-envfrom
kubectl apply -f pod-envfrom.yaml
```

=== "macOS/Linux"
    ```bash
    kubectl logs pod-envfrom | grep APP_ENV
    # APP_ENV=production  (변경 후 값 반영됨)
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl logs pod-envfrom | Select-String "APP_ENV"
    # APP_ENV=production  (변경 후 값 반영됨)
    ```

### 볼륨 마운트 방식 — 약 1분 내 자동 갱신

```bash
kubectl exec pod-volume -- cat /config/app.properties
# 1분 내외로 기다렸다가 다시 실행
kubectl exec pod-volume -- cat /config/app.properties
# 값이 갱신되어 있음
```

!!! tip
    볼륨 마운트 방식은 Pod 재시작 없이 설정을 핫리로드할 수 있어 운영 환경에서 유용합니다.

---

## 7) 존재하지 않는 ConfigMap 참조 — 에러 확인

`pod-error.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-error
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      envFrom:
        - configMapRef:
            name: does-not-exist
  restartPolicy: Never
```

```bash
kubectl apply -f pod-error.yaml
kubectl get pod pod-error
kubectl describe pod pod-error    # Events에서 오류 메시지 확인
```

`CreateContainerConfigError: configmap "does-not-exist" not found` 메시지를 확인합니다.

```bash
kubectl delete pod pod-error
```

---

## 정리 (리소스 삭제)

```bash
kubectl delete pod pod-envfrom pod-valuefrom pod-volume
kubectl delete configmap app-config
kubectl delete secret app-secret
```

---

## 주입 방식 비교

| 방식 | 사용 상황 | 환경변수 이름 변경 | 동적 갱신 |
|------|---------|-----------------|---------|
| `envFrom` | 전체 키를 한번에 환경변수로 | 불가 | Pod 재시작 필요 |
| `env.valueFrom` | 특정 키만 골라서 환경변수로 | 가능 | Pod 재시작 필요 |
| `volumeMount` | 설정 파일 형태로 마운트 | — | 자동 갱신 (약 1분) |

---

## ConfigMap vs Secret 비교

| 항목 | ConfigMap | Secret |
|------|-----------|--------|
| 용도 | 비민감 설정 (DB 주소, 포트, 환경 이름) | 민감 정보 (비밀번호, API 키, 인증서) |
| 저장 방식 | 평문 | base64 인코딩 |
| `kubectl describe` | 값이 그대로 보임 | 값이 숨겨짐 |
| Git 관리 | 가능 | 직접 커밋 비권장 |

---

## 트러블슈팅

| 증상 | 확인 사항 |
|------|---------|
| Pod `CreateContainerConfigError` | ConfigMap·Secret 이름 오타 또는 아직 생성 전 |
| 환경변수 값이 비어있음 | `key` 이름이 ConfigMap의 실제 키와 일치하는지 확인 |
| base64 디코딩 오류 | `echo -n` 옵션 누락 여부 확인 |
| ConfigMap 변경이 반영 안됨 | 환경변수 방식은 Pod 재시작 필요, 볼륨 방식은 1분 대기 |
