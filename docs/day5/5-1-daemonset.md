# DaemonSet 실습

## 실습 목표

- DaemonSet이 무엇인지, 언제 쓰는지 이해한다.
- 기본 DaemonSet을 만들고 노드당 1개씩 실행됨을 확인한다.
- 노드 셀렉터와 Toleration으로 배포 범위를 제어한다.
- 롤링 업데이트로 무중단 업데이트를 적용한다.
- 로그 수집이라는 실제 운영 패턴을 실습한다.

---

## DaemonSet이란?

DaemonSet은 **클러스터의 모든 노드(또는 특정 노드)에 Pod를 1개씩 자동으로 배포**하는 컨트롤러입니다.

```
노드 A  →  Pod 1개
노드 B  →  Pod 1개
노드 C  →  Pod 1개
```

노드가 새로 추가되면 Pod가 자동으로 생성되고, 노드가 삭제되면 Pod도 함께 사라집니다.

### Deployment와 비교

| 항목 | Deployment | DaemonSet |
|---|---|---|
| Pod 수 결정 | `replicas` 숫자로 지정 | 노드 수에 따라 자동 결정 |
| 배포 위치 | 스케줄러가 임의 배치 | 모든 노드(또는 지정 노드)에 1개씩 |
| 주요 용도 | 웹 서버, API 서버 | 모니터링 에이전트, 로그 수집기, CNI |

### 실제 운영에서 사용하는 경우

| 용도 | 예시 |
|---|---|
| 로그 수집 | Fluentd, Filebeat |
| 모니터링 에이전트 | Prometheus Node Exporter, Datadog Agent |
| 네트워크 플러그인 | Cilium, Calico, Flannel |
| 보안 에이전트 | Falco |
| 스토리지 드라이버 | CSI 노드 플러그인 |

---

## 1) 기본 DaemonSet 생성

`daemonset-basic.yaml` 파일을 만듭니다.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
  labels:
    app: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
        - name: agent
          image: busybox:1.36-musl
          command: ["sh", "-c", "while true; do echo '[$(date)] log from $(hostname)'; sleep 5; done"]
          resources:
            requests:
              cpu: 10m
              memory: 16Mi
            limits:
              cpu: 50m
              memory: 32Mi
```

```bash
kubectl apply -f daemonset-basic.yaml
kubectl get daemonset
kubectl get pods -o wide
```

출력 예시:

```
NAME        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
log-agent   1         1         1       1            1           <none>          10s
```

!!! info "로컬 환경에서 DESIRED가 1인 이유"
    Rancher Desktop과 Docker Desktop은 **단일 노드** 클러스터입니다.
    노드가 1개이므로 DaemonSet Pod도 1개만 생성됩니다.
    멀티 노드 클러스터에서는 노드 수만큼 Pod가 생깁니다.

로그 확인:

```bash
kubectl logs -l app=log-agent
```

---

## 2) 노드 정보 확인

DaemonSet은 어느 노드에 배포됐는지가 중요합니다.

```bash
# Pod가 어느 노드에 있는지 확인
kubectl get pods -o wide -l app=log-agent

# 노드 목록 확인
kubectl get nodes --show-labels
```

---

## 3) 노드 셀렉터 — 특정 노드에만 배포

`nodeSelector`를 사용하면 특정 레이블이 있는 노드에만 Pod를 배포할 수 있습니다.

### 노드에 레이블 추가

```bash
kubectl label node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') role=worker
kubectl get nodes --show-labels
```

### nodeSelector 적용

`daemonset-selector.yaml`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent-selector
spec:
  selector:
    matchLabels:
      app: log-agent-selector
  template:
    metadata:
      labels:
        app: log-agent-selector
    spec:
      nodeSelector:
        role: worker          # 이 레이블이 있는 노드에만 배포
      containers:
        - name: agent
          image: busybox:1.36-musl
          command: ["sh", "-c", "while true; do echo 'worker-only log'; sleep 5; done"]
```

```bash
kubectl apply -f daemonset-selector.yaml
kubectl get pods -o wide -l app=log-agent-selector
```

레이블이 없는 노드에는 Pod가 생성되지 않습니다.

---

## 4) Toleration — 컨트롤 플레인 노드 포함

기본적으로 DaemonSet은 컨트롤 플레인 노드(마스터)에는 배포되지 않습니다.
컨트롤 플레인에도 배포하려면 `tolerations`를 추가합니다.

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: agent
          image: busybox:1.36-musl
          command: ["sh", "-c", "while true; do sleep 60; done"]
```

!!! info "로컬 환경에서는 효과 없음"
    Rancher Desktop / Docker Desktop의 단일 노드는 컨트롤 플레인과 워커 역할을 동시에 합니다.
    Toleration 효과는 멀티 노드 클러스터에서 확인할 수 있습니다.

---

## 5) 롤링 업데이트

DaemonSet도 기본적으로 `RollingUpdate` 전략을 사용합니다.
`maxUnavailable`로 한 번에 업데이트할 노드 수를 제어합니다.

### 현재 업데이트 전략 확인

```bash
kubectl get daemonset log-agent -o yaml | grep -A5 updateStrategy
```

### 이미지 업데이트

=== "macOS/Linux"
    ```bash
    kubectl set image daemonset/log-agent agent=busybox:1.36
    kubectl rollout status daemonset/log-agent
    kubectl rollout history daemonset/log-agent
    ```
=== "Windows PowerShell"
    ```powershell
    kubectl set image daemonset/log-agent agent=busybox:1.36
    kubectl rollout status daemonset/log-agent
    kubectl rollout history daemonset/log-agent
    ```

### 롤백

```bash
kubectl rollout undo daemonset/log-agent
```

---

## 6) 운영 패턴 — 로그 수집 에이전트

실제 운영에서 자주 쓰이는 로그 수집 DaemonSet 패턴입니다.
호스트의 `/var/log`를 컨테이너에 마운트해서 노드 로그를 수집합니다.

`daemonset-log-collector.yaml`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: default
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
        - name: collector
          image: busybox:1.36-musl
          command:
            - sh
            - -c
            - |
              echo "=== 노드 로그 수집 시작 ==="
              while true; do
                echo "[$(date)] 수집 중: $(ls /host-logs | head -5)"
                sleep 10
              done
          volumeMounts:
            - name: host-log
              mountPath: /host-logs
              readOnly: true
          resources:
            requests:
              cpu: 10m
              memory: 16Mi
            limits:
              cpu: 100m
              memory: 64Mi
      volumes:
        - name: host-log
          hostPath:
            path: /var/log    # 노드의 실제 로그 경로
            type: Directory
```

```bash
kubectl apply -f daemonset-log-collector.yaml
kubectl get pods -l app=log-collector
kubectl logs -l app=log-collector
```

!!! warning "hostPath 보안 주의"
    `hostPath`는 노드의 실제 파일시스템을 마운트합니다.
    운영 환경에서는 `readOnly: true`를 반드시 설정하고, 최소한의 경로만 허용하세요.

---

## 정리 (리소스 삭제)

```bash
kubectl delete daemonset log-agent log-agent-selector log-collector
kubectl label node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') role-
```
