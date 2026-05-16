# Kubernetes Rolling Update 및 장애 복구 실습

> 분산 시스템을 위한 컨테이너 아키텍처 과목 / Rollout & Rollback

`Deployment`, `ReplicaSet`, `Service`의 상호작용을 직접 관찰하고, 의도적인 장애를 발생시킨 후 `kubectl rollout undo`로 복구하는 실습.

---

## 1. 실습 환경

| 항목 | 값 |
|---|---|
| OS | macOS 26.4 (arm64) |
| 컨테이너 런타임 | colima (lima + docker) |
| Kubernetes | minikube v1.38.1 (driver: docker) |
| Cluster version | v1.35.1 |
| kubectl | v1.36.1 |
| 사용 이미지 | `gcr.io/google-samples/hello-app:1.0` / `:2.0` (장애용 `:3.0` — 존재하지 않음) |

상세 환경 정보: [`results/00-environment.txt`](results/00-environment.txt)

## 2. 리소스 구성

### `deployment.yaml`
- `replicas: 3` — **항상 3개 서버 유지**
- `strategy.type: RollingUpdate` (`maxSurge: 1`, `maxUnavailable: 0`)
  - 새 Pod이 ready된 뒤에야 기존 Pod 제거 → **다운타임 0**
- `revisionHistoryLimit: 10` — 배포 이력 보관
- `readinessProbe` — 컨테이너가 8080에 응답해야 Service Endpoints에 등록
- `livenessProbe` — 죽은 컨테이너 자동 재시작

### `service.yaml`
- `type: NodePort`, `nodePort: 30080`
- `selector: {app: hello-app, tier: frontend}` — Deployment의 Pod 라벨과 매칭

## 3. Phase별 진행 결과

각 Phase의 raw 결과는 `results/`에, 캡처는 `screenshots/`에 있습니다.

| Phase | 내용 | 결과 파일 | 스크린샷 |
|---|---|---|---|
| 0 | 환경 준비 | `00-environment.txt` | — |
| 1 | 최초 배포 (v1.0) | `01-initial-deployment.txt` | `01-initial-state.png` |
| 2 | 관계/라벨/셀렉터/어노테이션 증명 | `02a..02c-*.txt` | `02-relationships-and-access.png` |
| 3 | 정상 업데이트 (v1.0→v2.0) | `03a..03d-*.txt` | `03-rolling-update.png` |
| 4 | 의도적 장애 (v3.0, 존재하지 않는 태그) | `04a..04c-*.txt` | `04-failure.png` |
| 5 | 롤백 (→ v2.0) | `05a..05b-*.txt` | `05-rollback.png` |

## 4. Pod ↔ ReplicaSet ↔ Deployment ↔ Service 관계

```
[Deployment hello-app]
       │  생성/소유
       ▼
[ReplicaSet hello-app-<pod-template-hash>]
       │  생성/소유
       ▼
[Pod ×3]   labels: app=hello-app, tier=frontend, pod-template-hash=...
       ▲  selector로 발견
       │
[Service hello-app-svc]   selector: app=hello-app, tier=frontend
       │  → resolves to
       ▼
[Endpoints  <Pod IP>:8080 × N]
```

증명한 방법(Phase 2):
- `kubectl get pods --show-labels` → Pod에 `app/tier/pod-template-hash` 라벨 존재
- `kubectl describe rs ...` → `Selector`가 Pod 라벨과 일치, `Controlled By: Deployment/hello-app`
- `kubectl describe svc ...` → `Endpoints`에 정확히 3개 Pod IP가 자동 등록 (`10.244.0.3:8080,10.244.0.4:8080,10.244.0.5:8080`)
- `Events`에 `replicaset-controller Created pod: ...` ×3 → RS가 Pod를 만든다는 것을 컨트롤러 로그로 확인

## 5. Label / Selector / Annotation 사용 목적

| 메커니즘 | 위치 | 목적 | 본 실습에서의 역할 |
|---|---|---|---|
| **Label** | `metadata.labels`, `spec.template.metadata.labels` | 리소스에 식별/분류 정보를 붙임 | `app=hello-app`, `tier=frontend`으로 Pod들을 한 묶음으로 식별 |
| **Selector** | `spec.selector` | Label을 기준으로 다른 리소스를 동적으로 발견 | Service가 selector로 Pod들을 찾아 Endpoints 자동 구성. RS는 selector로 자기 Pod만 관리 (`pod-template-hash` 포함) |
| **Annotation** | `metadata.annotations` | 식별이 아닌 부가 메타데이터 보관 | `kubernetes.io/change-cause`로 배포 사유 기록 → `kubectl rollout history`에 표시. 한글 description으로 운영자 메모 |

**중요한 차이**:
- Service의 selector는 `pod-template-hash` 없이 `app`, `tier`만 사용
  → **새 버전 RS의 Pod도 자동으로 같은 Service에 잡힘**. Rolling update 중 다운타임 없이 트래픽이 새 Pod으로 자연 이동.
- RS의 selector는 `pod-template-hash`를 포함
  → 자기 RS가 만든 Pod만 관리, 다른 버전 Pod은 건드리지 않음.

## 6. 장애 원인 및 복구 과정

### 시나리오
1. `kubectl set image deploy/hello-app hello-app=gcr.io/google-samples/hello-app:3.0`
2. `:3.0` 태그가 레지스트리에 존재하지 않음

### 발생한 오류
```
Failed to pull image "gcr.io/google-samples/hello-app:3.0":
  Error response from daemon: manifest for gcr.io/google-samples/hello-app:3.0 not found:
  manifest unknown: Failed to fetch "3.0"
Error: ErrImagePull → ImagePullBackOff
```

