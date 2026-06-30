# 1-3. 레지스트리와 이미지 실습

## 실습 목표

- 공개 레지스트리(Docker Hub)와 프라이빗 레지스트리(ACR)의 차이를 이해하기
- ACR에서 직접 이미지를 pull하며 주소 형식 체험하기
- 이미지 캐시(레이어 재사용)가 무엇인지 실습으로 이해하기

## 예상 소요 시간 (약 35분)

- 0~5분: 레지스트리 개념 이해
- 5~15분: ACR 로그인 및 이미지 pull
- 15~25분: 이미지로 컨테이너 실행
- 25~33분: 캐시/레이어 재사용 체감 실습
- 33~35분: 정리

## 사전 조건

- Docker 설치 완료
- Azure CLI 설치 완료
- 강사로부터 팀 계정 정보 수령

### Azure CLI 설치 (미설치 시)

**Ubuntu/Debian:**

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

**Mac:**

```bash
brew install azure-cli
```

### 설치 확인

```bash
az version
```

---

## 1) 레지스트리란?

1-1, 1-2에서는 `docker run nginx:latest`처럼 이미지 이름만 쓰면 자동으로 **Docker Hub**에서 이미지를 가져왔습니다.

하지만 실제 기업 환경에서는 보안, 속도, 접근 제어 등의 이유로 **프라이빗 레지스트리**를 사용합니다.

### 레지스트리 종류 비교

| 종류 | 예시 | 특징 |
|---|---|---|
| 공개 레지스트리 | Docker Hub (`docker.io`) | 누구나 접근 가능, 공식 이미지 제공 |
| 프라이빗 레지스트리 | Azure ACR, AWS ECR, GitHub GHCR | 인증 필요, 기업 내부 이미지 관리 |

### 주소 형식 비교

```
# Docker Hub (레지스트리 주소 생략 가능)
nginx:latest
docker.io/library/nginx:latest   ← 전체 주소

# 프라이빗 레지스트리 (주소 명시 필수)
skilleatlab.azurecr.io/lab/nginx:latest
│                        │   │
│                        │   └── 태그
│                        └────── 이미지 경로
└─────────────────────────────── 레지스트리 주소
```

**핵심 포인트:** 프라이빗 레지스트리에서 이미지를 가져오려면 레지스트리 주소를 앞에 붙여야 합니다.

---

## 2) ACR 로그인 및 이미지 pull

### 2-1. 팀 계정으로 Azure 로그인

각 팀별 계정 정보:

| 팀 | 이메일 | 비밀번호 |
|-----|--------|---------|
| Team 01 | `student01@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 02 | `student02@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 03 | `student03@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 04 | `student04@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 05 | `student05@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 06 | `student06@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 07 | `student07@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 08 | `student08@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 09 | `student09@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 10 | `student10@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 11 | `student11@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 12 | `student12@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 13 | `student13@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 14 | `student14@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 15 | `student15@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 16 | `student16@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 17 | `student17@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 18 | `student18@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 19 | `student19@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |
| Team 20 | `student20@nrkim0615outlook.onmicrosoft.com` | 강사 제공 |

```bash
az login --use-device-code
```

**예상 출력:**

```
To sign in, use a web browser to open the page https://microsoft.com/devicelogin
and enter the code XXXXXXXX to authenticate.
```

1. 출력된 URL(`https://microsoft.com/devicelogin`)을 브라우저에서 열기
2. 출력된 코드 입력
3. 팀 계정으로 로그인

로그인 확인:

```bash
az account show
```

### 2-2. ACR 로그인

```bash
az acr login --name skilleatlab
```

**예상 출력:**

```
Login Succeeded
```

### 2-3. ACR에서 이미지 pull

```bash
docker pull skilleatlab.azurecr.io/lab/nginx:latest
docker pull skilleatlab.azurecr.io/lab/httpd:latest
docker pull skilleatlab.azurecr.io/lab/redis:latest
docker pull skilleatlab.azurecr.io/lab/alpine:latest
```

목록 확인:

```bash
docker images
```

---

## 3) Docker Hub에서 이미지 찾기

