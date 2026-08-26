# 🔧 도전 과제 — Deployment + emptyDir 로그 공유

## 과제 설명

실습에서 배운 `--dry-run` 템플릿 추출과 `emptyDir` 볼륨을 결합하는 과제입니다.

**직접 Deployment YAML을 처음부터 작성하지 않고**, `kubectl`이 뽑아준 템플릿을 수정해서 완성하세요.

---

## 요구사항

아래 조건을 만족하는 `log-deploy.yaml`을 완성하고 배포하세요.

| 항목 | 값 |
|---|---|
| 리소스 종류 | `Deployment` |
| 이름 | `log-deploy` |
| 컨테이너 1 이름 | `writer` |
| 컨테이너 1 이미지 | `busybox:1.36-musl` |
| 컨테이너 2 이름 | `reader` |
| 컨테이너 2 이미지 | `busybox:1.36-musl` |
| 볼륨 이름 | `shared-log` |
| 볼륨 타입 | `emptyDir` |
| 마운트 경로 (두 컨테이너 모두) | `/shared` |
| writer 동작 | 매 2초마다 현재 시각을 `/shared/log.txt`에 추가 기록 |
| reader 동작 | `/shared/log.txt`를 실시간으로 출력 |

### 완성 후 확인 방법

```bash
# writer가 파일을 쓰고 있는지 확인
kubectl logs deploy/log-deploy -c writer

# reader가 파일을 읽고 있는지 확인
kubectl logs deploy/log-deploy -c reader
```

`reader` 로그에 타임스탬프가 계속 출력되면 성공입니다.

---

## 힌트

막혔을 때 하나씩 펼쳐보세요.

??? tip "힌트 1 — 템플릿 어떻게 뽑지?"
    `kubectl create deployment`에 `--dry-run=client -o yaml`을 붙이면 실제 배포 없이 YAML 템플릿만 출력됩니다.

    === "Windows PowerShell"
        ```powershell
        kubectl create deployment log-deploy `
          --image=busybox:1.36-musl `
          --dry-run=client -o yaml | Out-File log-deploy.yaml -Encoding utf8
        ```
    === "macOS/Linux"
        ```bash
        kubectl create deployment log-deploy \
          --image=busybox:1.36-musl \
          --dry-run=client -o yaml > log-deploy.yaml
        ```

    뽑힌 파일을 VS Code로 열어서 수정하세요.

    ```bash
    code log-deploy.yaml
    ```

??? tip "힌트 2 — 컨테이너가 두 개인데 어디에 추가하지?"
    `--dry-run`으로 뽑은 YAML에는 컨테이너가 한 개입니다.
    `spec.template.spec.containers` 아래에 두 번째 컨테이너를 추가하면 됩니다.

    또한 첫 번째 컨테이너의 이름도 `writer`로 바꿔야 합니다.

    ```yaml
    spec:
      template:
        spec:
          containers:
            - name: writer          # 이름 수정
              image: busybox:1.36-musl
              command: [???]        # 힌트 3 참고
            - name: reader          # 두 번째 컨테이너 추가
              image: busybox:1.36-musl
              command: [???]        # 힌트 3 참고
    ```

??? tip "힌트 3 — command를 어떻게 써야 하지?"
    `busybox`의 `sh -c`로 쉘 명령어를 실행합니다.

    **writer** — 2초마다 현재 시각을 파일에 추가:
    ```yaml
    command: ["sh", "-c", "while true; do date >> /shared/log.txt; sleep 2; done"]
    ```

    **reader** — 파일을 실시간으로 읽기 (`tail -f`):
    ```yaml
    command: ["sh", "-c", "sleep 3 && tail -f /shared/log.txt"]
    ```

    !!! note ""
        `sleep 3`은 writer가 파일을 먼저 만들 때까지 기다리는 용도입니다.

??? tip "힌트 4 — emptyDir 볼륨은 어디에 넣지?"
    볼륨 선언은 **컨테이너와 같은 레벨**인 `spec.template.spec.volumes`에 넣습니다.
    볼륨 마운트는 **각 컨테이너 안**의 `volumeMounts`에 넣습니다.

    ```yaml
    spec:
      template:
        spec:
          containers:
            - name: writer
              ...
              volumeMounts:
                - name: shared-log      # volumes에 선언한 이름과 일치
                  mountPath: /shared
            - name: reader
              ...
              volumeMounts:
                - name: shared-log
                  mountPath: /shared
          volumes:
            - name: shared-log
              emptyDir: {}              # 빈 중괄호 = 기본 설정
    ```

---

## 정답

모두 시도해봤다면 아래에서 확인하세요.

??? success "정답 보기"

    **1단계: 템플릿 추출**

    === "Windows PowerShell"
        ```powershell
        kubectl create deployment log-deploy `
          --image=busybox:1.36-musl `
          --dry-run=client -o yaml | Out-File log-deploy.yaml -Encoding utf8
        ```
    === "macOS/Linux"
        ```bash
        kubectl create deployment log-deploy \
          --image=busybox:1.36-musl \
          --dry-run=client -o yaml > log-deploy.yaml
        ```

    **2단계: `log-deploy.yaml` 완성본**

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: log-deploy
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: log-deploy
      template:
        metadata:
          labels:
            app: log-deploy
        spec:
          containers:
            - name: writer
              image: busybox:1.36-musl
              command: ["sh", "-c", "while true; do date >> /shared/log.txt; sleep 2; done"]
              volumeMounts:
                - name: shared-log
                  mountPath: /shared
            - name: reader
              image: busybox:1.36-musl
              command: ["sh", "-c", "sleep 3 && tail -f /shared/log.txt"]
              volumeMounts:
                - name: shared-log
                  mountPath: /shared
          volumes:
            - name: shared-log
              emptyDir: {}
    ```

    **3단계: 배포 및 확인**

    ```bash
    kubectl apply -f log-deploy.yaml
    kubectl get pods

    # reader 로그에 타임스탬프가 출력되면 성공
    kubectl logs deploy/log-deploy -c reader
    ```

    출력 예시:
    ```
    Wed Aug 26 05:12:01 UTC 2026
    Wed Aug 26 05:12:03 UTC 2026
    Wed Aug 26 05:12:05 UTC 2026
    ```

---

## 정리

```bash
kubectl delete deployment log-deploy
```

!!! info "핵심 포인트"
    - `--dry-run=client -o yaml`로 YAML 템플릿을 추출하면 처음부터 작성할 필요가 없습니다.
    - `emptyDir`은 **같은 Pod 안의 컨테이너끼리만** 파일을 공유합니다.
    - Pod가 재시작되어도 emptyDir 데이터는 유지되지만, **Pod 삭제 후 재생성 시 사라집니다.**