### 상태 변화
| 리소스 | 변화 |
|---|---|
| 새 RS `5959b68ffc` (v3.0) | desired=1, current=1, **ready=0** (Pod이 `ImagePullBackOff`) |
| 기존 RS `6fccdb7684` (v2.0) | **3/3/3 그대로 유지** ← `maxUnavailable: 0` 효과 |
| Deployment | `UP-TO-DATE: 1`, `AVAILABLE: 3` → 진행은 시작했지만 완료 못함 |
| Service Endpoints | 장애 Pod IP(`10.244.0.9`) **자동 제외**, 정상 Pod 3개만 등록 |
| 사용자 응답 | curl 5/5 모두 `Version: 2.0.0` ← **무중단** |

### 복구 과정
```bash
kubectl rollout undo deploy/hello-app
kubectl rollout status deploy/hello-app   # "successfully rolled out"
```

### 복구 후 상태
- Pod 3개 이름이 장애 전과 **완전 동일** (`dv5t2`, `h8d9l`, `x85hc`)
  → 장애 동안 죽은 적이 없으므로 진짜로는 "복구"가 아니라 "다시는 안 바뀌도록 확정"이 정확한 표현
- 실패한 RS `5959b68ffc`는 **삭제되지 않고 `0/0/0`으로 보관** (필요시 다시 시도 가능)
- 새 revision 4가 추가되며, 기존 v2.0 템플릿(revision 2)이 그대로 재사용됨

## 7. 발견한 K8s 동작 메커니즘

이번 실습에서 직접 관찰한 비교적 미세한 K8s 동작들.

### 7-1. `change-cause` 어노테이션의 전파
`kubectl annotate deploy/... kubernetes.io/change-cause=...`로 어노테이션을 갱신하면, Deployment 컨트롤러가 **모든 (현재 보관 중인) RS에 이 값을 동기화**한다. 따라서 `kubectl rollout history`의 `CHANGE-CAUSE` 열은 **revision별로 보존되는 게 아니라 현재의 Deployment annotation 값으로 일괄 표시**된다.

→ revision별로 다른 change-cause를 남기고 싶다면, 각 rollout 시점에 적절한 annotation을 설정하고 그 뒤로는 수정하지 않아야 한다. 또는 `kubectl set image --record`(deprecated)를 쓰는 방법도 있었음.

### 7-2. Rolling Update의 무중단성
`maxSurge: 1, maxUnavailable: 0` 설정에서:
- 새 Pod이 ready된 뒤에야 기존 Pod 1개를 종료
- 따라서 항상 **최소 3개 Pod이 ready** 상태로 유지됨
- 장애 상황에서도 새 Pod이 ready 못 되므로 기존 Pod이 그대로 살아있음 → 무중단

### 7-3. ReplicaSet의 사후 보관
업데이트 후에도 옛 RS는 **`replicas=0`으로 남아있음**.
- `revisionHistoryLimit: 10`이라 최대 10개까지 보관
- 이 RS들이 곧 "롤백 가능한 버전들"의 실체
- `kubectl rollout undo`는 새 RS를 만들지 않고 옛 RS를 다시 scale up

### 7-4. Service의 readiness 필터
`readinessProbe` 미통과 Pod은 Service Endpoints에서 자동 제외된다. 장애 시 사용자 트래픽이 끊기지 않은 핵심 이유.

## 8. 재현 방법

```bash
# 1. 환경 시작
colima start --cpu 4 --memory 4
minikube start --driver=docker

# 2. 배포
kubectl apply -f deployment.yaml -f service.yaml
kubectl rollout status deploy/hello-app

# 3. 접속 확인 (minikube docker 드라이버)
minikube ssh -- "curl -s http://localhost:30080"

# 4. 업데이트
kubectl annotate deploy/hello-app kubernetes.io/change-cause="Update image to hello-app v2.0" --overwrite
kubectl set image deploy/hello-app hello-app=gcr.io/google-samples/hello-app:2.0
kubectl rollout status deploy/hello-app
kubectl rollout history deploy/hello-app

# 5. 장애 유발
kubectl annotate deploy/hello-app kubernetes.io/change-cause="Bad image hello-app:3.0 (intentional failure test)" --overwrite
kubectl set image deploy/hello-app hello-app=gcr.io/google-samples/hello-app:3.0
kubectl get pods   # ImagePullBackOff 확인

# 6. 복구
kubectl rollout undo deploy/hello-app
kubectl rollout status deploy/hello-app

# 7. 정리
minikube delete
colima stop
```

## 9. 디렉토리 구조

```
rollout-and-rollback/
├── README.md                 # 본 파일
├── deployment.yaml           # 3-replica Deployment + RollingUpdate
├── service.yaml              # NodePort 30080
├── results/                  # 각 Phase의 명령어 출력 로그
│   ├── 00-environment.txt
│   ├── 01-initial-deployment.txt
│   ├── 02a-labels-and-rs.txt
│   ├── 02b-service-and-deployment.txt
│   ├── 02c-curl-access.txt
│   ├── 03a-update-trigger.txt
│   ├── 03b-update-progress.txt
│   ├── 03c-update-complete.txt
│   ├── 03d-curl-v2.txt
│   ├── 04a-bad-image-trigger.txt
│   ├── 04b-failure-state.txt
│   ├── 04c-failure-events-and-survival.txt
│   ├── 05a-rollback.txt
│   └── 05b-recovered.txt
└── screenshots/              # 주요 단계의 터미널 캡처
    ├── 01-initial-state.png
    ├── 02-relationships-and-access.png
    ├── 03-rolling-update.png
    ├── 04-failure.png
    └── 05-rollback.png
```