Docker Hub: [https://hub.docker.com/](https://hub.docker.com/)

```bash
docker search nginx
docker search redis
```

---

## 4) 이미지로 컨테이너 실행 실습

### 4-1. Nginx 실행

```bash
docker run -d --name nginx-web -p 8081:80 skilleatlab.azurecr.io/lab/nginx:latest
curl -I http://localhost:8081
```

### 4-2. Apache(httpd) 실행

```bash
docker run -d --name httpd-web -p 8082:80 skilleatlab.azurecr.io/lab/httpd:latest
curl -I http://localhost:8082
```

### 4-3. Redis 실행

```bash
docker run -d --name redis-db -p 6379:6379 skilleatlab.azurecr.io/lab/redis:latest
docker logs redis-db | head
```

### 4-4. Alpine 실행

```bash
docker run --name alpine-test skilleatlab.azurecr.io/lab/alpine:latest echo "hello from alpine"
```

상태 확인:

```bash
docker ps
docker ps -a
```

---

## 5) 컨테이너는 "프로세스"라는 점 이해하기

### 5-1. 바로 종료되는 컨테이너 보기

```bash
docker run --name exit-now skilleatlab.azurecr.io/lab/alpine:latest echo "bye"
```

=== "macOS/Linux"
    ```bash
    docker ps -a | grep exit-now
    ```
=== "Windows PowerShell"
    ```powershell
    docker ps -a | Select-String "exit-now"
    ```

### 5-2. 살아있게 유지하기 (`sleep`)

```bash
docker run -d --name keep-alive skilleatlab.azurecr.io/lab/alpine:latest sleep 600
```

=== "macOS/Linux"
    ```bash
    docker ps | grep keep-alive
    ```
=== "Windows PowerShell"
    ```powershell
    docker ps | Select-String "keep-alive"
    ```

### 5-3. 종료/재시작으로 동작 확인

```bash
docker stop keep-alive
docker start keep-alive
```

=== "macOS/Linux"
    ```bash
    docker ps -a | grep keep-alive
    docker ps | grep keep-alive
    ```
=== "Windows PowerShell"
    ```powershell
    docker ps -a | Select-String "keep-alive"
    docker ps | Select-String "keep-alive"
    ```

### 5-4. 무한 유지 패턴

```bash
docker run -d --name keep-forever skilleatlab.azurecr.io/lab/alpine:latest sh -c "while true; do sleep 60; done"
```

=== "macOS/Linux"
    ```bash
    docker ps | grep keep-forever
    ```
=== "Windows PowerShell"
    ```powershell
    docker ps | Select-String "keep-forever"
    ```

---

## 6) 캐시(레이어 재사용) 체감 실습

!!! note "Docker 버전에 따라 출력이 다릅니다"
    Docker Engine v29 이후 또는 Docker Desktop 4.34 이후 신규 설치는 기본 이미지 저장소가 **containerd image store**로 바뀌었습니다.

    내 환경 확인:
    === "macOS/Linux"
        ```bash
        docker info | grep "Storage Driver"
        ```
    === "Windows PowerShell"
        ```powershell
        docker info | Select-String "Storage Driver"
        ```
    - `overlay2` → 이전 방식 (`Already exists` 보임)
    - `overlayfs` → 새 방식 (라인 자체가 사라짐)

### 6-1. 같은 이미지를 다시 pull

```bash
docker pull skilleatlab.azurecr.io/lab/nginx:latest
```

### 6-2. 레이어 재사용 확인

```bash
docker pull python:3.11-slim
docker pull python:3.12-slim

docker image inspect python:3.11-slim | jq '.[0].RootFS.Layers'
docker image inspect python:3.12-slim | jq '.[0].RootFS.Layers'
```

### 6-3. 같은 이미지로 새 컨테이너 추가 실행

```bash
docker run -d --name nginx-web-2 -p 8083:80 skilleatlab.azurecr.io/lab/nginx:latest
docker ps
```

---

## 7) 실습 정리(삭제)

```bash
docker rm -f nginx-web nginx-web-2 httpd-web redis-db alpine-test 2>/dev/null || true
docker rm -f exit-now keep-alive keep-forever 2>/dev/null || true
docker rmi skilleatlab.azurecr.io/lab/nginx:latest
docker rmi skilleatlab.azurecr.io/lab/httpd:latest
docker rmi skilleatlab.azurecr.io/lab/redis:latest
docker rmi skilleatlab.azurecr.io/lab/alpine:latest
docker rmi python:3.11-slim python:3.12-slim
```
