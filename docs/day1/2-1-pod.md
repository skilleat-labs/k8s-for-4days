# 1-9. Pod 실습

## 실습 목표

- `kubectl run`과 YAML로 Pod를 생성하고 차이를 이해한다.
- 멀티 컨테이너 Pod 구조를 실습한다.
- ReplicaSet이 Pod를 자동으로 복구하는 동작을 확인한다.
- Label의 역할과 `--show-labels` 옵션을 익힌다.
- Deployment로 롤링 업데이트와 롤백을 실습한다.

## 공식 문서 참고

- [Pod 공식 문서](https://kubernetes.io/docs/concepts/workloads/pods/){target=_blank}
- [Deployment 공식 문서](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/){target=_blank}
- [kubectl 치트시트](https://kubernetes.io/docs/reference/kubectl/cheatsheet/){target=_blank}

---

## 1) kubectl run으로 Pod 빠르게 생성

```bash
kubectl run nginx-pod --image=nginx:1.25
kubectl get pods
kubectl get pods -o wide
```

### dry-run으로 YAML 미리 보기

```bash
kubectl run test --image=nginx --dry-run=client -o yaml
kubectl run test --image=nginx --dry-run=client -o yaml > pod-test.yaml
```

---

## 2) YAML로 Pod 생성

`pod-nginx.yaml` 파일:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    env: lab
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

배포 및 확인:

```bash
kubectl apply -f pod-nginx.yaml
kubectl get pods
kubectl describe pod nginx-pod
```

### 로그 확인

```bash
kubectl logs nginx-pod
kubectl logs nginx-pod -f
```

### Pod 내부 접속

```bash
kubectl exec -it nginx-pod -- bash
nginx -v
curl localhost
exit
```

### curl 이미지로 Pod 간 통신 확인

```bash
kubectl get pods -o wide    # nginx-pod의 IP 확인

kubectl run curl-test --image=curlimages/curl:latest --restart=Never -it --rm \
  -- curl http://<nginx-pod-IP>
```

### Pod 삭제 후 IP 변화 관찰

```bash
kubectl delete pod nginx-pod
kubectl apply -f pod-nginx.yaml
kubectl get pods -o wide    # 새 IP가 할당됨을 확인
```

!!! tip "포인트"
    Pod를 재생성하면 IP가 바뀝니다. 그래서 Service가 필요합니다.

---

## 3) 멀티 컨테이너 Pod

`pod-multi.yaml` 파일:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
  labels:
    app: multi
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
    - name: sidecar
      image: curlimages/curl:latest
      command: ["sh", "-c", "while true; do echo $(date) $(curl -s localhost); sleep 5; done"]
```

배포 및 확인:

```bash
kubectl apply -f pod-multi.yaml
kubectl get pods    # READY 2/2 확인

kubectl logs multi-pod -c nginx
kubectl logs multi-pod -c sidecar
kubectl exec -it multi-pod -c nginx -- bash
```

---

## 4) Label과 --show-labels

```bash
kubectl get pods --show-labels
kubectl get pods -l app=nginx
kubectl get pods -l env=lab
kubectl get pods -l app=nginx,env=lab

kubectl label pod nginx-pod version=v1
kubectl label pod nginx-pod env=production --overwrite
kubectl label pod nginx-pod version-
kubectl get pods --show-labels
```

---

## 5) ReplicaSet 실습

`replicaset.yaml` 파일:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-rs
  template:
    metadata:
      labels:
        app: nginx-rs
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f replicaset.yaml
kubectl get replicasets
kubectl get pods --show-labels
```

### Pod 삭제 후 자동 복구 확인

```bash
kubectl get pods -l app=nginx-rs
kubectl delete pod <pod이름 중 하나>
kubectl get pods -l app=nginx-rs -w    # 즉시 새 Pod가 생성되는 것을 실시간 확인
```

`Ctrl+C`로 watch 모드 종료.

### Label을 바꾸면 ReplicaSet 관리에서 빠짐

```bash
kubectl label pod <pod이름> app=orphan --overwrite
kubectl get pods --show-labels    # Pod가 4개가 됨
kubectl delete pod -l app=orphan
```

---

## 6) Deployment 생성

```bash
kubectl create deployment rollout-deploy \
  --image=skilleat/rollout-demo:v1.0.0 \
  --replicas=3 \
  --dry-run=client -o yaml > deploy-rollout.yaml
```

`deploy-rollout.yaml` 완성본:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rollout-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rollout
  template:
    metadata:
      labels:
        app: rollout
    spec:
      containers:
        - name: app
          image: skilleat/rollout-demo:v1.0.0
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deploy-rollout.yaml
kubectl get deployments
kubectl get replicasets
kubectl get pods --show-labels
```

port-forward로 현재 버전 확인 (**파란색** 페이지):

```bash
kubectl port-forward deployment/rollout-deploy 8080:80
```

`http://localhost:8080` 접속 후 `Ctrl+C`로 종료.

---

## 7) 롤링 업데이트

v2.0.0으로 업데이트합니다 (**초록색** 페이지).

```bash
kubectl set image deployment/rollout-deploy app=skilleat/rollout-demo:v2.0.0
kubectl rollout status deployment/rollout-deploy
kubectl get pods -w
```

완료 후 확인:

```bash
kubectl port-forward deployment/rollout-deploy 8080:80
kubectl describe deployment rollout-deploy | grep Image
```

---

## 8) 롤백

```bash
kubectl rollout history deployment/rollout-deploy
kubectl rollout undo deployment/rollout-deploy
kubectl rollout status deployment/rollout-deploy
kubectl describe deployment rollout-deploy | grep Image
```

---

## 9) 스케일 조정

```bash
kubectl scale deployment rollout-deploy --replicas=5
kubectl get pods
kubectl scale deployment rollout-deploy --replicas=2
kubectl get pods
```

---

## 정리 (리소스 삭제)

```bash
kubectl delete -f replicaset.yaml
kubectl delete -f deploy-rollout.yaml
kubectl delete -f pod-nginx.yaml
kubectl delete -f pod-multi.yaml
```

---

## 핵심 정리

| 항목 | Pod 단독 | ReplicaSet | Deployment |
|------|---------|-----------|-----------|
| 복제본 관리 | 수동 | 자동 | 자동 |
| 장애 복구 | 없음 | 자동 재생성 | 자동 재생성 |
| 롤링 업데이트 | 불가 | 불가 | 내장 지원 |
| 실사용 권장 | 디버깅용 | 거의 직접 사용 안 함 | 프로덕션 배포 |

---

## 트러블슈팅

| 증상 | 확인 사항 |
|------|---------|
| Pod `ImagePullBackOff` | 이미지 이름/태그 오타 또는 네트워크 문제 |
| Pod `Pending` | 노드 리소스 부족 (`kubectl describe pod` 이벤트 확인) |
| Pod `CrashLoopBackOff` | 컨테이너 실행 오류 (`kubectl logs <pod>` 확인) |
| READY `2/2` 안됨 | `kubectl describe pod`에서 각 컨테이너 상태 확인 |
