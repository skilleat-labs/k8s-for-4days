# StatefulSet 실습

## 실습 목표

- StatefulSet이 무엇인지, Deployment와 어떻게 다른지 이해한다.
- 안정적인 Pod 이름과 네트워크 ID를 확인한다.
- 순서가 보장된 기동·종료를 관찰한다.
- volumeClaimTemplate으로 Pod마다 독립적인 스토리지를 갖는 구조를 실습한다.
- 스케일링과 롤링 업데이트를 운영 관점에서 이해한다.

---

## StatefulSet이란?

StatefulSet은 **상태(State)가 있는 애플리케이션을 위한 컨트롤러**입니다.
Pod마다 고유한 이름, 네트워크 주소, 스토리지를 부여하고, 순서를 보장합니다.

### Deployment vs StatefulSet

| 항목 | Deployment | StatefulSet |
|---|---|---|
| Pod 이름 | `app-7d4f8c-xkr2p` (랜덤) | `app-0`, `app-1`, `app-2` (고정) |
| 네트워크 ID | 매번 바뀜 | 재시작해도 동일한 DNS 유지 |
| 스토리지 | Pod 간 공유 또는 없음 | Pod마다 독립적인 PVC |
| 기동 순서 | 동시 기동 | 0 → 1 → 2 순서 보장 |
| 종료 순서 | 임의 | 2 → 1 → 0 역순 보장 |
| 주요 용도 | 무상태 웹·API 서버 | DB, 메시지 큐, 분산 스토리지 |

### 실제 운영에서 사용하는 경우

| 워크로드 | 이유 |
|---|---|
| MySQL / PostgreSQL | 마스터-레플리카 역할 분리, 각자 독립 디스크 필요 |
| Redis Sentinel / Cluster | 노드 역할이 다르고, 재시작 후 같은 ID로 재연결 필요 |
| Kafka / Zookeeper | 브로커 ID가 고정되어야 클러스터가 유지됨 |
| Elasticsearch | 각 노드가 독립 데이터 샤드를 보관 |

---

## 1) Headless Service 생성

StatefulSet은 반드시 **Headless Service**와 함께 사용합니다.
Headless Service는 `clusterIP: None`으로 설정하여 각 Pod에 직접 DNS를 부여합니다.

```
일반 Service:  my-svc.default.svc.cluster.local → 로드밸런서 IP 하나
Headless Service: web-0.my-svc.default.svc.cluster.local → Pod-0 IP 직접
              web-1.my-svc.default.svc.cluster.local → Pod-1 IP 직접
```

`headless-svc.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  clusterIP: None        # Headless — IP를 할당하지 않음
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f headless-svc.yaml
kubectl get svc web
```

출력:

```
NAME   TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   None         <none>        80/TCP    5s
```

`CLUSTER-IP`가 `None`이면 정상입니다.

---

## 2) 기본 StatefulSet 생성

`statefulset-basic.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web       # 위에서 만든 Headless Service 이름
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f statefulset-basic.yaml
kubectl get pods -w    # -w로 생성 순서 실시간 관찰
```

Pod가 **0 → 1 → 2 순서로 하나씩** 생성되는 것을 확인합니다:

```
NAME    READY   STATUS              RESTARTS   AGE
web-0   0/1     ContainerCreating   0          2s
web-0   1/1     Running             0          4s
web-1   0/1     ContainerCreating   0          4s
web-1   1/1     Running             0          6s
web-2   0/1     ContainerCreating   0          7s
web-2   1/1     Running             0          9s
```

!!! info "순서 보장의 이유"
    StatefulSet은 이전 Pod가 `Running` 상태가 될 때까지 다음 Pod를 생성하지 않습니다.
    DB 클러스터처럼 마스터가 먼저 떠야 레플리카가 연결할 수 있는 구조를 지원하기 위해서입니다.

---

## 3) 안정적인 네트워크 ID 확인

각 Pod가 고유한 DNS 이름을 가집니다.

```bash
# 테스트용 Pod로 DNS 조회
kubectl run dns-test --image=busybox:1.36-musl --restart=Never -it --rm \
  -- nslookup web-0.web.default.svc.cluster.local
```

예상 출력:

```
Server:    10.96.0.10
Address 1: 10.96.0.10

Name:      web-0.web.default.svc.cluster.local
Address 1: 10.244.0.5
```

Pod 이름 패턴: `<StatefulSet이름>-<순번>.<Service이름>.<Namespace>.svc.cluster.local`

### Pod를 삭제해도 이름이 유지됨

```bash
kubectl delete pod web-1
kubectl get pods -w
```

`web-1`이 삭제되면 같은 이름 `web-1`으로 새 Pod가 생성됩니다. IP는 바뀌어도 DNS 이름은 동일합니다.

---

