# RBAC 02 — Role & RoleBinding 기초

## 실습 목표

- RBAC의 Role / RoleBinding 구조를 이해한다.
- 권한 없는 SA로 요청이 거부되는 것을 확인한다.
- Role을 만들고 RoleBinding으로 SA에 연결한다.
- `auth can-i`와 `--as` 옵션으로 권한을 검증한다.

---

## 개념

### RBAC 구조

```
Who?      What?     Where?
SA      →  Role   →  Namespace
         (verbs + resources)

연결고리: RoleBinding
```

- **Role**: "무엇을 할 수 있나" — verbs(동작) + resources(대상) 정의
- **RoleBinding**: "누가 이 Role을 가지나" — SA와 Role 연결

### 주요 verbs

| verb | HTTP 동작 |
|------|----------|
| `get` | 단건 조회 |
| `list` | 목록 조회 |
| `watch` | 실시간 감시 |
| `create` | 생성 |
| `update` | 전체 수정 |
| `patch` | 부분 수정 |
| `delete` | 삭제 |

---

## 사전 준비

```bash
# 실습용 네임스페이스 생성
kubectl create namespace rbac-test

# SA 생성
kubectl create serviceaccount pod-reader -n rbac-test
```

---

## Step 1. 권한 없는 SA로 요청 → 거부 확인

`auth can-i`로 특정 SA가 특정 동작을 할 수 있는지 확인합니다.

```bash
# pod-reader SA가 pods를 get할 수 있는가?
kubectl auth can-i get pods \
  --as=system:serviceaccount:rbac-test:pod-reader \
  -n rbac-test
```

출력:
```
no
```

```bash
# 여러 동작 한번에 확인
kubectl auth can-i list pods \
  --as=system:serviceaccount:rbac-test:pod-reader \
  -n rbac-test

kubectl auth can-i create pods \
  --as=system:serviceaccount:rbac-test:pod-reader \
  -n rbac-test
```

모두 `no`가 나오는 것을 확인합니다.

!!! info "--as 형식"
    SA를 사칭할 때는 `system:serviceaccount:<namespace>:<sa이름>` 형식을 씁니다.

---

## Step 2. Role 생성

`role-pod-reader.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: rbac-test
rules:
  - apiGroups: [""]       # core API group (Pod, Service 등)
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

```bash
kubectl apply -f role-pod-reader.yaml
kubectl get role -n rbac-test
kubectl describe role pod-reader -n rbac-test
```

출력:
```
Name:         pod-reader
Labels:       <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  pods       []                 []              [get list watch]
```

---

## Step 3. RoleBinding으로 SA와 Role 연결

`rolebinding-pod-reader.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: rbac-test
subjects:
  - kind: ServiceAccount
    name: pod-reader
    namespace: rbac-test
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rolebinding-pod-reader.yaml
kubectl get rolebinding -n rbac-test
kubectl describe rolebinding pod-reader-binding -n rbac-test
```

출력:
```
Name:         pod-reader-binding
Subjects:
  Kind            Name        Namespace
  ----            ----        ---------
  ServiceAccount  pod-reader  rbac-test
RoleRef:
  Kind:  Role
  Name:  pod-reader
```

---

## Step 4. 권한 부여 후 재확인

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:rbac-test:pod-reader \
  -n rbac-test
```

출력:
```
yes
```

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:rbac-test:pod-reader \
  -n rbac-test
# yes

kubectl auth can-i delete pods \
  --as=system:serviceaccount:rbac-test:pod-reader \
  -n rbac-test
# no  ← delete는 Role에 없으므로 여전히 거부
```

---

## Step 5. SA를 실제로 사용하는 Pod에서 검증

`pod-rbac-test.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-rbac-test
  namespace: rbac-test
spec:
  serviceAccountName: pod-reader
  containers:
    - name: app
      image: bitnami/kubectl:latest
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f pod-rbac-test.yaml
kubectl get pod -n rbac-test
```

Pod 안에서 직접 API 호출:

```bash
kubectl exec -n rbac-test pod-rbac-test -- \
  kubectl get pods -n rbac-test
# 성공 — pods 조회 허용

kubectl exec -n rbac-test pod-rbac-test -- \
  kubectl delete pod pod-rbac-test -n rbac-test
# Error — delete 권한 없음
```

---

## 정리

```bash
kubectl delete namespace rbac-test
```

---

## 핵심 정리

```
ServiceAccount  →  RoleBinding  →  Role
(who)              (연결)          (what: verbs + resources)
```

| 항목 | 설명 |
|------|------|
| `apiGroups: [""]` | core API group — Pod, Service, ConfigMap 등 |
| `apiGroups: ["apps"]` | apps group — Deployment, ReplicaSet 등 |
| `auth can-i` | 권한 검증 명령어 |
| `--as` | 다른 SA/User로 사칭해서 권한 확인 |
| RoleBinding scope | 해당 **Namespace 안에서만** 유효 |
