# Argo CD GitOps 테스트 구현 과정

대상 저장소: `C:/Users/Owner/kosa-gitops`

이 문서는 `kosa-gitops`의 YAML 파일을 기준으로 Argo CD GitOps 테스트를 진행한 과정을 명령어 중심으로 정리한 것이다. Jenkins, Harbor처럼 Argo CD 자체 테스트 범위를 벗어나는 개별 서비스 구축/장애 처리 내용은 제외했다.

## 1. 테스트 대상 YAML 구조 확인

관련 디렉토리 구조:

```text
kosa-gitops/
├── bootstrap/
│   └── root-app.yaml
└── apps/
    ├── _applications/
    │   ├── auto-demo.yaml
    │   ├── cert-manager.yaml
    │   ├── cert-manager-issuer.yaml
    │   ├── demo-nginx.yaml
    │   ├── harbor.yaml
    │   ├── jenkins.yaml
    │   ├── monitoring.yaml
    │   ├── redis.yaml
    │   └── ticket-app.yaml
    ├── auto-demo/
    │   └── all.yaml
    ├── cert-manager/
    │   └── cert-manager.yaml
    ├── cert-manager-issuer/
    │   ├── ca-secret.yaml
    │   └── clusterissuer.yaml
    ├── demo-nginx/
    │   └── all.yaml
    ├── monitoring/
    │   ├── application.yaml
    │   └── values.yaml
    ├── redis/
    │   ├── application.yaml
    │   └── values.yaml
    └── ticket-app/
        ├── deployment.yaml
        ├── ingress.yaml
        ├── namespace.yaml
        └── service.yaml
```

확인 명령어:

```bash
cd ~/kosa-gitops
ls
ls apps/
ls apps/_applications/
cat bootstrap/root-app.yaml
```

확인 내용:

- `bootstrap/root-app.yaml`이 Argo CD 최상위 Application이다.
- `root-app`은 `apps/_applications/` 경로를 바라본다.
- `apps/_applications/*.yaml` 파일들이 하위 Argo CD Application 역할을 한다.

## 2. Argo CD Pod 및 서비스 상태 확인

관련 디렉토리 구조:

```text
kosa-gitops/
└── bootstrap/
    └── root-app.yaml
```

확인 명령어:

```bash
kubectl get pods -n argocd
kubectl -n argocd get svc argocd-server
kubectl -n argocd get app
```

정상 기준:

```text
argocd-application-controller      Running
argocd-applicationset-controller   Running
argocd-dex-server                  Running
argocd-notifications-controller    Running
argocd-redis                       Running
argocd-repo-server                 Running
argocd-server                      Running
```

현재 확인 결과:

```bash
kubectl get pods -n argocd
```

```text
argocd-application-controller-0                    1/1 Running
argocd-applicationset-controller-5b654ff98-wxsxf   1/1 Running
argocd-dex-server-694dfd7fc5-722vm                 1/1 Running
argocd-notifications-controller-58c5965756-ppsfp   1/1 Running
argocd-redis-6b6f94d995-gssjp                      1/1 Running
argocd-repo-server-ddfd59675-r9zpn                 1/1 Running
argocd-server-5d99678b59-glbkz                     1/1 Running
```

## 3. Argo CD UI 접속 확인

관련 디렉토리 구조:

```text
kosa-gitops/
└── bootstrap/
    └── root-app.yaml
```

port-forward 명령어:

```bash
kubectl -n argocd port-forward --address 0.0.0.0 svc/argocd-server 8080:443
```

브라우저 접속:

```text
https://<master-node-ip>:8080
```

접속 확인 로그:

```text
Forwarding from 0.0.0.0:8080 -> 8080
Handling connection for 8080
```

초기 admin password 확인:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

확인 내용:

- Argo CD UI 접속 성공
- `root-app` 카드 확인
- Application 상태 화면 진입 가능

## 4. root-app 적용 및 App-of-Apps 동작 확인

