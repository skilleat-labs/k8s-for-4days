# 2-4. Namespace 실습

## 실습 목표

- Namespace의 역할과 필요성을 이해한다.
- 명령어와 YAML로 Namespace를 생성하고 관리한다.
- 기본 컨텍스트 Namespace를 변경하는 방법을 익힌다.
- Namespace 삭제 시 내부 리소스가 함께 삭제됨을 확인한다.

---

## Namespace란?

Kubernetes 클러스터 안에서 리소스를 **논리적으로 격리**하는 단위입니다.

```
클러스터
├── default          ← 아무 설정 없이 배포하면 여기 들어감
├── kube-system      ← K8s 시스템 컴포넌트
├── dev              ← 개발 환경
└── prod             ← 운영 환경
```

- 같은 Namespace 안에서는 Service 이름으로 서로 통신 가능
- 다른 Namespace의 리소스는 기본적으로 격리됨
- Namespace를 삭제하면 그 안의 **모든 리소스가 함께 삭제**됨

---

## 1) 기본 Namespace 확인

```bash
kubectl get namespaces    # 약어: kubectl get ns
```

출력 예시:

```
NAME              STATUS   AGE
default           Active   5d
kube-system       Active   5d
kube-public       Active   5d
kube-node-lease   Active   5d
```

| Namespace | 용도 |
|---|---|
| `default` | 아무 Namespace 지정 없이 생성하면 여기 배포됨 |
| `kube-system` | CoreDNS, metrics-server 등 시스템 컴포넌트 |
| `kube-public` | 인증 없이 읽을 수 있는 공개 리소스 |
| `kube-node-lease` | 노드 heartbeat 리소스 |

특정 Namespace의 리소스를 보려면 `-n` 플래그를 사용합니다.

```bash
kubectl get pods -n kube-system
```

---

## 2) 명령어로 Namespace 생성

```bash
kubectl create namespace dev
kubectl get ns              # dev 생성 확인
```

Namespace 안에 Pod를 배포하려면 `-n` 플래그를 사용합니다.

```bash
kubectl run nginx-dev --image=nginx:1.25 -n dev
kubectl get pods -n dev
```

모든 Namespace의 Pod를 한꺼번에 보려면 `--all-namespaces`(약어 `-A`)를 사용합니다.

```bash
kubectl get pods -A
```

---

## 3) YAML로 Namespace 생성

`namespace-prod.yaml` 파일을 만듭니다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    env: production
```

```bash
kubectl apply -f namespace-prod.yaml
kubectl get ns prod
```

Pod도 YAML 안에 `namespace`를 명시해서 배포할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-prod
  namespace: prod
spec:
  containers:
    - name: nginx
      image: nginx:1.25
```

```bash
kubectl apply -f namespace-prod.yaml   # 이미 있어도 오류 없음
kubectl get pods -n prod
```

---

## 4) 기본 Namespace 전환

매번 `-n dev`를 붙이는 대신 기본 Namespace를 바꿀 수 있습니다.

```bash
kubectl config set-context --current --namespace=dev
kubectl config view --minify | grep namespace    # 현재 기본 Namespace 확인
```

전환 후에는 `-n` 없이도 `dev` Namespace에 명령이 적용됩니다.

```bash
kubectl get pods    # dev Namespace의 Pod가 출력됨
```

실습이 끝나면 **반드시 `default`로 복원**합니다.

```bash
kubectl config set-context --current --namespace=default
kubectl get pods    # default Namespace의 Pod가 출력됨
```

!!! warning "Namespace 전환 후 꼭 복원하세요"
    기본 Namespace를 바꾼 채로 두면 이후 실습에서 리소스가 엉뚱한 Namespace에 배포될 수 있습니다.
    실습 후 반드시 `default`로 되돌리세요.

---

## 5) Namespace 삭제 — 내부 리소스 함께 삭제

Namespace를 삭제하면 그 안의 **모든 리소스(Pod, Service, Deployment 등)가 함께 삭제**됩니다.

먼저 dev Namespace 안의 리소스를 확인합니다.

```bash
kubectl get all -n dev
```

Namespace를 삭제합니다.

```bash
kubectl delete namespace dev
```

삭제 후 리소스가 사라졌는지 확인합니다.

```bash
kubectl get pods -n dev    # Error from server (NotFound) 출력
```

!!! danger "Namespace 삭제는 되돌릴 수 없습니다"
    Namespace 삭제 시 내부의 모든 리소스가 즉시 삭제됩니다.
    운영 환경에서 실수로 삭제하지 않도록 주의하세요.

---

## 정리

```bash
kubectl delete namespace prod
kubectl get ns    # dev, prod 모두 삭제 확인
```

---

## Namespace 명령어 정리

| 명령어 | 설명 |
|---|---|
| `kubectl get ns` | Namespace 목록 조회 |
| `kubectl create ns <이름>` | Namespace 생성 |
| `kubectl get pods -n <이름>` | 특정 Namespace의 Pod 조회 |
| `kubectl get pods -A` | 전체 Namespace의 Pod 조회 |
| `kubectl config set-context --current --namespace=<이름>` | 기본 Namespace 변경 |
| `kubectl delete namespace <이름>` | Namespace 삭제 (내부 리소스 포함) |

## 트러블슈팅

| 증상 | 확인 사항 |
|---|---|
| Pod가 보이지 않음 | 현재 기본 Namespace 확인 (`kubectl config view --minify`) |
| Service 이름으로 통신 안 됨 | 같은 Namespace인지 확인 |
| Namespace 삭제 후 `Terminating` 상태 지속 | finalizer가 걸린 리소스 존재 — `kubectl get all -n <이름>` 으로 확인 |
