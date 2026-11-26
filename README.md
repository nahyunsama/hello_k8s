# Hello k8s

나현의 k8s 공부와 테스트 REPO

> **환경**: minikube 3 nodes (`minikube`, `minikube-m02`, `minikube-m03`, Docker 드라이버, Windows) / Helm 설치 완료
> **목표**: 단일 노드에서는 못 해보는 스케줄링, 장애 대응, 노드 분산까지 단계적으로 실습

---

## 📍 진행 현황

| 단계 | 주제 | 폴더 | 상태 |
|---|---|---|---|
| 0 | 사전 준비 | - | 🟡 진행 중 |
| 1 | 상태 없는 앱 (podinfo) | [00_podinfo](00_podinfo/README.md) 01~05, 07, 09 | 🟡 진행 중 |
| 2 | 멀티 노드 실습 ⭐ | [00_podinfo/06-scheduling](00_podinfo/README.md) | ⬜ 시작 전 |
| 3 | 여러 계층 앱 + Ingress | `01_multi-tier` (예정) | ⬜ 시작 전 |
| 4 | 상태 있는 앱 (PostgreSQL) | `02_postgres` (예정) | ⬜ 시작 전 |
| 5 | 운영 도구 (모니터링 + GitOps) | `03_observability`, `04_gitops` (예정) | ⬜ 시작 전 |

상태 표시: ⬜ 시작 전 · 🟡 진행 중 · ✅ 완료

> 💡 아래 체크리스트의 `- [ ]`를 `- [x]`로 바꾸면 완료 표시가 된다. GitHub나 VS Code 미리보기(`Ctrl+Shift+V`)에서 체크박스로 보인다.

---

## ⚙️ 0단계: 사전 준비

- [x] 노드 상태 확인: `kubectl get nodes -o wide`
- [x] 실습용 네임스페이스 생성: `kubectl apply -f 00_podinfo/namespace.yaml`
- [x] `kubectl diff`용 diff 설정 (Windows): `$env:PATH += ";C:\Program Files\Git\usr\bin"`
- [ ] metrics-server 활성화 (HPA, `kubectl top`에 필요): `minikube addons enable metrics-server`
- [ ] ingress 활성화: `minikube addons enable ingress`
- [ ] 멀티 노드용 스토리지 활성화
  - `minikube addons enable volumesnapshots`
  - `minikube addons enable csi-hostpath-driver`

> 💡 기본 storage-provisioner는 멀티 노드를 제대로 지원하지 않으므로 PVC를 쓸 때는 `csi-hostpath-sc` StorageClass를 사용한다.

---

## 1️⃣ 1단계: 상태 없는(Stateless) 앱 — podinfo

**앱 소개**: `stefanprodan/podinfo`는 K8s 실습용으로 만든 작은 Go 웹앱. 헬스체크, 메트릭, 장애 주입 엔드포인트를 제공한다.

**학습 키워드**: Deployment · ReplicaSet · Service · Probe · Rolling Update · HPA

**실습 폴더**: [00_podinfo](00_podinfo/README.md)

### [01-basic-deploy](00_podinfo/01-basic-deploy/guideline.md): 기본 배포
- [x] Helm 없이 YAML로 직접 Deployment와 Service 작성해서 배포
- [x] Service 로드밸런싱 확인 (curl 파드), 브라우저 접속 (port-forward)
- [x] 자가 치유: 파드를 삭제하면 새 파드가 생기는 것 확인
- [x] `replicas` 조정: `kubectl scale` → `apply`하면 파일 값으로 돌아가는 것 확인
- [x] 이미지 버전 변경으로 롤링 업데이트: `kubectl set image ...`
- [x] 롤백: `kubectl rollout undo deploy/podinfo -n lab`
- [x] 일부러 망가뜨리기: `ImagePullBackOff` 원인을 찾고 복구