관련 디렉토리 구조:

```text
kosa-gitops/
├── bootstrap/
│   └── root-app.yaml
└── apps/
    └── _applications/
        ├── auto-demo.yaml
        ├── cert-manager.yaml
        ├── cert-manager-issuer.yaml
        ├── demo-nginx.yaml
        ├── monitoring.yaml
        ├── redis.yaml
        └── ticket-app.yaml
```

root-app 적용:

```bash
kubectl apply -f ~/kosa-gitops/bootstrap/root-app.yaml
```

root-app 상태 확인:

```bash
kubectl -n argocd get app root-app
kubectl -n argocd describe app root-app
```

하위 Application 생성 확인:

```bash
kubectl -n argocd get app
```

확인 내용:

- `root-app`이 `apps/_applications/` 경로를 읽는다.
- 하위 Application들이 Argo CD에 생성된다.
- 이 단계의 핵심은 각 앱이 모두 Healthy가 되는 것이 아니라, Argo CD가 Git repo를 읽고 Application 리소스를 생성하는지 확인하는 것이다.

## 5. Argo CD Refresh 및 Sync 테스트

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    └── _applications/
        ├── auto-demo.yaml
        ├── cert-manager.yaml
        ├── cert-manager-issuer.yaml
        ├── demo-nginx.yaml
        ├── monitoring.yaml
        ├── redis.yaml
        └── ticket-app.yaml
```

root-app 강제 refresh:

```bash
kubectl -n argocd annotate app root-app argocd.argoproj.io/refresh=hard --overwrite
```

root-app 수동 sync:

```bash
kubectl -n argocd patch app root-app --type=merge -p '{"operation":{"sync":{}}}'
```

전체 Application 상태 확인:

```bash
kubectl -n argocd get app
kubectl -n argocd get app -w
```

개별 Application sync 예시:

```bash
kubectl -n argocd patch app demo-nginx --type=merge -p '{"operation":{"sync":{}}}'
kubectl -n argocd patch app auto-demo --type=merge -p '{"operation":{"sync":{}}}'
```

확인 내용:

- Argo CD가 Git 변경사항을 감지한다.
- `OutOfSync` 상태를 수동 sync로 반영할 수 있다.
- `root-app` sync를 통해 하위 Application 변경사항이 관리된다.

## 6. Git 변경사항 반영 테스트

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    ├── _applications/
    │   ├── demo-nginx.yaml
    │   └── auto-demo.yaml
    ├── demo-nginx/
    │   └── all.yaml
    └── auto-demo/
        └── all.yaml
```

YAML 수정 후 Git 반영:

```bash
cd ~/kosa-gitops
git status
git add apps/_applications/<app>.yaml
git commit -m "<commit message>"
git push
```

Argo CD hard refresh:

```bash
kubectl -n argocd annotate app root-app argocd.argoproj.io/refresh=hard --overwrite
kubectl -n argocd get app
```

변경사항 반영 확인:

```bash
kubectl -n argocd get app <app-name>
kubectl -n argocd describe app <app-name>
```

확인 내용:

- Git push 이후 Argo CD가 변경사항을 인식한다.
- 필요 시 hard refresh로 즉시 갱신할 수 있다.
- Git repository가 desired state 역할을 한다.

## 7. selfHeal 및 prune 동작 확인

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    ├── _applications/
    │   ├── demo-nginx.yaml
    │   └── auto-demo.yaml
    ├── demo-nginx/
    │   └── all.yaml
    └── auto-demo/
        └── all.yaml
