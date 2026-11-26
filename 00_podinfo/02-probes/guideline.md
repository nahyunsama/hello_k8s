# 02. Probe로 "고장 난 파드" 걸러내기

> 01에서 남긴 질문:
> **"이미지는 정상인데 앱이 고장 나서 500 에러만 낸다면?"**
> K8s는 지금 **컨테이너 프로세스가 떠 있으면 정상**이라고 판단한다. 앱이 진짜 일할 준비가 됐는지는 모른다.
> Probe는 K8s가 앱에게 **"너 괜찮아?"라고 주기적으로 물어보는 장치**다.

---

## 0. 개념

### Probe 3종류

| Probe | 질문 | 실패하면 | 비유 |
|---|---|---|---|
| **readinessProbe** | "지금 요청 받을 수 있어?" | Service 엔드포인트에서 **빠진다** (재시작 ❌) | 가게 문 앞에 "준비 중" 팻말 |
| **livenessProbe** | "살아 있어? 멈춘 거 아니지?" | 컨테이너를 **재시작한다** | 쓰러진 직원 교체 |
| **startupProbe** | "시작 다 끝났어?" | 성공할 때까지 **나머지 probe를 보류**한다. 끝까지 실패하면 재시작 | 오픈 준비 시간 |

> 💡 가장 중요한 구분: **readiness는 "트래픽만 끊는다", liveness는 "죽이고 다시 띄운다".**

### 검사 방법
| 방식 | 예시 | 성공 조건 |
|---|---|---|
| `httpGet` | `GET /readyz` | HTTP 200~399 |
| `tcpSocket` | 포트 9898 연결 | 연결 성공 |
| `exec` | `cat /tmp/ready` | 종료 코드 0 |
| `grpc` | gRPC health check | SERVING |

### 타이밍 설정
| 필드 | 기본값 | 의미 |
|---|---|---|
| `initialDelaySeconds` | 0 | 컨테이너 시작 후 첫 검사까지 대기 |
| `periodSeconds` | 10 | 검사 주기 |
| `timeoutSeconds` | 1 | 응답 대기 시간 |
| `failureThreshold` | 3 | **연속** 몇 번 실패하면 실패로 판정할지 |
| `successThreshold` | 1 | 연속 몇 번 성공하면 성공으로 판정할지 |

> 실패 판정까지 걸리는 시간 ≈ `periodSeconds × failureThreshold`

### podinfo가 제공하는 엔드포인트
| 엔드포인트 | 용도 |
|---|---|
| `GET /healthz` | liveness 용 (정상이면 200) |
| `GET /readyz` | readiness 용 (정상이면 200, 준비 안 됐으면 503) |
| `POST /readyz/disable` | readiness를 **일부러 실패**시킴 |
| `POST /readyz/enable` | readiness 복구 |
| 환경 변수 `PODINFO_UNREADY=true` | 처음부터 readiness 실패 상태로 시작 |
| 환경 변수 `PODINFO_UNHEALTHY=true` | 처음부터 liveness 실패 상태로 시작 |

---

## Step 1. 준비

### 폴더 구조
```
00_podinfo/
├── namespace.yaml
├── 01-basic-deploy/
└── 02-probes/
    ├── guideline.md      ← 지금 보고 있는 문서
    ├── deployment.yaml   ← 01을 복사해서 probe 추가
    └── service.yaml      ← 01을 그대로 복사
```

### 01 파일 복사
```powershell
cd C:\Users\hamna\Documents\hello_k8s\00_podinfo
kubectl apply -f namespace.yaml            # 이미 있으면 unchanged
kubectl delete -f 01-basic-deploy\ --ignore-not-found   # 01 실습이 떠 있다면 정리

cd 02-probes
Copy-Item ..\01-basic-deploy\deployment.yaml .
Copy-Item ..\01-basic-deploy\service.yaml .
```

---

## Step 2. deployment.yaml에 probe 추가

`resources:` 블록 **아래에**, 같은 들여쓰기(`resources`와 같은 열)로 추가한다.

