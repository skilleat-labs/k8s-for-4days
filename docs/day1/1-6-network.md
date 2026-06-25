# 1-6. 도커 네트워크 (Docker Network) 실습

## 실습 목표

- Docker 네트워크 드라이버의 종류와 특징 이해하기
- 포트 바인딩과 사용자 정의 네트워크 생성/관리 실습하기
- 컨테이너 간 통신과 네트워크 트러블슈팅 체험하기

## 예상 소요 시간 (약 45분)

- 0~8분: 네트워크 개요와 기본 네트워크 확인
- 8~16분: bridge 네트워크 한계 체험
- 16~22분: host / none 네트워크 실습
- 22~32분: 포트 바인딩 완전 실습
- 32~40분: 사용자 정의 네트워크 및 컨테이너 간 통신
- 40~45분: 트러블슈팅 및 정리

## 사전 조건

- Docker 설치 완료
- 기본 컨테이너 실행 경험

확인:

```bash
docker --version
docker network ls
```

---

## 1) 도커 네트워크 개요

### 1-1. 기본 네트워크 확인

```bash
docker network ls
```

| 네트워크 | 드라이버 | 설명 |
|---------|---------|------|
| `bridge` | bridge | 기본 브리지 네트워크 |
| `host` | host | 호스트 네트워크 직접 사용 |
| `none` | null | 네트워크 격리 |

### 1-2. 네트워크 드라이버 종류

- **bridge**: 컨테이너 간 격리된 네트워크 (기본)
- **host**: 호스트 네트워크 공유 (성능 좋음, 보안 낮음)
- **none**: 네트워크 연결 없음 (완전 격리)
- **overlay**: 여러 호스트에 걸친 컨테이너 통신 (Swarm/K8s)
- **macvlan**: 컨테이너에 실제 MAC 주소 부여 (Linux 서버 환경)

---

## 2) bridge 네트워크 한계 체험

### 2-1. 기본 bridge 네트워크로 컨테이너 실행

```bash
docker run -d --name web1 -p 8081:80 nginx:alpine
docker run -d --name web2 -p 8082:80 nginx:alpine

docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web1
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web2
```

### 2-2. 컨테이너 간 통신 시도

```bash
# 이름으로 ping → 실패 (기본 bridge는 DNS 해석 안됨)
docker exec web1 ping -c 3 web2

# IP로 직접 시도 → 성공
docker exec web1 ping -c 3 <web2_IP>
```

---

## 3) host와 none 네트워크 실습

### 3-1. host 네트워크

!!! warning "사전 점검"
    호스트에 nginx 등이 이미 80 포트를 사용 중이면 컨테이너가 즉시 종료됩니다.
    ```bash
    ss -tlnp | grep :80
    sudo systemctl stop nginx
    ```

```bash
docker run -d --name host-test --network host nginx:alpine
curl http://localhost
docker exec host-test ip addr
```

### 3-2. none 네트워크

```bash
docker run -d --name none-test --network none alpine:latest sleep 300
docker exec none-test ip addr
docker exec none-test ping -c 3 8.8.8.8
```

관찰 포인트:

- `ip addr` 결과에 `lo`(127.0.0.1)만 존재 — `eth0` 없음
- 외부 통신 전혀 불가 (`ping: sendto: Network unreachable`)

### 3-3. 정리

```bash
docker rm -f host-test none-test
```

---

## 4) 포트 바인딩 완전 실습

### 4-1. -p 옵션 문법

```bash
# 0.0.0.0 — 모든 인터페이스
docker run -d --name port-test1 -p 8083:80 nginx:alpine

# localhost만 바인딩
docker run -d --name port-test2 -p 127.0.0.1:8084:80 nginx:alpine

curl http://localhost:8083    # 성공
curl http://<VM_IP>:8083      # 성공 (외부 접근 가능)

curl http://localhost:8084    # 성공
curl http://<VM_IP>:8084      # 실패 (127.0.0.1만 바인딩)
```

### 4-2. -P 옵션 (자동 포트 할당)

```bash
docker run -d --name port-auto -P nginx:alpine
docker port port-auto
```

### 4-3. EXPOSE 명령어의 역할

