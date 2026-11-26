# 01. podinfo를 YAML로 직접 배포하기

> 이 문서를 보면서 YAML 파일 3개를 **직접 타이핑**하고, `kubectl apply`로 배포해 본다.
> 복사해서 붙여넣지 말고 손으로 쳐보는 걸 추천. 오타를 고치는 과정도 공부가 된다.

---

## 0. 시작 전에

### 최종 목표 구조
```
00_podinfo/
├── namespace.yaml        ← Step 1에서 작성 (모든 실습 공용)
└── 01-basic-deploy/
    ├── guideline.md      ← 지금 보고 있는 문서
    ├── deployment.yaml   ← Step 2에서 작성
    └── service.yaml      ← Step 3에서 작성
```

### 그림으로 보면
```
요청 → Service(podinfo:80) → 라벨 app=podinfo 인 파드들(:9898)로 분산
         ↑
Deployment → ReplicaSet → Pod x3 (노드 3개에 분산)
```

### YAML 작성 규칙 (중요!)
- 들여쓰기는 **스페이스 2칸**. **탭(Tab)은 쓰면 안 된다.** 에디터 설정을 꼭 확인하자.
- `key: value`에서 콜론 뒤에 **공백 한 칸**을 넣는다.
- 목록(배열)은 `- `로 시작한다.
- `#` 뒤는 주석이다. 주석은 안 쳐도 된다.

### 모든 K8s 리소스의 공통 뼈대
```yaml
apiVersion:   # 이 리소스를 정의한 API 그룹/버전
kind:         # 리소스 종류 (Namespace, Deployment, Service ...)
metadata:     # 이름, 네임스페이스, 라벨
spec:         # "이렇게 되길 원한다" (원하는 상태)
```

> 💡 필드가 뭔지 궁금하면 `kubectl explain deployment.spec`, `kubectl explain deployment.spec.template.spec.containers`처럼 확인할 수 있다.

### 현재 상태 확인
```powershell
kubectl get nodes
kubectl get ns
```
노드 3개가 `Ready`이면 준비 완료.

---

## Step 1. Namespace 만들기

📄 `00_podinfo/namespace.yaml` (이 파일만 상위 폴더에 둔다)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab
```

**왜 나누나?** 실습 리소스를 한 곳에 모아두면 `kubectl delete ns lab` 한 번으로 전부 정리할 수 있다. 시스템 리소스(`kube-system`)와 섞이지도 않는다.

**왜 상위 폴더에?** 모든 podinfo 실습(01, 02, ...)이 같은 `lab` 네임스페이스를 쓴다. 실습 폴더에서 `kubectl delete -f .`을 해도 네임스페이스는 남아서 다음 실습으로 바로 넘어갈 수 있다.

### 적용 & 확인
```powershell
cd C:\Users\hamna\Documents\hello_k8s\00_podinfo
kubectl apply -f namespace.yaml
kubectl get ns
cd 01-basic-deploy     # 이후 Step은 이 폴더에서 진행
```
✅ 예상 결과
```
namespace/lab created
...
lab               Active   3s
```

---

## Step 2. Deployment 만들기

📄 `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  namespace: lab
  labels:
    app: podinfo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      labels:
        app: podinfo
    spec:
      containers:
        - name: podinfo
          image: ghcr.io/stefanprodan/podinfo:6.7.1
          ports:
            - name: http
              containerPort: 9898
          env:
            - name: PODINFO_UI_MESSAGE
              value: "hello k8s!"
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
```

### 한 줄씩 이해하기

| 필드 | 의미 |
|---|---|
| `apiVersion: apps/v1` | Deployment는 `apps` 그룹에 속한다 (Namespace, Service는 그냥 `v1`) |
| `replicas: 3` | 파드를 **항상 3개** 유지해 달라는 선언 |
| `selector.matchLabels` | 이 라벨이 붙은 파드를 "내 파드"로 관리한다 |
| `template` | 여기부터 **파드 설계도**. 파드를 만들 때 이 내용을 찍어낸다 |
| `template.metadata.labels` | ⚠️ 위의 `selector`와 **반드시 일치**해야 한다. 다르면 apply 단계에서 에러가 난다 |
| `image` | 실행할 컨테이너 이미지와 태그 |
| `containerPort: 9898` | podinfo가 리슨하는 포트. `name: http`로 이름을 붙여 둔다 |
| `env` | 환경 변수. podinfo는 이 값을 웹 화면에 띄운다 |
| `requests` | **스케줄링 기준**. 이만큼 여유 있는 노드에 배치한다 |
| `limits` | **최대 사용량**. 메모리를 초과하면 컨테이너가 강제 종료된다(OOMKilled) |
| `50m` | 0.05 CPU 코어 (m = 1/1000 코어) |
| `64Mi` | 64 메비바이트 |

> 💡 라벨(`app: podinfo`)은 K8s의 핵심 연결고리다. Deployment는 라벨로 파드를 찾고, Service도 라벨로 파드를 찾는다.

### 적용 전 문법 검사 (선택)
```powershell
kubectl apply -f deployment.yaml --dry-run=server
```
에러가 없으면 `(server dry run)`이라고 나온다. 오타가 있으면 여기서 몇 번째 줄인지 알려준다.

### 적용 & 확인
```powershell
kubectl apply -f deployment.yaml
kubectl rollout status deploy/podinfo -n lab
kubectl get deploy,rs,pods -n lab -o wide
```
✅ 예상 결과
```
deployment.apps/podinfo   3/3     3            3
replicaset.apps/podinfo-xxxxxxxxxx   3   3   3
pod/podinfo-xxxxxxxxxx-aaaaa   1/1   Running   ...   minikube
pod/podinfo-xxxxxxxxxx-bbbbb   1/1   Running   ...   minikube-m02
pod/podinfo-xxxxxxxxxx-ccccc   1/1   Running   ...   minikube-m03
```

🔍 관찰 포인트
- Deployment 하나를 만들었는데 **ReplicaSet**이 자동으로 생겼다.
- 파드 이름 = `Deployment이름-ReplicaSet해시-랜덤`
- `NODE` 열을 보면 파드들이 노드에 흩어져 있다.

---

## Step 3. Service 만들기

파드는 죽고 새로 뜰 때마다 **IP가 바뀐다**. 그래서 라벨로 파드들을 묶어 **고정 주소**를 주는 게 Service다.

📄 `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: podinfo
  namespace: lab