## 4) volumeClaimTemplate — Pod마다 독립 스토리지

StatefulSet의 핵심 기능입니다. `volumeClaimTemplates`를 사용하면 **Pod마다 PVC를 자동으로 생성**합니다.

`statefulset-with-pvc.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-storage
spec:
  serviceName: web
  replicas: 2
  selector:
    matchLabels:
      app: web-storage
  template:
    metadata:
      labels:
        app: web-storage
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:        # Pod마다 PVC 자동 생성
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Mi
```

```bash
kubectl apply -f statefulset-with-pvc.yaml
kubectl get pods,pvc
```

출력:

```
NAME              READY   STATUS    AGE
pod/web-storage-0   1/1     Running   10s
pod/web-storage-1   1/1     Running   12s

NAME                               STATUS   VOLUME         CAPACITY   STORAGECLASS
persistentvolumeclaim/data-web-storage-0   Bound    pvc-xxx...   100Mi      local-path
persistentvolumeclaim/data-web-storage-1   Bound    pvc-yyy...   100Mi      local-path
```

PVC 이름 패턴: `<volumeClaimTemplate이름>-<Pod이름>`

### 각 Pod에 다른 데이터 쓰기

```bash
kubectl exec web-storage-0 -- sh -c "echo 'I am pod-0' > /usr/share/nginx/html/index.html"
kubectl exec web-storage-1 -- sh -c "echo 'I am pod-1' > /usr/share/nginx/html/index.html"
```

### Pod 재시작 후 데이터 유지 확인

```bash
kubectl delete pod web-storage-0
kubectl wait --for=condition=Ready pod/web-storage-0 --timeout=60s
kubectl exec web-storage-0 -- cat /usr/share/nginx/html/index.html
```

```
I am pod-0
```

Pod가 재시작되어도 같은 PVC를 재마운트하므로 데이터가 유지됩니다.

!!! warning "PVC는 StatefulSet 삭제 후에도 남음"
    StatefulSet을 삭제해도 `volumeClaimTemplates`로 생성된 PVC는 **자동으로 삭제되지 않습니다.**
    데이터 보호를 위한 의도적인 설계입니다. 정리할 때는 PVC를 별도로 삭제해야 합니다.

---

## 5) 스케일링

### 스케일 업 (2 → 3)

```bash
kubectl scale statefulset web-storage --replicas=3
kubectl get pods -w
```

새 Pod `web-storage-2`가 생성되며, PVC `data-web-storage-2`도 자동으로 만들어집니다.

### 스케일 다운 (3 → 1)

```bash
kubectl scale statefulset web-storage --replicas=1
kubectl get pods -w
```

**역순(2 → 1 순서)**으로 Pod가 종료됩니다. 데이터 정합성이 중요한 DB 클러스터에서 레플리카를 먼저 끄고 마스터를 마지막으로 끄는 패턴을 지원합니다.

!!! info "스케일 다운 시 PVC는 유지"
    Pod가 삭제되어도 PVC는 남아 있습니다. 다시 스케일 업하면 기존 PVC에 재마운트됩니다.

---

## 6) 롤링 업데이트

StatefulSet의 기본 업데이트 전략은 `RollingUpdate`이며, **역순(높은 번호 → 낮은 번호)**으로 업데이트됩니다.

```bash
kubectl set image statefulset/web nginx=nginx:1.26
kubectl rollout status statefulset/web
```

Pod 업데이트 순서: `web-2` → `web-1` → `web-0`

### 파티션 업데이트 (카나리 배포)

`partition`을 설정하면 특정 번호 이상의 Pod만 먼저 업데이트합니다.

```bash
# web-2만 먼저 업데이트 (0, 1은 그대로)
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'
kubectl set image statefulset/web nginx=nginx:1.27

# web-2 확인 후 전체 적용
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
```

이 패턴으로 일부 Pod만 새 버전으로 먼저 테스트하고 문제가 없으면 전체에 적용합니다.

---

## 정리 (리소스 삭제)

```bash
# StatefulSet 삭제
kubectl delete statefulset web web-storage
kubectl delete svc web

# PVC는 자동 삭제되지 않으므로 별도 삭제
kubectl delete pvc -l app=web-storage
# 또는
kubectl get pvc | grep web-storage | awk '{print $1}' | xargs kubectl delete pvc
```

=== "macOS/Linux"
    ```bash
    kubectl delete statefulset web web-storage
    kubectl delete svc web
    kubectl get pvc | grep web-storage | awk '{print $1}' | xargs kubectl delete pvc
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl delete statefulset web web-storage
    kubectl delete svc web
    kubectl get pvc | Select-String "web-storage" | ForEach-Object { ($_ -split "\s+")[0] } | ForEach-Object { kubectl delete pvc $_ }
    ```
