# 3-3-1. 볼륨 실습 — emptyDir

## 실습 목표

- emptyDir 임시 볼륨을 사용해 컨테이너 간 데이터를 공유한다.
- Pod 삭제 시 emptyDir 데이터가 소멸됨을 직접 확인한다.

---

## emptyDir — 임시 볼륨

가장 단순한 볼륨 타입인 **emptyDir**부터 시작합니다. Pod가 실행되는 동안만 존재하며, Pod 삭제 시 데이터도 함께 삭제됩니다.

| 특징 | 내용 |
|---|---|
| 데이터 생명주기 | Pod와 동일 (Pod 삭제 시 소멸) |
| 주요 용도 | 컨테이너 간 파일 공유, 캐시, 임시 작업 공간 |
| 설정 난이도 | 매우 간단 (StorageClass, PVC 불필요) |
| Rancher Desktop 호환 | 추가 설정 없이 바로 사용 가능 |

### 1) 기본 emptyDir Pod

`pod-emptydir.yaml` 파일을 만듭니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
    - name: writer
      image: busybox:1.36-musl
      command: ["sh", "-c", "echo 'Hello emptyDir!' > /cache/hello.txt && sleep 3600"]
      volumeMounts:
        - name: cache-vol
          mountPath: /cache
  volumes:
    - name: cache-vol
      emptyDir: {}
```

```bash
kubectl apply -f pod-emptydir.yaml
kubectl exec emptydir-pod -- cat /cache/hello.txt
```

출력:

```
Hello emptyDir!
```

### 2) 컨테이너 간 파일 공유

emptyDir의 핵심 용도 중 하나는 **같은 Pod 내 컨테이너 간 데이터 공유**입니다.

`pod-emptydir-shared.yaml` 파일을 만듭니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-shared
spec:
  containers:
    - name: writer
      image: busybox:1.36-musl
      command: ["sh", "-c", "while true; do date >> /shared/log.txt; sleep 2; done"]
      volumeMounts:
        - name: shared-vol
          mountPath: /shared
    - name: reader
      image: busybox:1.36-musl
      command: ["sh", "-c", "sleep 5 && tail -f /shared/log.txt"]
      volumeMounts:
        - name: shared-vol
          mountPath: /shared
  volumes:
    - name: shared-vol
      emptyDir: {}
```

```bash
kubectl apply -f pod-emptydir-shared.yaml
kubectl logs emptydir-shared -c reader
```

`writer` 컨테이너가 기록한 타임스탬프를 `reader` 컨테이너가 읽어옵니다.

### 3) emptyDir 한계 확인 — Pod 삭제 시 데이터 소실

```bash
# 현재 데이터 확인
kubectl exec emptydir-pod -- cat /cache/hello.txt

# Pod 삭제 후 재생성
kubectl delete pod emptydir-pod
kubectl apply -f pod-emptydir.yaml

# 컨테이너 기동 대기 후 확인
kubectl wait --for=condition=Ready pod/emptydir-pod --timeout=30s
kubectl exec emptydir-pod -- ls /cache/
```

> **포인트**: Pod를 삭제하고 재생성하면 `/cache` 디렉토리는 **빈 상태**입니다. 컨테이너 재시작(크래시 복구)과 달리 Pod 삭제·재생성은 emptyDir 데이터를 소멸시킵니다.

### 정리

```bash
kubectl delete pod emptydir-pod emptydir-shared
```

> **결론**: emptyDir은 임시 데이터나 컨테이너 간 파일 공유에 적합하지만, Pod 삭제 후에도 데이터를 유지하려면 **PersistentVolume(PV) / PVC**가 필요합니다. → 다음 실습(3-3-2)에서 계속합니다.