spec:
  type: ClusterIP
  selector:
    app: podinfo
  ports:
    - name: http
      port: 80
      targetPort: http
```

| 필드 | 의미 |
|---|---|
| `type: ClusterIP` | 클러스터 **내부에서만** 접근 가능한 고정 IP (기본값) |
| `selector` | `app: podinfo` 라벨이 붙은 파드들로 트래픽을 보낸다 |
| `port: 80` | Service가 받는 포트 |
| `targetPort: http` | 파드의 포트. Deployment에서 이름 붙인 `http`(9898)를 가리킨다 |

### 적용 & 확인
```powershell
kubectl apply -f service.yaml
kubectl get svc -n lab
kubectl get endpointslices -n lab
```
✅ 엔드포인트에 **파드 IP 3개**가 보이면 Service와 파드가 제대로 연결된 것이다.
❌ 엔드포인트가 비어 있다면 → Service의 `selector`와 파드 라벨이 일치하는지 확인.

---

## Step 4. 동작 확인

### 4-1. 클러스터 내부에서 로드밸런싱 확인
```powershell
kubectl run curl -n lab --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- sh
```
curl 파드 안의 셸에 들어가면:
```sh
curl podinfo
curl podinfo.lab.svc.cluster.local
for i in 1 2 3 4 5 6; do curl -s podinfo | grep hostname; done
exit
```
✅ `hostname`이 여러 파드 이름으로 바뀌면서 나오면 로드밸런싱이 되는 것이다.
💡 같은 네임스페이스에서는 `podinfo`만 써도 되고, 정식 이름은 `<서비스>.<네임스페이스>.svc.cluster.local`이다.

### 4-2. 브라우저로 접속
**새 터미널**을 열고:
```powershell
kubectl port-forward svc/podinfo 8080:80 -n lab
```
→ http://localhost:8080 접속하면 "hello k8s!" 메시지가 보인다. 종료는 `Ctrl+C`.

---

## Step 5. 실습

### 5-1. 자가 치유(Self-healing)
```powershell
# 터미널 A: 실시간 관찰
kubectl get pods -n lab -w

