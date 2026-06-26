# 볼륨 실습 — PersistentVolume / PVC

## 실습 목표

- PersistentVolume(PV)과 PersistentVolumeClaim(PVC)을 생성하고 바인딩한다.
- Pod에서 PVC를 볼륨으로 마운트하여 데이터를 영속적으로 저장한다.
- Pod를 삭제하고 재생성해도 데이터가 유지되는 것을 확인한다.

---

## 1) StorageClass 확인

로컬 클러스터에서 기본 제공되는 StorageClass를 확인합니다.

```bash
kubectl get storageclass
```

환경에 따라 출력이 다릅니다.

=== "Docker Desktop"

    ```
    NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   AGE
    hostpath (default)   docker.io/hostpath   Delete          Immediate           10m
    ```

=== "Rancher Desktop (k3s)"

    ```
    NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
    local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  63d
    ```

기본 StorageClass(`(default)` 표시)가 있으면 동적 프로비저닝을 사용할 수 있습니다.

---

## 2) 방법 A — 정적 프로비저닝 (PV + PVC 수동 생성)

### PV 생성

`pv-local.yaml` 파일을 만듭니다.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /tmp/k8s-lab-data    # 노드의 실제 경로 (Rancher Desktop은 Lima VM 내부 경로)
```

```bash
kubectl apply -f pv-local.yaml
kubectl get pv
```

> **Rancher Desktop 사용자**: `/tmp/k8s-lab-data`는 Mac 로컬이 아닌 **Lima VM 내부 경로**입니다. Finder에서는 확인할 수 없습니다.

출력:

```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS
local-pv   1Gi        RWO            Retain           Available           manual
```

> Kubernetes 1.29 이상에서는 `VOLUMEATTRIBUTESCLASS`, `REASON`, `AGE` 컬럼이 추가로 표시될 수 있습니다. 주요 값(STATUS: Available, STORAGECLASS: manual)만 확인하면 됩니다.

### PVC 생성

`pvc-local.yaml` 파일을 만듭니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: manual
```

```bash
kubectl apply -f pvc-local.yaml
kubectl get pvc
kubectl get pv    # STATUS가 Bound로 변경됨
```

출력:

```
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS
local-pvc   Bound    local-pv   1Gi        RWO            manual
```

> Kubernetes 1.29 이상에서는 `VOLUMEATTRIBUTESCLASS`, `AGE` 컬럼이 추가로 표시될 수 있습니다. `STATUS: Bound`와 `VOLUME: local-pv` 값만 확인하면 됩니다.

---

## 3) 방법 B — 동적 프로비저닝 (StorageClass 자동 PV 생성)

기본 StorageClass가 있는 경우 PV 없이 PVC만 생성합니다.

`pvc-dynamic.yaml` 파일을 만듭니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  # storageClassName 생략 시 기본 StorageClass 사용
```

```bash
kubectl apply -f pvc-dynamic.yaml
kubectl get pvc dynamic-pvc
kubectl get pv    # PV가 자동 생성됨
```

!!! warning "Rancher Desktop — PVC가 Pending 상태로 머무는 경우"
    `local-path` StorageClass는 `WaitForFirstConsumer` 모드입니다.
    **PVC만 생성한 시점에서는 PV가 만들어지지 않으며, PVC 상태가 `Pending`으로 유지되는 것이 정상입니다.**
    다음 단계(4)에서 Pod를 배포하면 그때 비로소 PV가 생성되고 PVC가 `Bound`로 바뀝니다.

---

## 4) PVC를 사용하는 Pod 배포

`pod-with-pvc.yaml` 파일을 만듭니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-pod
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command: ["sh", "-c", "echo 'Hello K8s Storage!' > /data/hello.txt && cat /data/hello.txt && sleep 3600"]
      volumeMounts:
        - name: data-storage
          mountPath: /data
  volumes:
    - name: data-storage
      persistentVolumeClaim:
        claimName: local-pvc    # 또는 dynamic-pvc
  restartPolicy: Never
```

```bash
kubectl apply -f pod-with-pvc.yaml
kubectl logs data-pod
```

```
Hello K8s Storage!
```

