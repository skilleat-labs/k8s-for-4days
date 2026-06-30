# 2-2. Service 실습 (ClusterIP · NodePort · port-forward)

## 실습 목표

- ClusterIP, NodePort Service 생성 및 차이점 체감
- 클러스터 내 모든 Pod에서 ClusterIP 통신 확인
- Service 이름(DNS), ClusterIP, NodePort 세 가지 방식으로 통신
- label selector 기반 라우팅 동작 확인

## 전제 조건

```bash
kubectl apply -f deploy-rollout.yaml
kubectl get po -l app=rollout
```

## Service 개념

Pod는 재시작 시 IP가 변경되므로, Service는 고정된 진입점(ClusterIP 또는 NodePort)을 제공하여 항상 동일한 주소로 접근 가능하게 합니다.

---

## 1) ClusterIP Service 생성

`svc-clusterip.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rollout-svc
spec:
  type: ClusterIP
  selector:
    app: rollout
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f svc-clusterip.yaml
kubectl get svc rollout-svc
```

**출력 예시:**

```
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
rollout-svc   ClusterIP   10.96.45.123   <none>        80/TCP    10s
```

### Endpoints 확인

```bash
kubectl get endpoints rollout-svc
kubectl get pods -o wide
```

---

## 2) Pod 내부에서 ClusterIP로 통신

### 방법 A — ClusterIP 직접 사용

```bash
kubectl get svc rollout-svc
kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm \
  -- curl http://10.96.45.123
```

### 방법 B — Service 이름(DNS)으로 접근

```bash
kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm \
  -- curl http://rollout-svc
```

FQDN 형식:

```bash
kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm \
  -- curl http://rollout-svc.default.svc.cluster.local
```

!!! info "DNS 형식"
    `<service이름>.<namespace>.svc.cluster.local`

### 방법 C — 실행 중인 Pod 내부에서 접근

```bash
kubectl exec -it deployment/rollout-deploy -- sh

# Pod 내부에서:
curl http://rollout-svc
curl http://10.96.45.123
exit
```

---

## 3) label selector 동작 확인

```bash
kubectl describe svc rollout-svc
```

확인 항목:

- `Selector`: `app=rollout` — 해당 label을 가진 Pod에만 트래픽 전달
- `Endpoints`: 현재 연결된 Pod IP 목록
- `Port` / `TargetPort`: Service 포트 → Pod 포트 매핑

### label이 다른 Pod는 제외됨을 확인

```bash
kubectl run other-pod --image=nginx:1.25 --labels="app=other"
kubectl get endpoints rollout-svc
kubectl delete pod other-pod
```

---

## 4) NodePort Service 생성

`svc-nodeport.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rollout-nodeport
spec:
  type: NodePort
  selector:
    app: rollout
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

```bash
kubectl apply -f svc-nodeport.yaml
kubectl get svc rollout-nodeport
```

**출력 예시:**

```
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
rollout-nodeport   NodePort   10.96.78.200    <none>        80:30080/TCP   10s
```

### 노드 IP 확인

```bash
kubectl get nodes -o wide
```

### 세 가지 방법으로 모두 접근

=== "macOS/Linux"
    ```bash
    # 1. 로컬에서 localhost:nodePort (Docker Desktop/Rancher Desktop)
    curl http://localhost:30080

    # 2. 노드 IP:nodePort
    curl http://<노드 INTERNAL-IP>:30080

    # 3. ClusterIP (클러스터 내부 Pod에서)
    kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm \
      -- curl http://rollout-nodeport
    ```
=== "Windows PowerShell"
    ```powershell
    # 1. 로컬에서 localhost:nodePort (Docker Desktop/Rancher Desktop)
    curl.exe http://localhost:30080

    # 2. 노드 IP:nodePort
    curl.exe http://<노드 INTERNAL-IP>:30080

    # 3. ClusterIP (클러스터 내부 Pod에서)
    kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm `
      -- curl http://rollout-nodeport
    ```

---

## 5) port-forward (개발 시 빠른 접속)

```bash
kubectl port-forward svc/rollout-svc 8080:80
```

새 터미널에서:

=== "macOS/Linux"
    ```bash
    curl http://localhost:8080
    ```
=== "Windows PowerShell"
    ```powershell
    curl.exe http://localhost:8080
    ```

`Ctrl+C`로 종료.

---

## 6) Pod 삭제 후 Service 연속성 확인

**터미널 1 — 반복 요청:**

=== "macOS/Linux"
    ```bash
    while true; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:30080; sleep 1; done
    ```
=== "Windows PowerShell"
    ```powershell
    while ($true) { (curl.exe -s -o NUL -w "%{http_code}" http://localhost:30080); Start-Sleep 1 }
    ```

**터미널 2 — Pod 강제 삭제:**

```bash
kubectl delete pod -l app=rollout --wait=false
kubectl get pods -w
```

응답 코드가 계속 `200`으로 유지되면 Service가 새 Pod로 자동 전환된 것입니다.

---

## 7) kubectl expose — 명령어로 Service 즉시 생성

```bash
kubectl expose deployment rollout-deploy \
  --name=rollout-expose \
  --type=NodePort \
  --port=80 \
  --target-port=8080

kubectl get svc rollout-expose
```

**dry-run으로 YAML 미리 확인:**

```bash
kubectl expose deployment rollout-deploy \
  --name=rollout-expose \
  --type=NodePort \
  --port=80 \
  --target-port=8080 \
  --dry-run=client -o yaml
```

**YAML 파일 관리 권장:**

| 항목 | kubectl expose | YAML 파일 |
|------|---|---|
| 버전 관리 (git) | 불가 | 가능 |
| 팀 공유 · 리뷰 | 어려움 | 쉬움 |
| 재현성 | 명령어 기억에 의존 | 파일만 있으면 동일 재현 |
| nodePort 직접 지정 | 불가 | 가능 |
| label, annotation 추가 | 제한적 | 자유롭게 설정 |

!!! tip "유용한 경우"
    - 로컬에서 Pod·Deployment 동작을 빠르게 외부 노출해 확인할 때
    - `--dry-run=client -o yaml`로 YAML 시작점을 뽑아낼 때

**확인 후 삭제:**

```bash
kubectl delete svc rollout-expose
```

---

## 정리 (리소스 삭제)

```bash
kubectl delete -f svc-clusterip.yaml
kubectl delete -f svc-nodeport.yaml
```

---

## Service 타입 비교

| 타입 | 접근 방법 | 접근 범위 | 주요 용도 |
|------|---------|---------|---------|
| ClusterIP | Service명(DNS) 또는 ClusterIP | 클러스터 내부 Pod 전체 | 서비스 간 내부 통신 |
| NodePort | 노드IP:NodePort | 클러스터 외부 | 개발·테스트 환경 외부 노출 |
| LoadBalancer | 클라우드 LB IP | 클러스터 외부 | 프로덕션 외부 노출 (AKS 등) |

---

## 트러블슈팅

| 증상 | 확인 사항 |
|------|---------|
| Endpoints가 `<none>` | Pod label과 Service selector 일치 여부 확인 |
| ClusterIP로 접근 불가 | Pod 내부에서 실행했는지 확인 (로컬 터미널에서는 불가) |
| NodePort 접속 불가 | Rancher Desktop에서 `localhost:30080` 사용, 노드 IP는 `kubectl get nodes -o wide` 확인 |
| DNS 이름으로 접근 불가 | Service와 Pod가 같은 네임스페이스에 있는지 확인 |