# 터미널 B: 파드 하나 삭제
kubectl delete pod <파드이름> -n lab
```
✅ 삭제하자마자 새 파드가 생긴다. ReplicaSet이 "3개 유지" 약속을 지키는 것이다.

### 5-2. 스케일링
```powershell
kubectl scale deploy podinfo --replicas=6 -n lab
kubectl get pods -n lab -o wide
kubectl scale deploy podinfo --replicas=3 -n lab
```
❓ **생각해보기**: `kubectl scale`로 6개로 바꾼 뒤 `kubectl apply -f deployment.yaml`을 다시 하면 파드는 몇 개가 될까? 직접 해보고 이유를 생각해 보자.

### 5-3. 설정 바꾸기 (선언형 방식)
`deployment.yaml`에서 `PODINFO_UI_MESSAGE` 값을 원하는 문구로 고친 뒤:
```powershell
kubectl apply -f deployment.yaml
kubectl get pods -n lab -w
```
✅ 파드가 하나씩 교체되는 것(롤링 업데이트)을 볼 수 있다. 브라우저를 새로고침해서 바뀐 문구를 확인하자. port-forward는 다시 실행해야 할 수 있다.

### 5-4. 롤링 업데이트 & 롤백
```powershell
kubectl set image deploy/podinfo podinfo=ghcr.io/stefanprodan/podinfo:6.7.0 -n lab
kubectl rollout status deploy/podinfo -n lab
kubectl get rs -n lab
kubectl rollout history deploy/podinfo -n lab

kubectl rollout undo deploy/podinfo -n lab
kubectl get rs -n lab
```
🔍 ReplicaSet이 버전마다 하나씩 남아 있다. 롤백하면 이전 ReplicaSet이 다시 3개로 늘어난다.

### 5-5. 일부러 망가뜨리기 (트러블슈팅)
```powershell
kubectl set image deploy/podinfo podinfo=ghcr.io/stefanprodan/podinfo:nope -n lab
kubectl get pods -n lab
kubectl describe pod <새로생긴파드이름> -n lab
```
✅ `ImagePullBackOff` 상태가 보이고, `describe` 맨 아래 **Events**에서 원인을 찾을 수 있다.
❓ **생각해보기**: 새 파드가 실패하는 동안 기존 파드는 왜 살아 있을까? (힌트: `kubectl explain deployment.spec.strategy.rollingUpdate`)

복구:
```powershell
kubectl rollout undo deploy/podinfo -n lab
```

---

## 🧯 자주 만나는 에러

| 증상 | 원인 / 해결 |
|---|---|
| `error converting YAML to JSON` | 들여쓰기 오류, 탭 문자, 콜론 뒤 공백 누락 |
| `selector does not match template labels` | `selector.matchLabels`와 `template.metadata.labels` 불일치 |
| `namespaces "lab" not found` | Step 1을 먼저 apply 해야 한다 |
| 파드가 `ImagePullBackOff` | 이미지 이름이나 태그 오타 |
| 파드가 `Pending` | `kubectl describe pod`로 확인. 보통 리소스 부족 |
| 엔드포인트가 비어 있음 | Service `selector`와 파드 라벨 불일치 |

### 디버깅 3종 세트
```powershell
kubectl get pods -n lab                 # 상태 보기
kubectl describe pod <이름> -n lab      # Events 보기
kubectl logs <이름> -n lab              # 앱 로그 보기
```

---

## 🧹 정리
```powershell
kubectl delete -f .        # 이 실습의 리소스만 삭제 (lab 네임스페이스는 유지)
# 전부 지우고 싶다면
kubectl delete ns lab
```

---

## ✅ 체크리스트
- [ ] namespace.yaml 작성 및 apply
- [ ] deployment.yaml 작성 및 apply, 파드 3개 Running 확인
- [ ] service.yaml 작성 및 apply, 엔드포인트 3개 확인
- [ ] curl로 로드밸런싱 확인
- [ ] 브라우저 접속 (port-forward)
- [ ] 자가 치유 실습
- [ ] 스케일링 + ❓ 질문 답해보기
- [ ] 설정 변경 (apply)
- [ ] 롤링 업데이트 & 롤백
- [ ] 망가뜨리고 고치기 + ❓ 질문 답해보기

## 다음 단계
➡️ [02-probes](../02-probes/guideline.md): readiness/liveness/startup probe로 "고장 난 버전이 배포되는 것" 막기
