# 1-8. kubectl 기본 명령어 & 클러스터 구조 탐색

## 실습 목표

- kubectl의 핵심 명령어 패턴을 익힌다.
- Control Plane과 Worker Node 구성 요소를 직접 확인한다.
- `kube-system` 네임스페이스에서 K8s 내부 컴포넌트를 탐색한다.

## 전제 조건

- 1-7 실습 완료 (로컬 클러스터 실행 중)

---

## 1) kubectl 기본 명령어 패턴

kubectl 명령어는 `kubectl <동사> <리소스> [이름]` 형태입니다.

| 동사 | 설명 |
|------|------|
| `get` | 리소스 목록 또는 상세 조회 |
| `describe` | 리소스 상세 이벤트/상태 조회 |
| `apply -f` | YAML 파일로 리소스 생성/업데이트 |
| `delete` | 리소스 삭제 |
| `logs` | Pod 컨테이너 로그 확인 |
| `exec` | Pod 내부 명령어 실행 |
| `port-forward` | 로컬 포트를 Pod/Service에 포워딩 |

---

## 2) 노드 탐색

```bash
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node rancher-desktop
```

`describe` 출력에서 확인할 항목:

- `Roles`: control-plane 여부
- `Capacity` / `Allocatable`: CPU·메모리 가용량
- `Conditions`: 노드 상태 (Ready, MemoryPressure 등)
- `System Info`: K8s 버전, Container Runtime 버전

---

## 3) kube-system 네임스페이스 탐색

```bash
kubectl get pods -n kube-system
```

!!! info "-n kube-system"
    `-n kube-system`은 **네임스페이스를 지정**하는 옵션입니다. K8s 내부 시스템 Pod들이 모여있는 별도 공간입니다.

Rancher Desktop(K3s) 환경 예시:

```
NAME                                      READY   STATUS      RESTARTS   AGE
coredns-695cbbfcb9-jgvg8                  1/1     Running     0          1d
local-path-provisioner-546dfc6456-79c6c   1/1     Running     0          1d
metrics-server-c8774f4f4-b8r9n            1/1     Running     0          1d
traefik-788bc4688c-jwc4j                  1/1     Running     0          1d
```

| Pod 이름 패턴 | 역할 |
|---|---|
| `coredns-*` | 클러스터 내부 DNS |
| `local-path-provisioner-*` | 로컬 디스크 PVC 자동 프로비저닝 |
| `metrics-server-*` | CPU·메모리 메트릭 수집 |
| `traefik-*` | Ingress Controller |

!!! note "K3s는 control plane이 Pod로 뜨지 않습니다"
    일반 K8s에서는 `kube-apiserver-*`, `etcd-*` 등이 Pod로 보이지만, K3s는 이 컴포넌트들을 단일 프로세스로 실행합니다.

```bash
kubectl describe pod -l k8s-app=kube-dns -n kube-system
```

---

## 4) 리소스 단축 이름(alias)

```bash
kubectl get po          # pods
kubectl get deploy      # deployments
kubectl get svc         # services
kubectl get cm          # configmaps
kubectl get ns          # namespaces
kubectl get pv          # persistentvolumes
kubectl get pvc         # persistentvolumeclaims
kubectl get hpa         # horizontalpodautoscalers
```

---

## 5) 출력 형식 옵션

```bash
kubectl get nodes                   # 기본 테이블
kubectl get nodes -o wide           # 확장 정보
kubectl get nodes -o yaml           # YAML 전체 출력
kubectl get nodes -o json           # JSON 출력
kubectl get nodes --show-labels     # 라벨 포함
```

---

## 6) 네임스페이스 전체 조회

```bash
kubectl get pods -A                # 모든 네임스페이스 Pod 조회
kubectl get pods --all-namespaces  # 위와 동일
```

---

## 7) 실습 — API Server에 직접 요청

=== "macOS/Linux"
    ```bash
    kubectl proxy &
    ```

    새 터미널에서:

    ```bash
    curl http://localhost:8001/api/v1/pods
    ```

    프록시 종료:

    ```bash
    fg
    # Ctrl+C
    ```
=== "Windows PowerShell"
    ```powershell
    Start-Job { kubectl proxy }
    ```

    새 터미널에서:

    ```powershell
    curl.exe http://localhost:8001/api/v1/pods
    ```

    프록시 종료:

    ```powershell
    Get-Job | Stop-Job
    Get-Job | Remove-Job
    ```

---

## 정리

| 확인 항목 | 명령어 |
|---------|--------|
| 노드 상태 | `kubectl get nodes -o wide` |
| Control Plane Pod 목록 | `kubectl get pods -n kube-system` |
| 모든 네임스페이스 Pod | `kubectl get pods -A` |
