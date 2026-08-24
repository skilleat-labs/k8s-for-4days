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

## 1) 실습 환경 준비 — VS Code 설치 & 자동완성

### VS Code 설치

[VS Code 공식 다운로드 페이지](https://code.visualstudio.com/){target=_blank}에서 설치 후, 아래 확장을 설치합니다.

| 확장 이름 | 제공 | 역할 |
|-----------|------|------|
| **YAML** | Red Hat | YAML 문법 검사 · 스키마 자동완성 |
| **Kubernetes** | Microsoft | 클러스터 탐색기 · kubectl 팔레트 |

### VS Code 터미널에서 kubectl 자동완성 설정

**kubectl 자동완성**이란, 터미널에서 명령어를 입력하다가 `Tab` 키를 누르면 리소스 이름·옵션·서브커맨드가 자동으로 채워지는 기능입니다.
예를 들어 `kubectl get po`까지 치고 `Tab`을 누르면 실행 중인 Pod 이름이 바로 나타납니다. 긴 이름을 외우거나 직접 타이핑할 필요 없이 빠르게 명령어를 완성할 수 있습니다.

VS Code 내장 터미널(`Ctrl+`` `)을 열고 아래 명령어를 실행합니다.

=== "Windows PowerShell"
    현재 세션 적용:
    ```powershell
    kubectl completion powershell | Out-String | Invoke-Expression
    ```

    영구 적용 (VS Code 터미널 재시작 후에도 유지):
    ```powershell
    New-Item -ItemType File -Force -Path $PROFILE
    kubectl completion powershell >> $PROFILE
    ```

=== "macOS/Linux (bash)"
    현재 세션 적용:
    ```bash
    source <(kubectl completion bash)
    ```

    영구 적용:
    ```bash
    echo 'source <(kubectl completion bash)' >> ~/.bashrc
    source ~/.bashrc
    ```

!!! tip "자동완성 확인"
    `kubectl get po` 입력 후 `Tab`을 눌러 Pod 이름이 자동완성되면 성공입니다.

### VS Code에서 YAML 파일 작업

이후 실습에서 YAML 파일을 작성할 때는 VS Code에서 열어 작업합니다.

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

VS Code 탐색기에서 파일을 생성하거나, 내장 터미널에서 명령어를 실행하며 실습합니다.

---

## 2) kubectl run으로 Pod 빠르게 생성

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

## 3) YAML로 Pod 생성

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

## 4) 멀티 컨테이너 Pod

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

## 5) Label과 --show-labels

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

## 정리 (리소스 삭제)

```bash
kubectl delete -f pod-nginx.yaml
kubectl delete -f pod-multi.yaml
```