!!! note "Rancher Desktop — 이 단계에서 dynamic-pvc는 아직 Pending"
    이 Pod는 `local-pvc`를 사용합니다. `dynamic-pvc`는 해당 PVC를 사용하는 Pod가 실제로 배포되어야 PV가 생성되고 `Bound`로 바뀝니다.

    `dynamic-pvc`가 `Pending` 상태로 유지되는 것은 **정상**입니다. 다음 6단계에서 MySQL Deployment를 배포하면 그때 `Bound`로 전환됩니다.

    ```bash
    kubectl get pvc dynamic-pvc   # Pending — 정상, 아직 사용하는 Pod가 없음
    ```

---

## 5) 데이터 영속성 확인

Pod를 삭제하고 재생성해도 데이터가 유지되는지 확인합니다.

### Pod 삭제

```bash
kubectl delete pod data-pod
```

### 새 Pod에서 같은 PVC 재사용

`pod-with-pvc-2.yaml` 파일을 만듭니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-pod-2
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command: ["sh", "-c", "cat /data/hello.txt && sleep 3600"]
      volumeMounts:
        - name: data-storage
          mountPath: /data
  volumes:
    - name: data-storage
      persistentVolumeClaim:
        claimName: local-pvc
  restartPolicy: Never
```

```bash
kubectl apply -f pod-with-pvc-2.yaml
kubectl logs data-pod-2
```

```
Hello K8s Storage!
```

> **포인트**: Pod가 삭제되고 새 Pod가 생성되어도 `/data/hello.txt` 파일이 그대로 남아있습니다.

---

## 6) PVC를 Deployment에 사용

실제로는 Deployment에서 PVC를 사용합니다.

MySQL은 `/var/lib/mysql`이 **빈 디렉토리**여야 초기화됩니다. 앞 단계에서 데이터가 기록된 `local-pvc`를 재사용하면 실패하므로, 동적 프로비저닝으로 새 PVC를 사용합니다.

`deploy-with-pvc.yaml` 파일을 만듭니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-with-storage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql-storage
  template:
    metadata:
      labels:
        app: mysql-storage
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: testpassword
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: dynamic-pvc   # 앞 단계에서 생성한 빈 PVC 사용
```

```bash
kubectl apply -f deploy-with-pvc.yaml
kubectl get pods -l app=mysql-storage
```

---

## 접근 모드 비교

| 모드 | 설명 | 용도 |
|---|---|---|
| `ReadWriteOnce (RWO)` | 하나의 노드에서만 읽기/쓰기 | 데이터베이스 |
| `ReadOnlyMany (ROX)` | 여러 노드에서 읽기 전용 | 정적 파일 공유 |
| `ReadWriteMany (RWX)` | 여러 노드에서 읽기/쓰기 | NFS, 공유 스토리지 |

> 로컬(hostPath)은 RWO만 지원합니다. RWX는 NFS 또는 클라우드 파일 스토리지(Azure Files, EFS)가 필요합니다.

---

## 정리 (리소스 삭제)

```bash
kubectl delete pod data-pod-2
kubectl delete -f deploy-with-pvc.yaml
kubectl delete pvc local-pvc dynamic-pvc
kubectl delete pv local-pv
```

> PV `Reclaim Policy`가 `Retain`이면 PVC 삭제 후에도 PV가 `Released` 상태로 남습니다. 수동으로 삭제해야 합니다.

---

## 트러블슈팅

| 증상 | 확인 사항 |
|---|---|
| PVC `Pending` 상태 | StorageClass 이름 일치 여부, PV 용량이 충분한지 확인 |
| Pod `Pending` (PVC 관련) | PVC가 `Bound` 상태인지 확인 |
| 데이터 유실 | PV의 `Reclaim Policy`가 `Delete`이면 PVC 삭제 시 데이터 삭제됨 |
| MySQL Pod `Error` / `CrashLoopBackOff` | 마운트 경로(`/var/lib/mysql`)에 기존 파일이 있으면 초기화 실패 — 빈 PVC를 새로 생성해서 사용 |
| Rancher Desktop에서 PVC `Pending` 유지 | `local-path` StorageClass는 `WaitForFirstConsumer` 모드 — Pod를 배포해야 PV가 생성됨 |
