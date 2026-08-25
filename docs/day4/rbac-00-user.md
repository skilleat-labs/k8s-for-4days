# RBAC 00 — 개발팀 User 만들기 (인증서 기반)

## 실습 목표

- Kubernetes에서 User가 어떻게 관리되는지 이해한다.
- openssl로 인증서를 생성하고 K8s에 승인 요청한다.
- `dev-user` kubeconfig context를 만들어 사용자 전환을 실습한다.

---

## 개념

Kubernetes에는 `kubectl create user` 같은 명령어가 **없습니다.**
User는 K8s 내부 리소스가 아니라, **인증서(Certificate)의 CN(Common Name) 필드**로 신원을 표현합니다.

```
openssl로 개인키 + CSR 생성
        ↓
K8s CertificateSigningRequest 리소스로 제출
        ↓
관리자(admin)가 kubectl certificate approve 승인
        ↓
인증서 발급 → kubeconfig에 등록
        ↓
kubectl config use-context 로 해당 유저로 전환
```

!!! info "이 실습에서 만드는 User"
    - `dev-user` — 개발팀을 대표하는 사용자. 이후 RBAC 실습에서 이 유저에게 권한을 부여합니다.

---

## 사전 조건

`openssl`이 설치되어 있어야 합니다.

=== "Windows PowerShell"
    Git for Windows를 설치하면 openssl이 함께 제공됩니다.

    설치 확인:
    ```powershell
    & "C:\Program Files\Git\usr\bin\openssl.exe" version
    ```

    매번 경로 입력이 번거로우면 PowerShell에서 alias 등록:
    ```powershell
    Set-Alias openssl "C:\Program Files\Git\usr\bin\openssl.exe"
    ```
=== "macOS/Linux"
    ```bash
    openssl version
    ```

---

## 1) 작업 디렉토리 준비

=== "Windows PowerShell"
    ```powershell
    mkdir ~/rbac-lab; cd ~/rbac-lab
    ```
=== "macOS/Linux"
    ```bash
    mkdir ~/rbac-lab && cd ~/rbac-lab
    ```

---

## 2) 개인키 & CSR 생성

개인키를 만들고, 그 키로 CSR(인증서 서명 요청)을 생성합니다.
`/CN=dev-user`가 K8s에서 인식하는 **사용자 이름**, `/O=dev-team`이 **그룹 이름**입니다.

=== "Windows PowerShell"
    ```powershell
    & "C:\Program Files\Git\usr\bin\openssl.exe" genrsa -out dev-user.key 2048
    & "C:\Program Files\Git\usr\bin\openssl.exe" req -new -key dev-user.key -out dev-user.csr -subj "/CN=dev-user/O=dev-team"
    ```
=== "macOS/Linux"
    ```bash
    openssl genrsa -out dev-user.key 2048
    openssl req -new -key dev-user.key -out dev-user.csr -subj "/CN=dev-user/O=dev-team"
    ```

---

## 3) CertificateSigningRequest 리소스 생성

CSR 파일을 base64로 인코딩해서 K8s에 제출합니다.

=== "Windows PowerShell"
    CSR을 base64로 인코딩:
    ```powershell
    $CSR_B64 = [Convert]::ToBase64String([System.IO.File]::ReadAllBytes("$HOME/rbac-lab/dev-user.csr"))
    ```

    YAML 파일 생성:
    ```powershell
    @"
    apiVersion: certificates.k8s.io/v1
    kind: CertificateSigningRequest
    metadata:
      name: dev-user-csr
    spec:
      request: $CSR_B64
      signerName: kubernetes.io/kube-apiserver-client
      expirationSeconds: 86400
      usages:
      - client auth
    "@ | Set-Content csr.yaml
    ```

