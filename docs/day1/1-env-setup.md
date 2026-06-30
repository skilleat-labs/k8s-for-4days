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

=== "macOS/Linux"
    ```bash
    cat ~/.kube/config
    ```
=== "Windows PowerShell"
    ```powershell
    Get-Content "$env:USERPROFILE\.kube\config"
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

=== "macOS/Linux"
    ```bash
    source <(kubectl completion bash)
    echo 'source <(kubectl completion bash)' >> ~/.bashrc
    source ~/.bashrc
    ```
=== "Windows PowerShell"
    ```powershell
    New-Item -ItemType File -Force -Path $PROFILE
    kubectl completion powershell | Out-String | Invoke-Expression
    kubectl completion powershell >> $PROFILE
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