### [02-probes](00_podinfo/02-probes/guideline.md): 헬스체크
- [ ] readiness/liveness probe 설정 후 `/readyz` 실패시켜 보기
- [ ] 고장 난 버전(`PODINFO_UNREADY`)에서 롤링 업데이트가 멈추는 것 확인
- [ ] liveness 실패로 재시작이 반복되는 것 확인
- [ ] startupProbe로 느리게 시작하는 앱 처리

### 03-resources-hpa: 리소스 & 오토스케일링
- [ ] requests/limits 설정, 메모리 초과(OOMKilled) 확인
- [ ] HPA 설정 후 부하를 줘서 오토스케일링 확인

### 04-config-secret: 설정 주입
- [ ] ConfigMap, Secret으로 설정 주입
- [ ] 설정 변경 후 `kubectl rollout restart`

### 05-service-types: 외부 노출
- [ ] Service 타입 비교: ClusterIP, NodePort (`minikube service <name> -n lab`)
- [ ] LoadBalancer + `minikube tunnel`

### 07-ingress / 08-kustomize / 09-helm
- [ ] Ingress로 podinfo 외부 노출
- [ ] Kustomize로 폴더 간 중복 YAML 정리
- [ ] 같은 앱을 Helm 차트로 배포하고 YAML 방식과 비교

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm install podinfo podinfo/podinfo -n lab
```

---

## 2️⃣ 2단계: 멀티 노드 전용 실습 ⭐

**학습 키워드**: Scheduling · Affinity · Taint/Toleration · Drain · PDB

**실습 폴더**: `00_podinfo/06-scheduling` (예정)

- [x] 파드가 어느 노드에 떴는지 확인: `kubectl get pods -o wide -n lab`
- [ ] `podAntiAffinity`로 같은 앱 파드를 서로 다른 노드에 배치
- [ ] `topologySpreadConstraints`로 노드마다 골고루 분산
- [ ] `nodeSelector`, 노드 라벨로 특정 노드에만 배포
  - `kubectl label node minikube-m02 tier=backend`
- [ ] Taint/Toleration 실습
  - `kubectl taint node minikube-m03 dedicated=db:NoSchedule`
- [ ] 노드 장애 흉내 내기: `kubectl drain minikube-m02 --ignore-daemonsets --delete-emptydir-data`
- [ ] 다른 노드로 파드가 재스케줄되는 과정 관찰
- [ ] 원복: `kubectl uncordon minikube-m02`
- [ ] PodDisruptionBudget(PDB) 설정 후 drain 동작 차이 확인

---

## 3️⃣ 3단계: 여러 계층(Multi-tier) 앱 + Ingress

**구성 예시**: Frontend → Backend API → Redis

**학습 키워드**: Ingress · ConfigMap · Secret · 서비스 디스커버리(DNS)

**실습 폴더**: `01_multi-tier` (예정)

- [ ] Frontend, Backend, Redis를 각각 Deployment와 Service로 배포
- [ ] Service DNS로 통신 확인 (`redis.lab.svc.cluster.local`)
- [ ] ConfigMap으로 환경 설정 주입
- [ ] Secret으로 비밀번호 주입 (env와 volume mount 두 방식 모두)
- [ ] Ingress로 경로별 라우팅 (`/` → frontend, `/api` → backend)
- [ ] `minikube tunnel` 또는 `minikube ip`로 외부 접속 테스트

---

## 4️⃣ 4단계: 상태 있는(Stateful) 앱 — PostgreSQL

**학습 키워드**: StatefulSet · PVC · StorageClass · Headless Service · Operator/CRD

**실습 폴더**: `02_postgres` (예정)

- [ ] StorageClass 확인: `kubectl get sc` (`csi-hostpath-sc` 사용)
- [ ] StatefulSet으로 PostgreSQL 직접 배포 (volumeClaimTemplates)
- [ ] 파드를 삭제해도 데이터가 남는지 확인
- [ ] Headless Service로 파드별 고정 DNS 확인 (`postgres-0.postgres`)
- [ ] **CloudNativePG** 오퍼레이터로 PostgreSQL 클러스터(Primary 1, Replica 2) 구성
- [ ] Primary 파드 삭제 후 자동 failover 관찰

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm install cnpg cnpg/cloudnative-pg -n cnpg-system --create-namespace
```

