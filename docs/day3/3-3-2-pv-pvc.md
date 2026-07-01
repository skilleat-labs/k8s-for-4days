# 볼륨 실습 — PersistentVolume / PVC

## 실습 목표

- PersistentVolume(PV)과 PersistentVolumeClaim(PVC)을 생성하고 바인딩한다.
- Pod에서 PVC를 볼륨으로 마운트하여 데이터를 영속적으로 저장한다.
- Pod를 삭제하고 재생성해도 데이터가 유지되는 것을 확인한다.

---

## 1) StorageClass 확인

클러스터에서 사용 가능한 StorageClass를 확인합니다.

```bash
kubectl get storageclass
```

환경에 따라 출력이 다릅니다.

=== "AKS"

    ```
    NAME                    PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
    azurefile               file.csi.azure.com   Delete          Immediate              true                   Xd
    azurefile-csi           file.csi.azure.com   Delete          Immediate              true                   Xd
    azurefile-csi-premium   file.csi.azure.com   Delete          Immediate              true                   Xd
    default (default)       disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   Xd
    managed                 disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   Xd
    managed-csi             disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   Xd
    managed-csi-premium     disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   Xd
    ```

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

!!! info "AKS StorageClass 종류"
    | StorageClass | 백엔드 | 특징 |
    |---|---|---|
    | `managed-csi` | Azure Disk (Standard SSD) | 단일 노드 RWO, DB에 적합 |
    | `managed-csi-premium` | Azure Disk (Premium SSD) | 고성능, 운영 DB용 |
    | `azurefile-csi` | Azure Files (SMB) | 여러 노드 RWX 지원, 공유 파일용 |

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

## 3) 방법 B — 동적 프로비저닝 (Azure Disk 자동 생성)

AKS에서는 PVC만 선언하면 Azure Disk가 자동으로 생성되고 PV로 연결됩니다. PV를 수동으로 만들 필요가 없습니다.

`pvc-dynamic.yaml` 파일을 만듭니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-csi    # Azure Disk (Standard SSD)
  resources:
    requests:
      storage: 5Gi
```

```bash
kubectl apply -f pvc-dynamic.yaml
kubectl get pvc dynamic-pvc
kubectl get pv    # PV(Azure Disk)가 자동 생성됨
```

PVC가 바인딩되면 아래와 같이 출력됩니다:

```
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
dynamic-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWO            managed-csi    30s
```

!!! warning "AKS — PVC가 Pending 상태로 머무는 경우"
    `managed-csi`는 `WaitForFirstConsumer` 모드입니다.
    **PVC만 생성한 시점에서는 Azure Disk가 만들어지지 않으며, PVC 상태가 `Pending`으로 유지되는 것이 정상입니다.**
    다음 단계(4)에서 Pod를 배포하면 Pod가 스케줄된 노드 위치에 맞춰 Azure Disk가 생성되고 PVC가 `Bound`로 바뀝니다.

!!! tip "Azure Portal에서 확인"
    PVC가 Bound되면 Azure Portal → 리소스 그룹 → **MC_** 로 시작하는 노드 리소스 그룹 안에
    `kubernetes-dynamic-pvc-xxx` 형태의 **Managed Disk**가 자동 생성된 것을 확인할 수 있습니다.

### Azure Files PVC (RWX — 여러 Pod가 동시에 마운트)

Azure Disk는 하나의 노드에서만 마운트할 수 있습니다(RWO). 여러 Pod가 동시에 같은 볼륨을 공유해야 한다면 **Azure Files**를 사용합니다.

`pvc-files.yaml` 파일을 만듭니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: files-pvc
spec:
  accessModes:
    - ReadWriteMany          # 여러 노드/Pod에서 동시 마운트 가능
  storageClassName: azurefile-csi  # Azure Files (SMB)
  resources:
    requests:
      storage: 5Gi
```

```bash
kubectl apply -f pvc-files.yaml
kubectl get pvc files-pvc
```

Azure Files는 `Immediate` 모드이므로 Pod 없이도 PVC가 즉시 `Bound`로 바뀝니다:

```
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
files-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWX            azurefile-csi   10s
```

!!! info "Azure Disk vs Azure Files 선택 기준"
    | 항목 | Azure Disk (`managed-csi`) | Azure Files (`azurefile-csi`) |
    |---|---|---|
    | 접근 모드 | RWO (단일 노드) | RWX (여러 노드 동시) |
    | 바인딩 시점 | WaitForFirstConsumer (Pod 배포 후) | Immediate (PVC 생성 즉시) |
    | 성능 | 고성능 (로컬 디스크 수준) | 상대적으로 낮음 (네트워크 파일시스템) |
    | 적합한 워크로드 | DB, 단일 인스턴스 앱 | 로그 집계, 공유 설정 파일, 멀티 레플리카 앱 |