📄 `deployment.yaml` (추가할 부분만, `# ↓` 표시)
```yaml
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
          # ↓↓↓ 여기부터 추가 ↓↓↓
          readinessProbe:
            httpGet:
              path: /readyz
              port: http          # 포트 이름으로 참조 (9898)
            periodSeconds: 5      # 5초마다 검사
            failureThreshold: 2   # 2번 연속 실패 → 약 10초 만에 트래픽 차단
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10     # 10초마다 검사
            failureThreshold: 3   # 3번 연속 실패 → 약 30초 만에 재시작
```

> 💡 **왜 liveness를 더 느긋하게?** liveness 실패는 재시작이라는 **강한 조치**다. 잠깐 느려진 걸 "죽었다"고 오판하면 멀쩡한 파드를 계속 죽이게 된다. 그래서 readiness는 민감하게, liveness는 보수적으로 잡는 게 일반적이다.

### 적용 & 확인
```powershell
kubectl diff -f .            # 처음 배포라면 전체가 + 로 보인다
kubectl apply -f .
kubectl rollout status deploy/podinfo -n lab
kubectl get pods -n lab -o wide
kubectl describe pod <파드이름> -n lab
```
✅ `describe` 출력에서 이런 줄을 찾아보자.
```
Liveness:   http-get http://:http/healthz delay=0s timeout=1s period=10s #success=1 #failure=3
Readiness:  http-get http://:http/readyz delay=0s timeout=1s period=5s #success=1 #failure=2
```

---

## Step 3. 실습 ① readiness 실패: 트래픽만 끊긴다

파드 하나의 readiness를 일부러 꺼보자.

### 3-1. 관찰 터미널 열기
**터미널 A**
```powershell
kubectl get pods -n lab -o wide -w
```

### 3-2. 파드 하나를 "준비 안 됨"으로 만들기
**터미널 B**
```powershell
kubectl get pods -n lab -o wide          # 파드 IP 하나 골라두기

kubectl run curl -n lab --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- sh
```
curl 파드 안에서:
```sh
curl -X POST http://<고른파드IP>:9898/readyz/disable
```

### 3-3. 관찰
✅ 약 10초 뒤 터미널 A에서 그 파드가 `0/1`로 바뀐다. **RESTARTS는 0 그대로**다.

curl 파드 안에서 Service로 요청해 보면:
```sh
for i in $(seq 1 10); do curl -s podinfo | grep hostname; done
```
✅ **준비 안 된 파드의 hostname은 나오지 않는다.** 나머지 2개만 응답한다.

**터미널 B를 하나 더 열어서** 엔드포인트도 확인해 보자.
```powershell
kubectl describe endpointslice -n lab -l kubernetes.io/service-name=podinfo
```
✅ 그 파드 IP에 `Ready: false`가 표시된다.

> ⚠️ `kubectl get endpointslices`의 ENDPOINTS 열은 **준비 안 된 IP까지 전부** 보여준다. Ready 여부는 `describe`로 봐야 한다.

### 3-4. 복구
```sh
curl -X POST http://<고른파드IP>:9898/readyz/enable
exit
```
✅ 다시 `1/1`이 되고 트래픽을 받는다.

### 🔍 정리
- readiness 실패 = **"잠깐 손님 받지 마"**. 파드는 살아 있다.
- 실무 예시: 앱 시작 직후 캐시 로딩 중, DB 연결이 잠깐 끊김, 과부하로 잠시 쉬어야 할 때
- 파드가 **종료될 때**도 엔드포인트에서 먼저 빠진다. 그래서 무중단 배포가 가능하다.

---

## Step 4. 실습 ② 고장 난 버전 배포: 롤링 업데이트가 멈춘다 ⭐

01에서 `ImagePullBackOff`로 해봤던 걸, 이번에는 **이미지는 정상인데 앱이 준비가 안 되는** 상황으로 해본다.

```powershell
# 터미널 A는 계속 -w 관찰 중
kubectl set env deploy/podinfo PODINFO_UNREADY=true -n lab
kubectl rollout status deploy/podinfo -n lab --timeout=60s
```

