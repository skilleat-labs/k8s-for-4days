# 1-2. Docker 기본 명령어와 동작 원리

## 실습 목표

- Docker 명령어 문법 구조 이해
- ubuntu 컨테이너로 격리된 프로세스 직접 체감
- nginx 컨테이너 로그를 실시간으로 관찰
- redis 컨테이너로 "웹서버만이 아니다"를 경험
- stop, start, restart, rm 으로 컨테이너 생명주기 이해

---

## 1) Docker 명령어 문법 구조

기본 형태:

```
docker <명령> [옵션] [대상]
```

| 요소 | 예시 | 의미 |
|------|------|------|
| 명령 | `run`, `ps`, `exec`, `logs`, `stop`, `rm` | 수행할 동작 |
| 옵션 | `-d`, `--name`, `-p`, `-it` | 동작 방식 설정 |
| 대상 | `nginx`, `ubuntu`, `web1` | 이미지 또는 컨테이너 이름 |

**자주 쓰는 옵션:**

| 옵션 | 의미 |
|------|------|
| `-d` | 백그라운드 실행 (detached) |
| `--name` | 컨테이너 이름 지정 |
| `-p 호스트포트:컨테이너포트` | 포트 연결 |
| `-it` | 터미널 입출력 연결 (interactive + tty) |

!!! info "-p 옵션 이해하기"
    `-p 8082:80` 이라고 쓰면, 컨테이너 포트(오른쪽: 80)는 컨테이너 안에서 앱이 실제로 열고 있는 포트이고, 호스트 포트(왼쪽: 8082)는 내 PC(또는 VM)에서 접근할 포트입니다.

---

## 2) ubuntu 컨테이너 — 격리된 프로세스 체감

### 컨테이너 실행

```bash
docker run -it --name shell-test ubuntu:24.04 bash
```

접속하면 프롬프트가 컨테이너 ID로 바뀝니다:

```
root@a1b2c3d4e5f6:/#
```

### 격리 확인

컨테이너 안에서:

```bash
# 호스트명 확인 — 컨테이너 ID가 출력됨
hostname

# 프로세스 목록 — bash 하나뿐, VM의 프로세스는 안 보임
ps aux

# 파일 시스템 — VM과 완전히 분리된 독립 환경
ls /

# 패키지 설치 (VM에는 영향 없음)
apt update && apt install -y curl
curl --version
```

!!! tip "핵심 포인트"
    VM에 curl이 없어도 이 컨테이너 안에는 설치됩니다. 컨테이너를 삭제하면 설치한 모든 것이 사라집니다.

### 컨테이너 삭제 후 재실행

```bash
docker rm shell-test
docker run -it --name shell-test ubuntu:24.04 bash
curl --version
```

결과: `curl: command not found` — 이미지는 변하지 않았으므로 설치한 내용은 사라졌습니다.

```bash
exit
```

---

## 3) nginx 컨테이너 — 실시간 로그 관찰

### 컨테이너 실행

```bash
docker run -d --name web1 -p 8082:80 nginx:latest
docker ps
```

### 실시간 로그 보기

**터미널 1** — 로그 실시간 추적:

```bash
docker logs -f web1
```

**브라우저** — 요청 보내기:

```
http://192.168.56.10:8082
http://192.168.56.10:8082/없는페이지
```

**로그 해석:**

```
192.168.56.1 - - [27/Apr/2026] "GET / HTTP/1.1" 200 615
192.168.56.1 - - [27/Apr/2026] "GET /없는페이지 HTTP/1.1" 404 153
```

- `200` → 정상 응답
- `404` → 없는 페이지 요청

로그 추적 종료: `Ctrl + C`

---

## 4) redis 컨테이너 — DB 타입 컨테이너

### 컨테이너 실행

```bash
docker run -d --name myredis redis:alpine
docker ps
```

### redis-cli로 데이터 저장/조회

```bash
docker exec -it myredis redis-cli
```

redis 프롬프트가 열립니다:

```
127.0.0.1:6379>
```

데이터 작업:

```bash
SET name "docker-lab"
GET name
SET count 1
INCR count
GET count
EXIT
```

| 명령어 | 의미 |
|--------|------|
| `SET name "docker-lab"` | `name` 이라는 키에 값을 저장 |
| `GET name` | `name` 키에 저장된 값을 조회 |
| `SET count 1` | `count` 키에 숫자 `1` 저장 |
| `INCR count` | `count` 값을 1 증가 (1 → 2) |
| `EXIT` | redis-cli 종료 |

### redis 컨테이너 삭제 후 데이터 확인

```bash
docker rm -f myredis
docker run -d --name myredis redis:alpine
docker exec -it myredis redis-cli
GET name
```

결과: `(nil)` — 컨테이너가 삭제되면서 데이터도 사라졌습니다.

```bash
EXIT
```

---

## 5) 컨테이너 생명주기 — stop / start / restart / rm

```bash
# 중지
docker stop web1
docker ps        # web1 사라짐
docker ps -a     # web1 존재는 함 (Exited 상태)

# 재시작 (컨테이너 재활용)
docker start web1
docker ps        # web1 다시 실행 중

# 재시작 (실행 중 상태에서)
docker restart web1

# 삭제 (중지 후 삭제)
docker stop web1
docker rm web1

# 강제 삭제 (실행 중이어도)
docker rm -f myredis

# 전체 확인
docker ps -a
```

!!! info "stop vs rm"
    - `stop`: 컨테이너를 멈추지만 남겨둠 (start로 재활용 가능)
    - `rm`: 컨테이너를 완전히 삭제 (이미지는 남아있음)

---

## 6) 도전 과제: 버전 교체 체험

이미지: `skilleat/rollout-demo` (v1.0.0 / v2.0.0 / v3.0.0)
컨테이너 포트: `8080`

| 태그 | 배경색 |
|------|--------|
| v1.0.0 | 파란색 |
| v2.0.0 | 초록색 |
| v3.0.0 | 주황색 |

### 과제 1. 버전 교체

- `8083` 포트로 v1을 실행해 브라우저에서 **파란색** 페이지를 확인합니다.
- 같은 포트(`8083`)에서 v2로 교체해 **초록색**으로 바뀌는 것을 확인합니다.

### 과제 2. 세 버전 동시 실행

- v1 / v2 / v3를 **각각 다른 포트**로 동시에 실행합니다.
- 브라우저 탭 3개에서 파란색 / 초록색 / 주황색이 동시에 보이면 성공입니다.
- 확인 후 실행한 컨테이너를 모두 정리합니다.

---

## 정리

| 명령어 | 용도 |
|--------|------|
| `docker run -it` | 컨테이너에 직접 접속 |
| `docker run -d` | 백그라운드 실행 |
| `docker ps / ps -a` | 실행 중 / 전체 컨테이너 조회 |
| `docker exec -it` | 실행 중 컨테이너에 접속 |
| `docker logs -f` | 실시간 로그 추적 |
| `docker stop / start` | 중지 / 재시작 |
| `docker rm / rm -f` | 삭제 / 강제 삭제 |
