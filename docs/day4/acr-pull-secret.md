# Private Registry — ACR 이미지 Pull Secret

## 실습 목표

- Public 이미지(Docker Hub)와 Private 이미지(ACR)의 차이를 체감한다.
- `docker-registry` 타입 Secret으로 ACR 인증 정보를 저장한다.
- `imagePullSecrets`를 사용해 Private Registry에서 이미지를 가져온다.

---

## 왜 Secret이 필요한가?

Docker Hub의 public 이미지는 인증 없이 누구나 pull할 수 있습니다.

```yaml
# 인증 불필요 — 그냥 됨
image: nginx:1.25
image: busybox:1.36
```

하지만 Private Registry(ACR, ECR, GCR 등)는 **인증 없이 pull하면 실패**합니다.

```yaml
# 인증 필요 — Secret 없으면 ImagePullBackOff 발생
image: myregistry.azurecr.io/myapp:v1
```

쿠버네티스는 Registry 인증 정보를 **`docker-registry` 타입 Secret**으로 저장하고,
Pod에 `imagePullSecrets`로 연결합니다.

---

## 실습 1 — Secret 없이 배포 (실패 확인)

먼저 Secret 없이 배포해서 어떤 에러가 나는지 확인합니다.

`pod-no-secret.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-no-secret
spec:
  containers:
    - name: app
      image: <ACR이름>.azurecr.io/skilleat/rollout-demo:v1.0.0
  restartPolicy: Never
```

```bash
kubectl apply -f pod-no-secret.yaml
kubectl get pod pod-no-secret
kubectl describe pod pod-no-secret
```

Events에서 아래 메시지를 확인합니다:

```
Failed to pull image "...": unauthorized: authentication required
```

```
  Warning  Failed  ErrImagePull       unauthorized: authentication required
  Warning  Failed  ImagePullBackOff   Back-off pulling image
```

!!! info "ImagePullBackOff란?"
    인증 실패 또는 이미지가 존재하지 않을 때 발생합니다.
    쿠버네티스가 재시도 간격을 점점 늘려가며 반복 시도합니다.

```bash
kubectl delete pod pod-no-secret
```

---

## 실습 2 — imagePullSecret 생성

강사에게 전달받은 ACR 인증 정보로 Secret을 생성합니다.

```bash
kubectl create secret docker-registry acr-secret \
  --docker-server=<ACR이름>.azurecr.io \
  --docker-username=<username> \
  --docker-password=<password>
```

Secret 확인:

```bash
kubectl get secret acr-secret
kubectl describe secret acr-secret
```

출력 예시:

```
Name:         acr-secret
Namespace:    default
Type:         kubernetes.io/dockerconfigjson

Data
====
.dockerconfigjson:  173 bytes
```

!!! info "Secret 내부 구조"
    `kubectl create secret docker-registry`는 내부적으로 `.dockerconfigjson` 키에
    base64로 인코딩된 인증 정보를 저장합니다.
    `kubectl describe`에서는 값이 보이지 않습니다.

실제 저장된 값 확인 (학습 목적):

```bash
kubectl get secret acr-secret -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
```

---

## 실습 3 — imagePullSecrets로 Pod 배포

`pod-with-secret.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-secret
spec:
  containers:
    - name: app
      image: <ACR이름>.azurecr.io/skilleat/rollout-demo:v1.0.0
      ports:
        - containerPort: 8080
  imagePullSecrets:
    - name: acr-secret
  restartPolicy: Never
```

```bash
kubectl apply -f pod-with-secret.yaml
kubectl get pod pod-with-secret -w
```

```
NAME              READY   STATUS              AGE
pod-with-secret   0/1     ContainerCreating   3s
pod-with-secret   1/1     Running             8s   ← 성공!
```

!!! success "확인 포인트"
    `ContainerCreating` → `Running`으로 전환되면 인증 성공입니다.
    Secret 없이 배포했을 때의 `ImagePullBackOff`와 비교해보세요.

```bash
kubectl delete pod pod-with-secret
```

---

## 실습 4 — Deployment에 imagePullSecrets 적용

실제 운영에서는 Pod 단독이 아니라 Deployment에 설정합니다.

`deploy-acr.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: acr-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: acr-app
  template:
    metadata:
      labels:
        app: acr-app
    spec:
      containers:
        - name: app
          image: <ACR이름>.azurecr.io/skilleat/rollout-demo:v1.0.0
          ports:
            - containerPort: 8080
      imagePullSecrets:
        - name: acr-secret
```

```bash
kubectl apply -f deploy-acr.yaml
kubectl get pods -l app=acr-app
kubectl get pods -l app=acr-app -o wide
```

2개의 Pod가 서로 다른 노드에서 이미지를 pull해 실행되는지 확인합니다.

!!! info "노드별 이미지 캐시"
    각 노드가 처음 이미지를 pull할 때 Secret을 사용합니다.
    같은 노드에서 두 번째 이후는 캐시에서 가져오므로 Secret 없이도 됩니다.
    하지만 새 노드가 추가되거나 이미지 태그가 바뀌면 다시 pull하므로
    항상 Secret을 유지해야 합니다.

```bash
kubectl delete -f deploy-acr.yaml
```

---

## 정리 (리소스 삭제)

```bash
kubectl delete secret acr-secret
```

---

## Secret vs imagePullSecrets 구조 정리

```
Secret (docker-registry 타입)
  └── .dockerconfigjson
        └── { "auths": { "registry주소": { "username", "password" } } }

Pod / Deployment
  └── spec.imagePullSecrets
        └── - name: acr-secret   ← Secret 이름 참조
```

---

## 비교 정리

| 항목 | Public Registry | Private Registry (ACR) |
|------|----------------|----------------------|
| 인증 필요 | 없음 | 있음 |
| Secret 필요 | 없음 | `docker-registry` 타입 Secret |
| 실패 시 증상 | — | `ImagePullBackOff` |
| imagePullSecrets | 불필요 | Pod/Deployment에 필수 |

---

## 트러블슈팅

| 증상 | 확인 사항 |
|------|---------|
| `ImagePullBackOff` | `kubectl describe pod`에서 Events 확인 — 인증 실패인지, 이미지 이름 오타인지 구분 |
| `unauthorized` | Secret의 username/password가 올바른지 확인 |
| `image not found` | ACR에 해당 이미지/태그가 실제로 존재하는지 확인 |
| Secret 만들었는데도 실패 | `imagePullSecrets.name`이 Secret 이름과 정확히 일치하는지 확인 |
| 특정 노드에서만 실패 | 해당 노드에서 Secret이 존재하는 namespace인지 확인 |