=== "macOS/Linux"
    CSR을 base64로 인코딩 후 YAML 생성:
    ```bash
    cat <<EOF > csr.yaml
    apiVersion: certificates.k8s.io/v1
    kind: CertificateSigningRequest
    metadata:
      name: dev-user-csr
    spec:
      request: $(cat dev-user.csr | base64 | tr -d '\n')
      signerName: kubernetes.io/kube-apiserver-client
      expirationSeconds: 86400
      usages:
      - client auth
    EOF
    ```

CSR 제출:

```bash
kubectl apply -f csr.yaml
kubectl get csr
```

예상 출력:
```
NAME           AGE   SIGNERNAME                            REQUESTOR          CONDITION
dev-user-csr   5s    kubernetes.io/kube-apiserver-client   rancher-desktop    Pending
```

---

## 4) CSR 승인 (관리자 역할)

```bash
kubectl certificate approve dev-user-csr
kubectl get csr
```

`CONDITION`이 `Approved,Issued`로 바뀌면 성공입니다.

---

## 5) 인증서 추출

승인된 인증서를 파일로 저장합니다.

=== "Windows PowerShell"
    ```powershell
    $CERT = kubectl get csr dev-user-csr -o jsonpath='{.status.certificate}'
    [System.IO.File]::WriteAllBytes(
      "$HOME/rbac-lab/dev-user.crt",
      [Convert]::FromBase64String($CERT)
    )
    ```
=== "macOS/Linux"
    ```bash
    kubectl get csr dev-user-csr -o jsonpath='{.status.certificate}' | base64 -d > dev-user.crt
    ```

인증서 내용 확인:

=== "Windows PowerShell"
    ```powershell
    & "C:\Program Files\Git\usr\bin\openssl.exe" x509 -in dev-user.crt -noout -text | Select-String "Subject:"
    ```
=== "macOS/Linux"
    ```bash
    openssl x509 -in dev-user.crt -noout -text | grep Subject:
    ```

`Subject: CN=dev-user, O=dev-team`이 보이면 성공입니다.

---

## 6) kubeconfig에 dev-user 등록

```bash
kubectl config set-credentials dev-user \
  --client-certificate=$HOME/rbac-lab/dev-user.crt \
  --client-key=$HOME/rbac-lab/dev-user.key
```

현재 클러스터 이름 확인 후 context 생성:

```bash
kubectl config get-clusters
```

=== "Windows PowerShell"
    ```powershell
    kubectl config set-credentials dev-user `
      --client-certificate="$HOME/rbac-lab/dev-user.crt" `
      --client-key="$HOME/rbac-lab/dev-user.key"

    kubectl config set-context dev-context `
      --cluster=rancher-desktop `
      --user=dev-user
    ```
=== "macOS/Linux"
    ```bash
    kubectl config set-credentials dev-user \
      --client-certificate=$HOME/rbac-lab/dev-user.crt \
      --client-key=$HOME/rbac-lab/dev-user.key

    kubectl config set-context dev-context \
      --cluster=rancher-desktop \
      --user=dev-user
    ```

---

## 7) dev-user로 전환 & 권한 확인

```bash
kubectl config use-context dev-context
kubectl config current-context
```

Pod 조회 시도:

```bash
kubectl get pods
```

예상 출력:
```
Error from server (Forbidden): pods is forbidden:
User "dev-user" cannot list resource "pods" in API group "" in the namespace "default"
```

!!! success "의도된 결과입니다"
    `dev-user`에게 아직 아무 권한도 없기 때문입니다.
    다음 실습(RBAC 02)에서 Role과 RoleBinding으로 권한을 부여합니다.

관리자 context로 복귀:

```bash
kubectl config use-context rancher-desktop
```

---

## context 전환 치트시트

| 명령어 | 설명 |
|--------|------|
| `kubectl config get-contexts` | 등록된 context 목록 |
| `kubectl config use-context dev-context` | dev-user로 전환 |
| `kubectl config use-context rancher-desktop` | 관리자로 복귀 |
| `kubectl config current-context` | 현재 context 확인 |
| `kubectl auth can-i get pods` | 현재 유저의 권한 확인 |
