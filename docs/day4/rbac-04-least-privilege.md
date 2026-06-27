# RBAC 04 — 최소 권한 설계

## 실습 목표

- 와일드카드(`*`) 권한의 위험성을 직접 확인한다.
- `auth can-i --list`로 현재 권한을 한눈에 파악한다.
- 필요한 verb/resource만 남겨 권한을 축소한다.
- `resourceNames`로 특정 리소스 인스턴스만 허용한다.

---

## 개념

### 최소 권한 원칙 (Principle of Least Privilege)

> "필요한 것만, 필요한 범위만"

앱이 Pod만 읽으면 된다면 `pods/get,list`만 줘야 합니다.
`*`(와일드카드)로 전체 권한을 주면 의도치 않은 동작이나 보안 사고로 이어집니다.

---

## 사전 준비

```bash
kubectl create namespace least-priv
kubectl create serviceaccount app-sa -n least-priv
```

---

## Step 1. 와일드카드 권한 — 위험성 확인

`role-wildcard.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: wildcard-role
  namespace: least-priv
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

```bash
kubectl apply -f role-wildcard.yaml

kubectl create rolebinding wildcard-binding \
  --role=wildcard-role \
  --serviceaccount=least-priv:app-sa \
  -n least-priv
```

---

## Step 2. `auth can-i --list` 로 전체 권한 확인

```bash
kubectl auth can-i --list \
  --as=system:serviceaccount:least-priv:app-sa \
  -n least-priv
```

출력 예시:
```
Resources                                       Non-Resource URLs   Resource Names   Verbs
*.*                                             []                  []               [*]
...
```

`*.*`에 `[*]` Verbs — **모든 리소스에 모든 동작이 허용**된 상태입니다.

```bash
# 삭제도 된다
kubectl auth can-i delete secrets \
  --as=system:serviceaccount:least-priv:app-sa \
  -n least-priv
# yes ← 위험!

# 다른 Role 수정도 된다
kubectl auth can-i update roles \
  --as=system:serviceaccount:least-priv:app-sa \
  -n least-priv
# yes ← 매우 위험!
```

!!! danger "와일드카드 권한의 위험성"
    `*` 권한을 가진 SA가 탈취되면 해당 Namespace의 모든 리소스를 마음대로 조작할 수 있습니다.
    Secret 탈취, Role 수정, Pod 삭제 등 무엇이든 가능합니다.

```bash
# 와일드카드 Role 제거
kubectl delete rolebinding wildcard-binding -n least-priv
kubectl delete role wildcard-role -n least-priv
```

---

## Step 3. 필요한 권한만 남기기

앱이 실제로 필요한 것: `pods`를 `get/list`, `configmaps`를 `get`

`role-minimal.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: minimal-role
  namespace: least-priv
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
```

```bash
kubectl apply -f role-minimal.yaml

kubectl create rolebinding minimal-binding \
  --role=minimal-role \
  --serviceaccount=least-priv:app-sa \
  -n least-priv
```

권한 확인:

```bash
kubectl auth can-i --list \
  --as=system:serviceaccount:least-priv:app-sa \
  -n least-priv
```

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:least-priv:app-sa \
  -n least-priv
# yes

kubectl auth can-i delete pods \
  --as=system:serviceaccount:least-priv:app-sa \
  -n least-priv
# no

kubectl auth can-i delete secrets \
  --as=system:serviceaccount:least-priv:app-sa \
  -n least-priv
# no
```

---

## Step 4. `resourceNames` — 특정 리소스만 허용

특정 ConfigMap **하나**만 읽을 수 있도록 더 좁힐 수 있습니다.

```bash
# 테스트용 ConfigMap 2개 생성
kubectl create configmap allowed-config --from-literal=key=value -n least-priv
kubectl create configmap secret-config --from-literal=key=secret -n least-priv
```

`role-resourcenames.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: specific-config-reader
  namespace: least-priv
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
    resourceNames: ["allowed-config"]   # 이 ConfigMap만 허용
```

```bash
# 기존 binding 제거 후 새 Role로 교체
kubectl delete rolebinding minimal-binding -n least-priv

kubectl apply -f role-resourcenames.yaml

kubectl create rolebinding specific-binding \
  --role=specific-config-reader \
  --serviceaccount=least-priv:app-sa \
  -n least-priv
```

---

## Step 5. 허용/차단 조합 확인

```bash
# allowed-config 조회 → 허용
kubectl auth can-i get configmap/allowed-config \
  --as=system:serviceaccount:least-priv:app-sa \
  -n least-priv
# yes

# secret-config 조회 → 차단
kubectl auth can-i get configmap/secret-config \
  --as=system:serviceaccount:least-priv:app-sa \
  -n least-priv
# no

# list (전체 목록) → 차단 (resourceNames가 있으면 list/watch 불가)
kubectl auth can-i list configmaps \
  --as=system:serviceaccount:least-priv:app-sa \
  -n least-priv
# no
```

!!! warning "resourceNames + list/watch"
    `resourceNames`를 지정하면 `list`, `watch`는 자동으로 차단됩니다.
    특정 리소스 하나를 `get`만 하게 할 때 유용합니다.

---

## 정리

```bash
kubectl delete namespace least-priv
```

---

## 설계 체크리스트

```
[ ] verbs에 "*" 사용하지 않았는가?
[ ] resources에 "*" 사용하지 않았는가?
[ ] 실제로 필요한 verb만 나열했는가? (get이면 get만, list 필요없으면 제거)
[ ] 특정 리소스 인스턴스만 필요하다면 resourceNames 사용했는가?
[ ] ClusterRole 대신 Role로 Namespace 범위로 제한할 수 있는가?
[ ] auth can-i --list 로 의도치 않은 권한 없는지 최종 확인했는가?
```

---

## 권한 설계 비교

| 상황 | 잘못된 설계 | 올바른 설계 |
|------|-----------|-----------|
| Pod 목록만 필요 | `resources: ["*"]` | `resources: ["pods"]` |
| 읽기만 필요 | `verbs: ["*"]` | `verbs: ["get", "list", "watch"]` |
| 특정 ConfigMap만 | `resources: ["configmaps"]` | `resources: ["configmaps"]` + `resourceNames: ["app-config"]` |
| 같은 NS만 | ClusterRoleBinding | RoleBinding |
