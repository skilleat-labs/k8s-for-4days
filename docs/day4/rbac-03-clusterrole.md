# RBAC 03 — ClusterRole & ClusterRoleBinding

## 실습 목표

- Role vs ClusterRole, RoleBinding vs ClusterRoleBinding 차이를 이해한다.
- 클러스터 범위 리소스(nodes, pv 등)에 접근하는 ClusterRole을 만든다.
- 4가지 조합의 차이를 직접 확인한다.

---

## 개념

### Role vs ClusterRole

| 항목 | Role | ClusterRole |
|------|------|------------|
| 범위 | 특정 Namespace | 클러스터 전체 |
| 적용 리소스 | Namespace 리소스 (Pod, Service 등) | 클러스터 리소스 (Node, PV 등) + 모든 NS 리소스 |
| 생성 위치 | Namespace 안 | 클러스터 레벨 (NS 없음) |

클러스터 범위 리소스는 Role로 접근 권한을 줄 수 없습니다.

```bash
# 클러스터 범위 리소스 목록 확인
kubectl api-resources --namespaced=false
# nodes, persistentvolumes, clusterroles, namespaces 등...
```

### 4가지 조합

```
ClusterRole  +  ClusterRoleBinding  →  클러스터 전체 범위 권한
ClusterRole  +  RoleBinding         →  특정 Namespace로 범위 제한
Role         +  RoleBinding         →  특정 Namespace 범위 권한 (기본)
Role         +  ClusterRoleBinding  →  사용 불가 (오류)
```

---

## 사전 준비

```bash
kubectl create namespace ns-a
kubectl create namespace ns-b
kubectl create serviceaccount cluster-reader -n ns-a
```

---

## Step 1. ClusterRole 생성

노드와 PV(클러스터 범위 리소스)를 조회하는 ClusterRole을 만듭니다.

`clusterrole-reader.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-resource-reader
rules:
  - apiGroups: [""]
    resources: ["nodes", "persistentvolumes", "namespaces"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

```bash
kubectl apply -f clusterrole-reader.yaml
kubectl get clusterrole cluster-resource-reader
kubectl describe clusterrole cluster-resource-reader
```

---

## Step 2. ClusterRoleBinding으로 연결 → 클러스터 전체 권한

`clusterrolebinding-reader.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-resource-reader-binding
subjects:
  - kind: ServiceAccount
    name: cluster-reader
    namespace: ns-a
roleRef:
  kind: ClusterRole
  name: cluster-resource-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f clusterrolebinding-reader.yaml
```

---

## Step 3. 클러스터 범위 권한 확인

```bash
# 노드 조회 (클러스터 리소스)
kubectl auth can-i list nodes \
  --as=system:serviceaccount:ns-a:cluster-reader
# yes

# PV 조회
kubectl auth can-i list persistentvolumes \
  --as=system:serviceaccount:ns-a:cluster-reader
# yes

# ns-b의 Pod 조회 (다른 NS)
kubectl auth can-i list pods \
  --as=system:serviceaccount:ns-a:cluster-reader \
  -n ns-b
# yes ← 클러스터 전체 범위이므로 다른 NS도 가능
```

---

## Step 4. ClusterRole + RoleBinding → Namespace 범위로 제한

같은 ClusterRole이지만 이번엔 **RoleBinding**으로 연결하면 `ns-a`에서만 유효합니다.

```bash
# 기존 ClusterRoleBinding 제거
kubectl delete clusterrolebinding cluster-resource-reader-binding
```

`rolebinding-cluster-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-role-ns-binding
  namespace: ns-a          # ns-a에만 적용
subjects:
  - kind: ServiceAccount
    name: cluster-reader
    namespace: ns-a
roleRef:
  kind: ClusterRole        # ClusterRole 참조 가능
  name: cluster-resource-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rolebinding-cluster-role.yaml
```

권한 재확인:

```bash
# ns-a의 Pod 조회
kubectl auth can-i list pods \
  --as=system:serviceaccount:ns-a:cluster-reader \
  -n ns-a
# yes ← ns-a는 가능

# ns-b의 Pod 조회
kubectl auth can-i list pods \
  --as=system:serviceaccount:ns-a:cluster-reader \
  -n ns-b
# no ← RoleBinding이라서 ns-a로 제한됨

# 노드 조회 (클러스터 리소스 — NS 무관)
kubectl auth can-i list nodes \
  --as=system:serviceaccount:ns-a:cluster-reader
# no ← RoleBinding은 클러스터 리소스에 효력 없음
```

!!! info "핵심 차이"
    ClusterRole을 **ClusterRoleBinding**으로 연결 → 클러스터 전체 적용
    ClusterRole을 **RoleBinding**으로 연결 → 해당 Namespace로 제한

---

## Step 5. 내장 ClusterRole 활용

쿠버네티스에는 기본 제공 ClusterRole이 있습니다. 직접 만들지 않고 재사용할 수 있습니다.

=== "macOS/Linux"
    ```bash
    kubectl get clusterrole | grep -E "^view|^edit|^admin|^cluster-admin"
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl get clusterrole | Select-String "^view|^edit|^admin|^cluster-admin"
    ```

| 내장 ClusterRole | 설명 |
|-----------------|------|
| `view` | 대부분 리소스 읽기 전용 |
| `edit` | 대부분 리소스 읽기/쓰기 (Role 수정 불가) |
| `admin` | Namespace 내 거의 모든 권한 |
| `cluster-admin` | 클러스터 전체 모든 권한 |

```bash
# 내장 view ClusterRole을 특정 NS에 적용
kubectl create rolebinding ns-a-view \
  --clusterrole=view \
  --serviceaccount=ns-a:cluster-reader \
  -n ns-a

kubectl auth can-i list pods \
  --as=system:serviceaccount:ns-a:cluster-reader \
  -n ns-a
# yes
```

---

## 정리

```bash
kubectl delete namespace ns-a ns-b
kubectl delete clusterrole cluster-resource-reader
kubectl delete clusterrolebinding cluster-resource-reader-binding --ignore-not-found
```

---

## 4가지 조합 한눈에 정리

| Role 종류 | Binding 종류 | 결과 |
|----------|-------------|------|
| Role | RoleBinding | 특정 NS 안에서만 |
| ClusterRole | RoleBinding | 특정 NS 안에서만 (클러스터 리소스 제외) |
| ClusterRole | ClusterRoleBinding | 클러스터 전체 |
| Role | ClusterRoleBinding | ❌ 사용 불가 |
