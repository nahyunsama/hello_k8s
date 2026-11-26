# 00_podinfo: podinfo로 배우는 Kubernetes

[podinfo](https://github.com/stefanprodan/podinfo)는 K8s 실습용으로 만든 작은 Go 웹앱이다. 헬스체크, 메트릭, 장애 주입 기능이 들어 있어서 K8s 기능을 하나씩 실험해 보기 좋다.

## 실습 목록

| # | 폴더 | 주제 | 상태 |
|---|---|---|---|
| 01 | [01-basic-deploy](01-basic-deploy/guideline.md) | Deployment, Service, 스케일링, 롤링 업데이트, 롤백 | ✅ |
| 02 | [02-probes](02-probes/guideline.md) | readiness, liveness, startup probe | ⬜ |
| 03 | 03-resources-hpa | requests/limits, OOMKilled, HPA 오토스케일링 | 📝 예정 |
| 04 | 04-config-secret | ConfigMap, Secret, `rollout restart` | 📝 예정 |
| 05 | 05-service-types | ClusterIP, NodePort, LoadBalancer, `minikube tunnel` | 📝 예정 |
| 06 | 06-scheduling | affinity, topology spread, taint, drain, PDB (멀티 노드) ⭐ | 📝 예정 |
| 07 | 07-ingress | Ingress 경로 라우팅 | 📝 예정 |
| 08 | 08-kustomize | 중복된 YAML을 base/overlay로 정리 | 📝 예정 |
| 09 | 09-helm | 같은 앱을 Helm 차트로 배포하고 비교 | 📝 예정 |

## 폴더 규칙

- **폴더마다 독립 실행**: 각 폴더에 그 실습에 필요한 YAML이 전부 들어 있다. 이전 폴더의 YAML을 복사해 와서 새로 배우는 부분만 추가한다.
- **중복은 일부러 허용**: 폴더끼리 비교하면 단계마다 무엇이 추가됐는지 보인다. 이 중복이 불편해질 때쯤 08 Kustomize와 09 Helm이 왜 필요한지 알게 된다.
  ```powershell
  git diff --no-index 01-basic-deploy/deployment.yaml 02-probes/deployment.yaml
  ```
- **네임스페이스는 공용**: `namespace.yaml`(`lab`)은 이 폴더에 하나만 둔다. 실습 폴더를 지워도 네임스페이스는 남는다.

## 사용법

```powershell
cd C:\Users\hamna\Documents\hello_k8s\00_podinfo
kubectl apply -f namespace.yaml          # 처음 한 번만

kubectl apply -f 02-probes\              # 실습 폴더 단위로 배포
kubectl delete -f 02-probes\             # 실습 폴더 단위로 정리

kubectl delete ns lab                    # 전부 정리
```

> ⚠️ 모든 실습이 같은 이름(`podinfo`)의 Deployment를 쓴다. **다음 실습으로 넘어가기 전에 이전 실습을 정리**하거나, 그대로 덮어써서 이어가자.

## 환경

- minikube 3 nodes (Docker 드라이버, Windows)
- kindnet CNI, kube-proxy iptables 모드
- `kubectl diff` 사용 시 Git의 diff 필요: `$env:PATH += ";C:\Program Files\Git\usr\bin"`