✅ 예상 결과
```
Waiting for deployment "podinfo" rollout to finish: 1 out of 3 new replicas have been updated...
error: timed out waiting for the condition
```
```
NAME                       READY   STATUS    RESTARTS
podinfo-<새해시>-xxxxx      0/1     Running   0     ← 새 버전: 떴지만 Ready 아님
podinfo-<옛해시>-aaaaa      1/1     Running   0     ← 옛 버전 3개는 그대로 서비스 중
podinfo-<옛해시>-bbbbb      1/1     Running   0
podinfo-<옛해시>-ccccc      1/1     Running   0
```

### 🔍 01의 `ImagePullBackOff`와 비교
| | 01 (이미지 태그 오류) | 02 (앱이 준비 안 됨) |
|---|---|---|
| 파드 STATUS | `ImagePullBackOff` | `Running` 😮 |
| READY | `0/1` | `0/1` |
| 막아준 것 | 컨테이너가 아예 못 뜸 | **readinessProbe** |

**probe가 없었다면?** `Running`이니 Ready로 판정했을 것이고, 롤링 업데이트가 끝까지 진행돼서 **고장 난 파드 3개가 전부 배포됐을 것**이다.

### ❓ 생각해보기: 지금 `kubectl apply -f .`을 하면 복구될까?
`deployment.yaml`에는 `PODINFO_UNREADY`가 없으니 apply하면 원래대로 돌아갈 것 같다. **직접 해보자.**
```powershell
kubectl apply -f .
kubectl get deploy podinfo -n lab -o jsonpath='{.spec.template.spec.containers[0].env}'
```
결과를 보고 왜 그런지 생각해 보자. (힌트: 01에서 `rollout undo` 할 때 본 경고 메시지, `last-applied-configuration`)

### 복구
```powershell
kubectl set env deploy/podinfo PODINFO_UNREADY- -n lab    # 이름 뒤에 - 를 붙이면 삭제
kubectl rollout status deploy/podinfo -n lab
kubectl diff -f .                                         # 출력이 없으면 파일과 일치
```

---

## Step 5. 실습 ③ liveness 실패: 재시작된다

```powershell
kubectl set env deploy/podinfo PODINFO_UNHEALTHY=true -n lab
kubectl get pods -n lab -w
```

✅ 관찰 포인트
- readiness는 정상이라서 롤링 업데이트는 **끝까지 진행된다.**
- 그런데 약 30초마다 **RESTARTS가 1씩 늘어난다.**
- 재시작이 반복되면 재시작 간격이 점점 길어지고(back-off), STATUS에 `CrashLoopBackOff`가 보일 수 있다.

```powershell
kubectl describe pod <파드이름> -n lab
```
Events에서 찾아보기:
```
Warning  Unhealthy  ...  Liveness probe failed: HTTP probe failed with statuscode: 503
Normal   Killing    ...  Container podinfo failed liveness probe, will be restarted
```

### ⚠️ liveness의 위험성
이번처럼 **모든 파드의 liveness가 동시에 실패하면** 파드 3개가 같이 재시작을 반복한다. 사실상 전체 장애다. 실무에서 자주 하는 실수들:
- liveness에서 **DB 같은 외부 의존성까지 검사** → DB가 잠깐 끊기면 앱 전체가 재시작 → 상황 악화
- liveness 타이밍을 **너무 빡빡하게** 설정 → 부하가 몰려 느려지면 멀쩡한 파드를 죽임

> 💡 원칙: **liveness는 "프로세스가 멈췄나(데드락 등)"만 가볍게 검사**하고, 외부 의존성 검사는 readiness에 맡긴다.

### 복구
```powershell
kubectl set env deploy/podinfo PODINFO_UNHEALTHY- -n lab
kubectl rollout status deploy/podinfo -n lab
```

---

## Step 6. 실습 ④ 느리게 시작하는 앱: startupProbe

시작하는 데 30초 걸리는 앱을 흉내 내보자. 컨테이너 시작 명령을 **30초 쉬고 podinfo 실행**으로 바꾼다.

### 6-1. startupProbe 없이 해보기
`deployment.yaml`의 `image:` 줄 **바로 아래에** 추가:
```yaml
          image: ghcr.io/stefanprodan/podinfo:6.7.1
          command: ["sh", "-c", "echo 'starting slowly...'; sleep 30; exec ./podinfo"]   # ← 추가
```
```powershell
kubectl apply -f .
kubectl get pods -n lab -w
```