> ⚠️ Bitnami 차트는 2025년부터 무료 이미지 정책이 바뀌었다. 학습용으로는 CloudNativePG를 추천.

---

## 5️⃣ 5단계: 운영 도구 (Observability + GitOps)

### 📊 모니터링: kube-prometheus-stack
**실습 폴더**: `03_observability` (예정)

- [ ] Helm으로 Prometheus, Grafana, Alertmanager 설치
- [ ] Grafana 대시보드에서 노드와 파드 리소스 확인
- [ ] podinfo 메트릭을 ServiceMonitor로 수집
- [ ] 간단한 알림 규칙(PrometheusRule) 작성

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

> ⚠️ 리소스를 많이 쓰므로 다른 실습 앱을 정리한 뒤에 설치하는 것을 권장.

### 🔄 GitOps: Argo CD
**실습 폴더**: `04_gitops` (예정)

- [ ] Argo CD 설치 및 UI 접속
- [ ] 이 실습 Git 저장소(`hello_k8s`)를 Application으로 등록
- [ ] Git에 push하면 클러스터에 자동 반영되는지 확인
- [ ] 수동 변경(drift) 감지와 self-heal 확인

---

## 💡 배운 개념 메모

실습하면서 따로 알게 된 것들.

- **ReplicaSet 해시**: 파드 템플릿으로 계산한다. 템플릿이 같으면 같은 ReplicaSet을 재사용한다.
- **`rollout status`**: `apply`는 요청을 접수만 하고 바로 끝난다. 실제로 배포가 끝났는지는 `rollout status`로 기다려서 확인한다.
- **`port-forward`**: 파드 하나에만 붙는 임시 터널이다. Service 로드밸런싱을 거치지 않는다.
- **Service 로드밸런싱**: 별도 프로세스가 아니라 kube-proxy가 노드마다 써 둔 iptables 규칙이다. ClusterIP는 가상 IP다.
- **kindnet (CNI)**: 파드에 IP를 주고 노드 간 라우팅 경로를 만든다. kube-proxy는 그 위에서 Service IP를 바꿔치기한다.
- **`last-applied-configuration`**: `set image`나 `rollout undo`로 바꾼 내용은 이 메모에 반영되지 않는다. 파일을 고쳐서 `apply`하는 게 정석이다.
- **롤링 업데이트**: replicas=3일 때 `maxSurge=1`, `maxUnavailable=0`이다. 새 파드가 Ready가 되기 전에는 옛 파드를 지우지 않는다. 자동 롤백은 없다.

---

## 📌 참고 사항

| 항목 | 권장 |
|---|---|
| 노드당 리소스 (1~4단계) | 2 CPU / 2~4GB RAM |
| 노드당 리소스 (5단계) | 4GB 이상 권장 |
| 나중에 도전해볼 데모 | Google Online Boutique (마이크로서비스 약 10개, 무거움) |

### 자주 쓰는 명령어
- `kubectl get all -n lab`: 리소스 전체 보기
- `kubectl describe pod <name> -n lab`: 이벤트, 에러 확인
- `kubectl logs -f <pod> -n lab`: 로그 보기 (`--previous`는 재시작 전 로그)
- `kubectl diff -f <파일>`: apply 전에 바뀔 내용 미리 보기
- `kubectl top nodes` / `kubectl top pods -n lab`: 리소스 사용량 보기
- `helm list -A`: Helm 릴리스 목록 보기
- `minikube dashboard`: 웹 대시보드 열기
