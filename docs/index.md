# K8s for 4 Days

4일 완성 컨테이너 & 쿠버네티스 실습 가이드입니다.

---

## 커리큘럼

| 일차 | 주제 | 핵심 키워드 |
|------|------|------------|
| **1일차** | Docker 기초 · VM vs 컨테이너 · K8s 환경 구축 · Pod 실습 | VM, Docker, 컨테이너 이미지, ACR, kubectl, Pod |
| **2일차** | 워크로드 컨트롤러 · 네트워크 · 설정 관리 | ReplicaSet, Deployment, Service, Namespace, ConfigMap, Secret, Ingress |
| **3일차** | 트래픽 · 스토리지 · 안정성 · 자원 관리 | Gateway API, emptyDir, PV/PVC, Probe, Requests/Limits, QoS |
| **4일차** | 노드 스케줄링 · 보안 (RBAC) | nodeSelector, Taint/Toleration, ServiceAccount, Role, ClusterRole, SA 토큰 |
| **6일차** | 네트워크 정책 · Cilium mTLS | NetworkPolicy, PodSelector, NamespaceSelector, Egress, ipBlock, CiliumNetworkPolicy, WireGuard, mTLS |

---

## 일차별 상세 내용

=== "1일차"
    | # | 실습 | 내용 |
    |---|------|------|
    | 1 | VM 직접 구축하기 | VirtualBox로 Ubuntu VM 설치 및 초기 설정 |
    | 2 | 컨테이너 실습 | Docker 설치 · 이미지 pull/run · 포트 바인딩 |
    | 3 | 스스로 하기 과제 | 컨테이너 실습 자체 도전 |
    | 4 | 레지스트리와 이미지 실습 | Docker Hub / ACR 활용, 이미지 push/pull |
    | 5 | 나만의 이미지 만들고 ACR에 올리기 | Dockerfile 작성 → 빌드 → ACR push |
    | 6 | 도커 네트워크 실습 | bridge · host · none 네트워크 비교 |
    | 7 | 실습 환경 설치 | VS Code / Rancher Desktop / Docker Desktop |
    | 8 | kubectl 기본 명령어 | cluster-info, get, describe, logs, exec |
    | 9 | 파드 실습 | Pod YAML 작성, 멀티 컨테이너, 라벨/셀렉터 |

=== "2일차"
    | # | 실습 | 내용 |
    |---|------|------|
    | 1 | ReplicaSet & Deployment | replicas 조정, rolling update, rollback |
    | 🔧 | 도전 과제 — Deployment | Deployment 직접 만들어보기 |
    | 2 | Service | ClusterIP · NodePort · port-forward |
    | 🔧 | 디버깅 도전 과제 | 의도적으로 망가진 리소스 수정하기 |
    | 3 | Namespace | 네임스페이스 분리 · 격리 실습 |
    | 🔧 | 도전 과제 — Namespace | 네임스페이스에 앱 배포하고 관찰하기 |
    | 4 | ConfigMap & Secret | 환경변수 주입 · 볼륨 마운트 |
    | 5 | Ingress | nginx ingress controller, 경로 기반 라우팅 |
    | 6 | Ingress TLS & Redirect | cert-manager, HTTPS 강제 리다이렉트 |

=== "3일차"
    | # | 실습 | 내용 |
    |---|------|------|
    | 1 | Gateway API | GatewayClass · Gateway · HTTPRoute |
    | 2 | 볼륨 — emptyDir | Pod 내 컨테이너 간 파일 공유 |
    | 3 | 볼륨 — PV/PVC | 정적/동적 프로비저닝, Azure Disk/Files |
    | 4 | Probe 실습 | Liveness · Readiness · Startup |
    | 5 | Resource Requests & Limits | Pending, CPU throttling, OOMKilled |
    | 6 | Resource QoS | BestEffort · Burstable · Guaranteed, Eviction |
    | 🔧 | 도전 과제 — 미니 블로그 | Gateway API + PVC + 멀티 Deployment |

=== "4일차"
    | # | 실습 | 내용 |
    |---|------|------|
    | 1 | 노드 스케줄링 | nodeSelector, Taint/Toleration, cordon/drain |
    | 2 | RBAC 01 — ServiceAccount | SA 개념, 토큰 자동 마운트 확인 |
    | 3 | RBAC 02 — Role & RoleBinding | 네임스페이스 권한 제어 |
    | 4 | RBAC 03 — ClusterRole | 클러스터 전체 범위 권한 |
    | 5 | RBAC 04 — 최소 권한 설계 | 실무 보안 설계 패턴 |
    | 6 | RBAC 05 — SA 토큰 관리 | automount 제어, Projected Volume |

=== "6일차"
    | # | 실습 | 내용 |
    |---|------|------|
    | 1 | NetworkPolicy 01 — 개념 & Deny-All | NetworkPolicy 기초, 기본 통신 허용 확인, Ingress/Egress Deny-All |
    | 2 | NetworkPolicy 02 — PodSelector | 라벨로 특정 Pod 허용, matchExpressions, OR/AND 조건 |
    | 3 | NetworkPolicy 03 — NamespaceSelector | NS 라벨 기반 허용, AND/OR 조합, kubernetes.io/metadata.name |
    | 4 | NetworkPolicy 04 — Egress & ipBlock | 아웃바운드 제어, CIDR 허용/차단, DNS(53) 예외 |
    | 5 | NetworkPolicy 05 — Cilium L7 정책 | HTTP 경로/메서드 필터링, toFQDNs, Hubble 관찰 |
    | 6 | NetworkPolicy 06 — Cilium mTLS | WireGuard 암호화, SPIFFE Identity, authentication.mode: required |