```

selfHeal 테스트 예시:

```bash
kubectl scale deployment -n demo <deployment-name> --replicas=0
kubectl -n argocd get app demo-nginx
kubectl get pods -n demo
```

prune 확인용 상태 조회:

```bash
kubectl -n argocd get app demo-nginx
kubectl -n argocd describe app demo-nginx
kubectl get all -n demo
```

확인 내용:

- Git에 정의된 상태와 클러스터 상태가 달라지면 Argo CD가 차이를 감지한다.
- `selfHeal: true`인 Application은 클러스터 수동 변경을 Git 상태로 되돌린다.
- `prune: true`인 Application은 Git에서 삭제된 리소스를 클러스터에서도 정리한다.

## 8. 최종 점검

관련 디렉토리 구조:

```text
kosa-gitops/
├── bootstrap/
│   └── root-app.yaml
└── apps/
    ├── _applications/
    │   ├── auto-demo.yaml
    │   ├── cert-manager.yaml
    │   ├── cert-manager-issuer.yaml
    │   ├── demo-nginx.yaml
    │   ├── monitoring.yaml
    │   ├── redis.yaml
    │   └── ticket-app.yaml
    ├── auto-demo/
    │   └── all.yaml
    ├── cert-manager/
    │   └── cert-manager.yaml
    ├── cert-manager-issuer/
    │   ├── ca-secret.yaml
    │   └── clusterissuer.yaml
    ├── demo-nginx/
    │   └── all.yaml
    ├── monitoring/
    │   ├── application.yaml
    │   └── values.yaml
    ├── redis/
    │   ├── application.yaml
    │   └── values.yaml
    └── ticket-app/
        ├── deployment.yaml
        ├── ingress.yaml
        ├── namespace.yaml
        └── service.yaml
```

전체 Application 확인:

```bash
kubectl -n argocd get app
```

Argo CD Pod 확인:

```bash
kubectl get pods -n argocd
```

비정상 Pod 확인:

```bash
kubectl get pods -A --no-headers | awk '$4!="Running" && $4!="Completed" {print}'
```

Service 및 Ingress 확인:

```bash
kubectl get svc -A
kubectl get ingress -A
```

## 9. 제외한 내용

이번 문서에서는 Argo CD 테스트 흐름과 직접 관련이 낮은 항목을 제외했다.

제외 항목:

- Jenkins 설치, Jenkins chart 수정, Jenkins admin 계정 확인
- Harbor 설치, Harbor S3 backend, Docker push 테스트
- Ceph RGW bucket 생성 및 S3 access key 관련 명령
- HAProxy 외부 라우팅 수정
- Proxmox VM 재기동, worker 노드 장애 테스트
- Redis/PXC/Monitoring의 개별 장애 복구 상세
- VolumeAttachment finalizer 강제 삭제 상세
- 특정 서비스 비밀번호, Secret, Access Key 값

## 10. 핵심 요약

관련 디렉토리 구조:

```text
kosa-gitops/
├── bootstrap/
│   └── root-app.yaml
└── apps/
    ├── _applications/
    │   ├── auto-demo.yaml
    │   ├── cert-manager.yaml
    │   ├── cert-manager-issuer.yaml
    │   ├── demo-nginx.yaml
    │   ├── monitoring.yaml
    │   ├── redis.yaml
    │   └── ticket-app.yaml
    ├── auto-demo/
    ├── cert-manager/
    ├── cert-manager-issuer/
    ├── demo-nginx/
    ├── monitoring/
    ├── redis/
    └── ticket-app/
```

정리:

- `bootstrap/root-app.yaml`로 Argo CD App-of-Apps 구조를 구성했다.
- `root-app`은 `apps/_applications/` 경로를 바라보며 하위 Application들을 관리한다.
- Argo CD UI 접속은 `argocd-server` port-forward로 확인했다.
- `kubectl apply`, `annotate refresh`, `patch sync` 명령으로 root-app 적용과 동기화를 테스트했다.
- Git push 후 Argo CD가 변경사항을 감지하는지 확인했다.
- `selfHeal`과 `prune` 설정으로 Git desired state 기반 복구/정리 동작을 검증할 수 있다.
- 최종적으로 Argo CD 구성 Pod가 모두 `Running` 상태인 것을 확인했다.