❓ **먼저 예측해 보자.** liveness 설정은 `period 10s × failure 3`이다. 30초 동안 앱이 안 떠 있으면 어떻게 될까?

✅ 결과: 새 파드가 계속 `0/1`이고, liveness 실패로 재시작된다. 재시작하면 다시 30초 대기부터 시작하니 **영원히 못 뜬다.** 롤링 업데이트는 멈추고 옛 파드는 살아 있다. readiness 덕분이다.

> 🔍 **RESTARTS가 늦게 올라가는 이유**: kubelet이 종료 신호(SIGTERM)를 보내는데, 지금 PID 1은 `sh`이고 `sleep` 중이라 신호를 무시한다. 그래서 `terminationGracePeriodSeconds`(기본 30초)를 다 기다린 뒤 강제 종료(SIGKILL)한다. 실무에서 "파드가 왜 이렇게 늦게 죽지?"의 흔한 원인이다.

### 6-2. 잘못된 해결: liveness를 느슨하게?
`initialDelaySeconds: 60`을 주면 해결은 된다. 하지만 그러면 **앱이 정상 운영 중에 멈췄을 때도** 감지가 그만큼 늦어진다.

### 6-3. 올바른 해결: startupProbe 추가
`livenessProbe` 블록 **아래에** 추가:
```yaml
          startupProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 5
            failureThreshold: 12   # 5초 × 12 = 최대 60초까지 시작을 기다려 줌
```
```powershell
kubectl apply -f .
kubectl get pods -n lab -w
```
✅ 약 30초 뒤 `1/1`이 되고, 재시작 없이 롤링 업데이트가 끝난다.

### 🔍 정리
- **startupProbe가 성공하기 전까지는 liveness와 readiness가 동작하지 않는다.**
- 시작은 넉넉하게 기다려 주고(startup), 시작한 뒤에는 민감하게 감시(liveness)할 수 있다.
- 실무 예시: JVM 앱(Spring Boot 등)처럼 기동이 느린 앱

### 원복
실습이 끝나면 `command:` 줄을 지워서 바로 시작하도록 돌려놓는다. startupProbe는 남겨둬도 된다.
```powershell
kubectl apply -f .
kubectl diff -f .
```

---

## 🧯 자주 만나는 상황

| 증상 | 원인 / 확인 |
|---|---|
| `Running`인데 `0/1` | readiness 실패. `describe pod` → Events의 `Readiness probe failed` |
| RESTARTS가 계속 증가 | liveness 실패, 또는 앱 자체 크래시. `kubectl logs <파드> --previous`로 **죽기 직전 로그** 확인 |
| `connection refused` | 앱이 아직 포트를 안 열었다 (시작 중이거나 포트 번호가 틀림) |
| `statuscode: 404` | probe의 `path` 오타 |
| `context deadline exceeded` | 응답이 `timeoutSeconds`(기본 1초)보다 느림 |

---

## 🧹 정리
```powershell
kubectl delete -f .        # lab 네임스페이스는 유지
```

---

## ✅ 체크리스트
- [ ] 01 파일 복사 후 readiness와 liveness probe 추가
- [ ] `describe pod`에서 probe 설정 확인
- [ ] ① readiness disable → `0/1`, 트래픽 제외, 재시작 없음 확인
- [ ] ② `PODINFO_UNREADY`로 롤링 업데이트가 멈추는 것 확인
- [ ] ②의 ❓ `apply`로 복구되는지 직접 확인하고 이유 생각해 보기
- [ ] ③ `PODINFO_UNHEALTHY`로 재시작 반복 확인
- [ ] ④ 느린 시작 + liveness → 무한 재시작 확인
- [ ] ④ startupProbe로 해결
- [ ] `git diff --no-index ..\01-basic-deploy\deployment.yaml deployment.yaml`로 01과 달라진 점 확인

## 다음 단계
➡️ 03-resources-hpa: requests/limits, OOMKilled, HPA 오토스케일링