```bash
mkdir ~/expose-test && cd ~/expose-test

cat > Dockerfile <<'EOF'
FROM alpine:latest
RUN apk add --no-cache busybox-extras
RUN echo "Hello from EXPOSE test" > /tmp/index.html
EXPOSE 80
CMD ["httpd", "-f", "-p", "80", "-h", "/tmp"]
EOF

docker build -t expose-test .

# 실험 1: EXPOSE만, -p 없이 → 외부 접근 불가
docker run -d --name test-no-p expose-test
curl http://localhost:80      # 실패
docker port test-no-p         # 아무것도 없음
docker rm -f test-no-p

# 실험 2: -p 붙여서 실행 → 성공
docker run -d --name test-with-p -p 8085:80 expose-test
curl http://localhost:8085    # 성공
docker rm -f test-with-p
```

!!! info "EXPOSE의 역할"
    - EXPOSE만으로는 외부 접근 불가 — 호스트 포트 바인딩이 일어나지 않음
    - `-p`가 있어야 실제 연결됨
    - EXPOSE는 강제사항이 아닌 **문서화 용도**

---

## 5) 사용자 정의 네트워크

### 5-1. 네트워크 생성

```bash
docker network create --driver bridge my-network
docker network ls
docker network inspect my-network
```

### 5-2. 사용자 정의 네트워크로 컨테이너 실행

```bash
docker run -d --name app1 --network my-network nginx:alpine
docker run -d --name app2 --network my-network nginx:alpine
```

### 5-3. 컨테이너 간 이름으로 통신

```bash
docker exec app1 ping -c 3 app2    # 이름으로 성공!
docker exec app1 curl http://app2
```

### 5-4. 네트워크 연결/해제

```bash
docker network connect my-network port-test1
docker network inspect my-network
docker exec app1 curl http://port-test1
docker network disconnect my-network port-test1
```

### 5-5. 정리

```bash
docker rm -f app1 app2 port-test1 port-test2 port-auto
docker network rm my-network
```

---

## 6) 컨테이너 간 통신 구조 심화

### 6-1. 내부 DNS와 서비스 디스커버리

```bash
docker network create my-network
docker run -d --name web --network my-network nginx:alpine
docker run -d --name api --network my-network nginx:alpine
docker run -d --name db  --network my-network alpine:latest sleep 300

docker exec web nslookup api
docker exec web nslookup db
docker exec web curl http://api
```

### 6-2. 네트워크 격리 확인

```bash
docker network create isolated-network
docker run -d --name isolated --network isolated-network nginx:alpine

docker exec web curl http://isolated  # 실패 (서로 다른 네트워크)
```

---

## 7) 네트워크 트러블슈팅

### 7-1. 네트워크 상태 확인

```bash
docker network ls
docker network inspect my-network
docker inspect web | grep -A 20 "NetworkSettings"
```

### 7-2. 컨테이너 상태 점검

```bash
docker ps -a
docker logs web
docker exec web ip addr
docker exec web ip route
docker exec web cat /etc/resolv.conf
```

### 7-3. 통신 테스트

```bash
docker exec web nslookup api
docker exec web nc -zv api 80
docker port web
```

### 7-4. 일반적인 문제 해결

**문제 1: 컨테이너 간 통신 안됨**

```bash
docker network connect my-network container-name
```

**문제 2: 포트 바인딩 실패**

```bash
netstat -tulpn | grep :8080
docker ps | grep 8080
```

**문제 3: DNS 해석 실패**

```bash
docker network disconnect my-network container-name
docker network connect my-network container-name
```

---

## 8) 정리

```bash
docker rm -f $(docker ps -aq) 2>/dev/null || true
docker network rm my-network isolated-network 2>/dev/null || true
docker network ls
```

### 네트워크 선택 가이드

| 네트워크 타입 | 사용 시기 | 장점 | 단점 |
|---------------|-----------|------|------|
| **기본 bridge** | 간단한 단일 호스트 | 자동 생성 | 이름 통신 불가 |
| **사용자 정의 bridge** | 다중 컨테이너 앱 | DNS 지원, 격리 | 설정 필요 |
| **host** | 최고 성능 필요 | 빠름 | 보안 낮음 |
| **none** | 완전 격리 | 보안 높음 | 통신 불가 |
| **overlay** | Swarm/K8s | 멀티 호스트 | Swarm 필요 |
