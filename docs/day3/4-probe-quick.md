# Probe 실습 — Liveness / Readiness / Startup

## Liveness Probe — 실패 시 자동 재시작

30초 후 파일을 삭제해 Probe를 실패시키고, 컨테이너가 자동 재시작되는 것을 확인합니다.

```yaml
# pod-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-fail
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "touch /tmp/healthy && sleep 30 && rm /tmp/healthy && sleep 3600"
      livenessProbe:
        exec:
          command: [cat, /tmp/healthy]
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
```

```bash
kubectl apply -f pod-liveness.yaml
kubectl get pod liveness-fail -w
```

30~60초 후 `RESTARTS`가 1로 올라가면 성공입니다.

```bash
kubectl describe pod liveness-fail   # Events에서 Liveness probe failed 확인
kubectl delete pod liveness-fail
```

---

## Readiness Probe — 준비 안 된 Pod는 트래픽 차단

파일을 삭제하면 Endpoint에서 제거되고, 다시 만들면 자동 복귀되는 것을 확인합니다.

```yaml
# pod-readiness.yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-exec
  labels:
    app: readiness-exec
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "touch /tmp/ready && sleep 3600"
      readinessProbe:
        exec:
          command: [cat, /tmp/ready]
        initialDelaySeconds: 3
        periodSeconds: 3
        failureThreshold: 2
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-svc
spec:
  selector:
    app: readiness-exec
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f pod-readiness.yaml
kubectl get endpoints readiness-svc   # IP 1개 확인
```

**Readiness 실패 → Endpoint 제거**

```bash
kubectl exec readiness-exec -- rm /tmp/ready
kubectl get pod readiness-exec        # READY 0/1 확인
kubectl get endpoints readiness-svc   # <none> 확인
```

**Readiness 회복 → Endpoint 자동 복귀**

```bash
kubectl exec readiness-exec -- touch /tmp/ready
kubectl get pod readiness-exec        # READY 1/1 확인
kubectl get endpoints readiness-svc   # IP 복귀 확인
kubectl delete -f pod-readiness.yaml
```

---

## Startup Probe — 초기화가 긴 앱 보호

초기화에 15초 걸리는 앱을 Startup Probe로 보호합니다.
Startup이 성공하기 전까지 Liveness는 동작하지 않아 불필요한 재시작이 없습니다.

```yaml
# pod-startup.yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-app
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "sleep 15 && touch /tmp/started && sleep 3600"
      startupProbe:
        exec:
          command: [cat, /tmp/started]
        failureThreshold: 10
        periodSeconds: 3        # 최대 30초 대기
      livenessProbe:
        exec:
          command: [cat, /tmp/started]
        periodSeconds: 10
        failureThreshold: 3
```

```bash
kubectl apply -f pod-startup.yaml
kubectl get pod startup-app -w
```

15초 동안 `RESTARTS 0`을 유지하다가 `READY 1/1`로 바뀌면 성공입니다.

```bash
kubectl delete pod startup-app
```

---

## 3가지 Probe 비교

| Probe | 실패 시 동작 | 언제 사용 |
|---|---|---|
| Liveness | 컨테이너 재시작 | 교착 상태·무한 루프 감지 |
| Readiness | Service Endpoint에서 제거 | 초기화 중 트래픽 차단 |
| Startup | Liveness/Readiness 보호 | 초기화가 긴 앱 |
