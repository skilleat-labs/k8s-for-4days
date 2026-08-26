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

---

## 8) VS Code로 YAML 파일 다루기

### 8-1. 권장 확장 설치

VS Code를 열고 아래 확장을 설치합니다.

| 확장 이름 | 제공 | 역할 |
|-----------|------|------|
| **YAML** | Red Hat | YAML 문법 검사 · 스키마 자동완성 |
| **Kubernetes** | Microsoft | kubectl 명령어 팔레트 · 클러스터 탐색기 |

### 8-2. VS Code에서 YAML 파일 작성 및 적용

작업 폴더를 VS Code로 열기:

=== "Windows PowerShell"
    ```powershell
    mkdir ~/k8s-lab; cd ~/k8s-lab
    code .
    ```
=== "macOS/Linux"
    ```bash
    mkdir ~/k8s-lab && cd ~/k8s-lab
    code .
    ```

VS Code 내 터미널(`Ctrl+`` `)에서 YAML 파일 생성:

=== "Windows PowerShell"
    ```powershell
    @'
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-pod
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
    '@ | Set-Content my-pod.yaml
    ```
=== "macOS/Linux"
    ```bash
    cat > my-pod.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-pod
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
    EOF
    ```

적용 및 확인:

```bash
kubectl apply -f my-pod.yaml
kubectl get pod my-pod
kubectl delete -f my-pod.yaml
```

!!! tip "YAML 자동완성 활용"
    VS Code에서 `.yaml` 파일을 열고 `apiVersion:` 입력 후 `Ctrl+Space`를 누르면 스키마 기반 자동완성이 동작합니다.

---

## 9) kubectl 자동완성 설정

터미널에서 `kubectl get po` 뒤에 `Tab`을 누르면 리소스 이름이 자동완성됩니다.

=== "Windows PowerShell"
    현재 세션에만 적용:
    ```powershell
    kubectl completion powershell | Out-String | Invoke-Expression
    ```

    **영구 적용** (PowerShell 시작 시 자동 로드):

    **① 실행 정책 확인 및 허용** (한 번만):
    ```powershell
    Get-ExecutionPolicy
    ```
    `Restricted` 또는 `AllSigned`가 나오면 아래 명령어 실행:
    ```powershell
    Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
    ```

    **② Profile 파일에 자동완성 줄 추가**:
    ```powershell
    New-Item -ItemType File -Force -Path $PROFILE
    Add-Content -Path $PROFILE -Value 'kubectl completion powershell | Out-String | Invoke-Expression'
    ```

    **③ 적용 확인**:
    ```powershell
    # 현재 세션에 바로 적용
    . $PROFILE
    ```

    !!! info "이후부터는 PowerShell을 새로 열 때마다 자동완성이 적용됩니다."
=== "macOS/Linux"
    ```bash
    source <(kubectl completion bash)
    echo 'source <(kubectl completion bash)' >> ~/.bashrc
    source ~/.bashrc
    ```

---

## 10) 공식 문서 & 레퍼런스

| 문서 | 링크 | 설명 |
|------|------|------|
| Kubernetes 공식 문서 | [kubernetes.io/docs](https://kubernetes.io/docs/home/){target=_blank} | 개념·가이드 전체 |
| kubectl 치트시트 | [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/){target=_blank} | 자주 쓰는 명령어 모음 |
| kubectl 명령어 레퍼런스 | [kubectl Reference](https://kubernetes.io/docs/reference/kubectl/){target=_blank} | 전체 옵션 상세 설명 |
| API 레퍼런스 | [Kubernetes API](https://kubernetes.io/docs/reference/kubernetes-api/){target=_blank} | YAML 스펙 전체 필드 확인 |

!!! tip "모르는 YAML 필드가 있을 때"
    ```bash
    kubectl explain pod.spec.containers
    kubectl explain deployment.spec.template
    ```
    공식 문서 없이도 터미널에서 바로 필드 설명을 확인할 수 있습니다.

---

## 정리

| 확인 항목 | 명령어 |
|---------|--------|
| 노드 상태 | `kubectl get nodes -o wide` |
| Control Plane Pod 목록 | `kubectl get pods -n kube-system` |
| 모든 네임스페이스 Pod | `kubectl get pods -A` |
| YAML 적용 | `kubectl apply -f 파일명.yaml` |
| 자동완성 (macOS/Linux) | `source <(kubectl completion bash)` |
| 자동완성 (PowerShell 영구) | `Add-Content -Path $PROFILE -Value 'kubectl completion powershell \| Out-String \| Invoke-Expression'` |
