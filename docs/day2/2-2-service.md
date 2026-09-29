# 2-2. Service 실습 (ClusterIP · NodePort · LoadBalancer · port-forward)

## 실습 목표

- ClusterIP, NodePort, LoadBalancer Service 생성 및 차이점 체감
- 클러스터 내 모든 Pod에서 ClusterIP 통신 확인
- Service 이름(DNS), ClusterIP, LoadBalancer 세 가지 방식으로 통신
- label selector 기반 라우팅 동작 확인

!!! warning "Docker Desktop 환경 주의"
    이 실습은 **Docker Desktop Kubernetes** 기준입니다.
    Docker Desktop에서는 **NodePort로 `localhost` 접근이 되지 않습니다.**
    외부에서 `localhost`로 접근하려면 **LoadBalancer** 타입을 사용해야 합니다.
    Docker Desktop은 LoadBalancer Service에 자동으로 `localhost`를 EXTERNAL-IP로 할당합니다.

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

`-o wide`는 Pod의 IP 주소와 배치 노드 같은 추가 컬럼을 함께 출력합니다. Endpoints의 IP 목록과 Pod IP가 일치하는지 비교할 때 유용합니다.

```bash
kubectl get endpoints rollout-svc
kubectl get pods -o wide
```

---

## 2) Pod 내부에서 ClusterIP로 통신

### 방법 A — ClusterIP 직접 사용

```bash
kubectl get svc rollout-svc
```

=== "Windows PowerShell"
    ```powershell
    kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm `
      -- curl http://10.96.45.123
    ```
=== "macOS/Linux"
    ```bash
    kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm \
      -- curl http://10.96.45.123
    ```

### 방법 B — Service 이름(DNS)으로 접근

=== "Windows PowerShell"
    ```powershell
    kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm `
      -- curl http://rollout-svc
    ```
=== "macOS/Linux"
    ```bash
    kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm \
      -- curl http://rollout-svc
    ```

FQDN 형식:

=== "Windows PowerShell"
    ```powershell
    kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm `
      -- curl http://rollout-svc.default.svc.cluster.local
    ```
=== "macOS/Linux"
    ```bash
    kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm \
      -- curl http://rollout-svc.default.svc.cluster.local
    ```

!!! info "DNS 형식"
    `<service이름>.<namespace>.svc.cluster.local`

### 방법 C — 실행 중인 Pod 내부에서 접근

`rollout-demo` 이미지에는 curl이 없으므로 curl이 포함된 임시 Pod를 띄워서 접근합니다.

```bash
kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm -- sh

# Pod 내부에서:
curl http://rollout-svc
curl http://10.96.45.123
exit
```

---

## 3) label selector 동작 확인

`kubectl describe`는 리소스의 상세 정보와 이벤트를 출력합니다. Service가 어떤 Pod를 선택했는지, Endpoints가 올바르게 연결됐는지 확인할 때 사용합니다.

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

!!! warning "Docker Desktop에서는 NodePort로 localhost 접근 불가"
    NodePort는 개념 이해용으로만 실습합니다.
    Docker Desktop 환경에서는 `localhost:30080`으로 접근이 되지 않습니다.
    **외부 접근이 필요하면 5) LoadBalancer를 사용하세요.**

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

### 클러스터 내부에서만 접근 확인

```bash
kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm \
  -- curl http://rollout-nodeport
```

---

## 5) LoadBalancer Service 생성

Docker Desktop에서 **localhost로 외부 접근**하려면 LoadBalancer 타입을 사용합니다.
Docker Desktop은 LoadBalancer Service에 자동으로 `localhost`를 EXTERNAL-IP로 할당합니다.

`svc-lb.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rollout-lb
spec:
  type: LoadBalancer
  selector:
    app: rollout
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f svc-lb.yaml
kubectl get svc rollout-lb
```

**출력 예시:**

```
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
rollout-lb   LoadBalancer   10.96.12.34     localhost     80:31234/TCP   10s
```

`EXTERNAL-IP`가 `localhost`로 표시되면 정상입니다.

### localhost로 접근 확인

=== "Windows PowerShell"
    ```powershell
    curl.exe http://localhost
    ```
=== "macOS/Linux"
    ```bash
    curl http://localhost
    ```

---

## 6) port-forward (개발 시 빠른 접속)

```bash
kubectl port-forward svc/rollout-svc 8080:80
```

새 터미널에서:

=== "Windows PowerShell"
    ```powershell
    curl.exe http://localhost:8080
    ```
=== "macOS/Linux"
    ```bash
    curl http://localhost:8080
    ```

`Ctrl+C`로 종료.

---

## 7) Pod 삭제 후 Service 연속성 확인

LoadBalancer(`localhost:80`)로 반복 요청을 보내면서 Pod를 삭제해 Service가 자동으로 새 Pod로 전환되는 것을 확인합니다.

**터미널 1 — 반복 요청:**

=== "Windows PowerShell"
    ```powershell
    while ($true) { (curl.exe -s -o NUL -w "%{http_code}" http://localhost); Start-Sleep 1 }
    ```
=== "macOS/Linux"
    ```bash
    while true; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost; sleep 1; done
    ```

**터미널 2 — Pod 강제 삭제:**

`--wait=false`를 붙이면 삭제 완료를 기다리지 않고 즉시 프롬프트를 반환합니다. 터미널 1에서 요청을 계속 보내는 동안 빠르게 다음 명령을 실행하기 위해 사용합니다.

```bash
kubectl delete pod -l app=rollout --wait=false
kubectl get pods -w
```

응답 코드가 계속 `200`으로 유지되면 Service가 새 Pod로 자동 전환된 것입니다.

---

## 8) kubectl expose — 명령어로 Service 즉시 생성

=== "Windows PowerShell"
    ```powershell
    kubectl expose deployment rollout-deploy `
      --name=rollout-expose `
      --type=NodePort `
      --port=80 `
      --target-port=8080
    ```
=== "macOS/Linux"
    ```bash
    kubectl expose deployment rollout-deploy \
      --name=rollout-expose \
      --type=NodePort \
      --port=80 \
      --target-port=8080
    ```

```bash
kubectl get svc rollout-expose
```

**dry-run으로 YAML 미리 확인:**

=== "Windows PowerShell"
    ```powershell
    kubectl expose deployment rollout-deploy `
      --name=rollout-expose `
      --type=NodePort `
      --port=80 `
      --target-port=8080 `
      --dry-run=client -o yaml
    ```
=== "macOS/Linux"
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
kubectl delete -f svc-lb.yaml
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
| NodePort 접속 불가 | Docker Desktop에서는 NodePort로 localhost 접근 불가 — LoadBalancer 타입 사용 |
| DNS 이름으로 접근 불가 | Service와 Pod가 같은 네임스페이스에 있는지 확인 |