!!! tip "Azure Portal에서 확인"
    PVC가 Bound되면 Azure Portal → 스토리지 계정 안에 **파일 공유(File Share)** 가 자동 생성됩니다.
    스토리지 계정은 `MC_` 리소스 그룹 안에서 확인할 수 있습니다.

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

!!! note "이 단계에서 dynamic-pvc는 아직 Pending"
    이 Pod는 `local-pvc`를 사용합니다. `dynamic-pvc`는 해당 PVC를 사용하는 Pod가 실제로 배포되어야 Azure Disk가 생성되고 PVC가 `Bound`로 바뀝니다.

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

MySQL은 `/var/lib/mysql`이 **빈 디렉토리**여야 초기화됩니다. 앞 단계에서 데이터가 기록된 `local-pvc`를 재사용하면 실패하므로, 3단계에서 생성한 `dynamic-pvc`(Azure Disk, 빈 상태)를 사용합니다.

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
            claimName: dynamic-pvc   # 3단계에서 생성한 Azure Disk PVC
```

```bash
kubectl apply -f deploy-with-pvc.yaml
kubectl get pods -l app=mysql-storage
kubectl get pvc dynamic-pvc   # 이 시점에서 Pending → Bound로 전환됨
```

Pod가 정상 기동되면 Azure Disk가 해당 노드에 마운트되고 MySQL 데이터가 영속적으로 저장됩니다.
Pod가 재시작되거나 다른 노드로 이동해도 동일한 Azure Disk가 자동으로 재연결됩니다.

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

---

## AKS 실습 후 리소스 일괄 정리

AKS 환경에서 실습이 끝난 뒤 생성한 모든 리소스를 한 번에 삭제합니다.
`--wait=false` 옵션을 사용하면 삭제 완료를 기다리지 않고 백그라운드로 진행됩니다.

=== "macOS/Linux"
    ```bash
    # 실습 리소스 일괄 삭제 (백그라운드)
    kubectl delete pod data-pod data-pod-2 --ignore-not-found --wait=false
    kubectl delete deployment mysql-with-storage --ignore-not-found --wait=false
    kubectl delete pvc local-pvc dynamic-pvc --ignore-not-found --wait=false
    kubectl delete pv local-pv --ignore-not-found --wait=false

    # 삭제 진행 상태 확인
    kubectl get pods,pvc,pv
    ```
=== "Windows PowerShell"
    ```powershell
    # 실습 리소스 일괄 삭제 (백그라운드)
    kubectl delete pod data-pod data-pod-2 --ignore-not-found --wait=false
    kubectl delete deployment mysql-with-storage --ignore-not-found --wait=false
    kubectl delete pvc local-pvc dynamic-pvc --ignore-not-found --wait=false
    kubectl delete pv local-pv --ignore-not-found --wait=false

    # 삭제 진행 상태 확인
    kubectl get pods,pvc,pv
    ```

!!! info "AKS에서 PVC 삭제 시 Azure Disk도 함께 삭제됨"
    AKS의 기본 StorageClass(`managed-csi`)는 Reclaim Policy가 `Delete`입니다.
    PVC를 삭제하면 연결된 **Azure Disk(PV)도 자동으로 삭제**됩니다. 별도로 Azure Portal에서 디스크를 정리할 필요가 없습니다.

    PV가 `Released` 상태로 남아있는 경우 (Retain Policy) 수동 삭제가 필요합니다:

    ```bash
    kubectl delete pv $(kubectl get pv --no-headers | awk '{print $1}')
    ```

!!! tip "네임스페이스 전체 삭제로 한 번에 정리"
    실습을 별도 네임스페이스에서 진행했다면 네임스페이스를 통째로 삭제하는 것이 가장 빠릅니다:

    ```bash
    kubectl delete namespace <실습-네임스페이스> --wait=false
    ```

---

## 도전 과제 정답 — Azure Files PVC 볼륨 마운트

### 템플릿 생성 명령어

```bash
kubectl create deployment test --replicas 2 --image nginx --dry-run=client -o yaml > filepvc-test-deploy.yaml
```

### 정답 YAML

`filepvc-test-deploy.yaml` 에 `volumeMounts`와 `volumes` 블록을 추가합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: test
  name: test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: test
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
        volumeMounts:                  # 추가
        - name: shared-storage         # 추가
          mountPath: /usr/share/nginx/html  # 추가
      volumes:                         # 추가
      - name: shared-storage           # 추가
        persistentVolumeClaim:         # 추가
          claimName: files-pvc         # 추가
status: {}
```

```bash
kubectl apply -f filepvc-test-deploy.yaml
kubectl get pods -l app=test
kubectl get pvc files-pvc   # Bound 확인
```

> `replicas: 2`이므로 Pod 2개가 모두 같은 `files-pvc` (Azure Files)를 동시에 마운트합니다.
> Azure Disk(`managed-csi`)였다면 RWO 제한으로 두 번째 Pod가 `Pending` 상태에 빠집니다.
