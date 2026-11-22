# 1-7. 실습 환경 설치 (VS Code / Rancher Desktop / Docker Desktop)

## 실습 목표

- Docker Desktop 또는 Rancher Desktop에서 로컬 Kubernetes 클러스터를 활성화
- kubectl이 클러스터에 연결되었음을 확인
- 이후 모든 실습의 기반 환경 준비

## 준비물

- macOS 또는 Windows PC
- 인터넷 연결
- 관리자 권한

---

## 사전 준비 — VS Code 설치 및 확장

### 1) VS Code 설치

[VS Code 공식 다운로드 페이지](https://code.visualstudio.com/){target=_blank}에서 운영체제에 맞는 버전을 설치합니다.

### 2) 권장 확장 설치

| 확장 이름 | 제공 | 역할 |
|--------|------|------|
| **YAML** | Red Hat | YAML 문법 검사 · 스키마 기반 자동완성 |
| **Indent Rainbow** | oderwat | 들여쓰기 단계별 색상 표시 |

---

## VirtualBox 사용 중이라면 먼저 확인

VM 실습 후 Docker Desktop / Rancher Desktop으로 넘어올 때 아래 항목을 순서대로 확인하세요.

### 1) VirtualBox VM 전부 종료

Docker Desktop / Rancher Desktop 설치 중 재부팅이 발생합니다. VM이 켜진 상태에서 재부팅하면 디스크 손상 위험이 있습니다.

```
VirtualBox → 모든 VM → 오른쪽 클릭 → 닫기 → 전원 끄기
```

### 2) Windows — WSL2 백엔드 사용 (Hyper-V 대신)

Docker Desktop은 **Hyper-V** 또는 **WSL2** 백엔드를 선택할 수 있습니다.
VirtualBox를 계속 쓸 예정이라면 반드시 **WSL2 백엔드**를 선택하세요.

!!! warning "Hyper-V 활성화 시 VirtualBox 충돌"
    Hyper-V를 활성화하면 VirtualBox 6.1 미만에서 64비트 VM이 실행되지 않습니다.
    VirtualBox 7.0 이상은 공존 가능하지만, WSL2 백엔드가 더 안정적입니다.

Docker Desktop 설치 화면에서 **"Use WSL 2 instead of Hyper-V"** 옵션을 선택합니다.

### 3) 포트 충돌 확인

VirtualBox에서 포트 포워딩을 설정한 경우 아래 포트가 겹치면 Kubernetes가 뜨지 않습니다.

| 포트 | 용도 |
|------|------|
| 6443 | Kubernetes API Server |
| 8080 | 실습 중 자주 쓰는 NodePort |

=== "Windows PowerShell"
    ```powershell
    netstat -ano | Select-String "6443|8080"
    ```
=== "macOS"
    ```bash
    lsof -i :6443 -i :8080
    ```

포트를 점유하는 프로세스가 VirtualBox면 해당 VM을 종료하고 진행합니다.

### 4) kubeconfig 덮어쓰기 주의

Docker Desktop / Rancher Desktop 첫 실행 시 `~/.kube/config`의 **current-context가 자동으로 변경**됩니다.
기존 VM 기반 kubeconfig를 보존하고 싶다면 미리 백업하세요.

=== "Windows PowerShell"
    ```powershell
    Copy-Item "$env:USERPROFILE\.kube\config" "$env:USERPROFILE\.kube\config.bak"
    ```
=== "macOS"
    ```bash
    cp ~/.kube/config ~/.kube/config.bak
    ```

---

## 옵션 A — Docker Desktop으로 설정

### 1) Docker Desktop 설치

[Docker Desktop 공식 다운로드 페이지](https://www.docker.com/products/docker-desktop/){target=_blank}에서 운영체제에 맞는 버전을 설치합니다.

### 2) Kubernetes 활성화

1. 오른쪽 상단 톱니바퀴(Settings) 클릭
2. 왼쪽 메뉴에서 **Kubernetes** 선택
3. **Enable Kubernetes** 체크박스 활성화
4. **Apply & Restart** 버튼 클릭
5. 하단 상태 표시줄에 초록색 Kubernetes 아이콘이 나타날 때까지 대기 (1~3분)

### 3) 연결 확인

```bash
kubectl config current-context
```

→ `docker-desktop` 출력되면 정상

```bash
kubectl get nodes
```

```
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   1m    v1.29.x
```

---

## 옵션 B — Rancher Desktop으로 설정

### 0) Windows 사전 준비 (Windows 사용자만)

**① BIOS 가상화 활성화 확인**

```powershell
(Get-ComputerInfo).HyperVRequirementVirtualizationFirmwareEnabled
```

→ `True` 출력되면 정상.

**② WSL2 설치 및 활성화**

```powershell
wsl --install
```

재부팅 후:

```powershell
wsl --status
```

→ `기본 버전: 2`로 표시되면 준비 완료

WSL 버전 1인 경우:

```powershell
wsl --set-default-version 2
```

!!! info "Windows 버전 요구사항"
    Windows 10 버전 2004(빌드 19041) 이상 또는 Windows 11

### 1) Rancher Desktop 설치

1. [Rancher Desktop 공식 다운로드 페이지](https://rancherdesktop.io/){target=_blank}에서 설치 파일 다운로드
2. 설치 후 실행
3. 초기 설정 화면에서:
   - **Container Runtime**: `dockerd (moby)`
   - **Kubernetes Version**: 최신 안정 버전 (예: v1.29.x)

### 2) kubeconfig 자동 설정 확인

=== "Windows PowerShell"
    ```powershell
    Get-Content "$env:USERPROFILE\.kube\config"
    ```
=== "macOS/Linux"
    ```bash
    cat ~/.kube/config
    ```

→ `rancher-desktop` 클러스터 항목이 포함되면 정상

### 3) 연결 확인

```bash
kubectl config current-context
```

→ `rancher-desktop` 출력되면 정상

```bash
kubectl get nodes
```

```
NAME                   STATUS   ROLES                  AGE   VERSION
lima-rancher-desktop   Ready    control-plane,master   2m    v1.29.x
```

---

## 공통 확인 사항

### kubectl 자동완성 활성화

=== "Windows PowerShell"
    실행 정책 확인 — `Restricted`면 아래 명령어 먼저 실행 (`-Force`로 확인 프롬프트 생략):
    ```powershell
    Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force
    ```
    Profile에 자동완성 등록 후 현재 세션 적용:
    ```powershell
    New-Item -ItemType File -Force -Path $PROFILE
    Add-Content -Path $PROFILE -Value 'kubectl completion powershell | Out-String | Invoke-Expression'
    . $PROFILE
    ```
=== "macOS/Linux"
    ```bash
    source <(kubectl completion bash)
    echo 'source <(kubectl completion bash)' >> ~/.bashrc
    source ~/.bashrc
    ```

### 클러스터 전체 상태 확인

```bash
kubectl cluster-info
kubectl get nodes -o wide
```

---

## Windows PowerShell 사용 시 참고 사항

| 상황 | macOS / Linux | Windows PowerShell |
|------|---------------|-------------------|
| HTTP 요청 테스트 | `curl http://...` | `curl.exe http://...` |
| 텍스트 필터 | `kubectl ... \| grep "패턴"` | `kubectl ... \| Select-String "패턴"` |
| base64 인코딩 | `echo -n "값" \| base64` | `[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("값"))` |

---

## 트러블슈팅

| 증상 | 확인 사항 |
|------|---------|
| `kubectl: command not found` | Rancher Desktop 재설치 또는 PATH 설정 확인 |
| `Unable to connect to the server` | Rancher Desktop이 실행 중인지, Kubernetes가 활성화되었는지 확인 |
| 노드 상태 `NotReady` | 1~2분 후 재확인, 또는 Rancher Desktop 재시작 |
