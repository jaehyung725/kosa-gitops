# Argo CD GitOps 구현 및 테스트 과정 정리

대상 저장소: `C:/Users/Owner/kosa-gitops`

이 문서는 `kosa-gitops` 저장소의 YAML 파일들과 실제 테스트 history를 기준으로, 구현 순서와 검증 명령어를 팀원들이 이해하기 쉽게 정리한 문서이다.

공유 시 주의:

- Secret, 비밀번호, Access Key, Token 값은 반드시 `<MASKED>` 처리한다.
- `harborAdminPassword`, Jenkins admin password, S3 access key, Argo CD 초기 admin password는 원문 공유 금지.

## 1. 클러스터 및 Argo CD 기본 상태 확인

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
        ├── harbor.yaml
        ├── jenkins.yaml
        ├── monitoring.yaml
        ├── redis.yaml
        └── ticket-app.yaml
```

목적:

- Kubernetes 클러스터와 Argo CD 구성요소가 정상 동작하는지 확인한다.

사용 명령어:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pods -n argocd
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

## 2. root-app 기반 App-of-Apps 적용

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
        ├── harbor.yaml
        ├── jenkins.yaml
        ├── monitoring.yaml
        ├── redis.yaml
        └── ticket-app.yaml
```

목적:

- `root-app`이 `apps/_applications/` 하위 Application YAML들을 읽어서 Argo CD Application으로 생성하는지 확인한다.

사용 명령어:

```bash
kubectl apply -f ~/kosa-gitops/bootstrap/root-app.yaml
kubectl -n argocd get app root-app
kubectl -n argocd get app
```

강제 refresh 및 sync:

```bash
kubectl -n argocd annotate app root-app argocd.argoproj.io/refresh=hard --overwrite
kubectl -n argocd patch app root-app --type=merge -p '{"operation":{"sync":{}}}'
kubectl -n argocd get app -w
```

확인 대상:

```text
cert-manager
cert-manager-issuer
demo-nginx
auto-demo
redis
monitoring
ticket-app
harbor
jenkins
```

## 3. GitOps 변경 반영 방식

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    └── _applications/
        ├── harbor.yaml
        ├── jenkins.yaml
        ├── monitoring.yaml
        ├── redis.yaml
        └── ticket-app.yaml
```

목적:

- YAML 수정 후 Git push를 통해 Argo CD가 변경사항을 감지하고 반영하는지 확인한다.

사용 명령어:

```bash
cd ~/kosa-gitops
git status
git add apps/_applications/<app>.yaml
git commit -m "<commit message>"
git push
```

즉시 반영:

```bash
kubectl -n argocd annotate app <app-name> argocd.argoproj.io/refresh=hard --overwrite
kubectl -n argocd patch app <app-name> --type=merge -p '{"operation":{"sync":{}}}'
kubectl -n argocd get app <app-name>
```

## 4. StorageClass 및 Ceph CSI 확인

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    └── _applications/
        ├── harbor.yaml
        ├── jenkins.yaml
        ├── monitoring.yaml
        └── redis.yaml
```

목적:

- Redis, Monitoring, Harbor, Jenkins가 사용하는 `team2-rbd-block` StorageClass와 Ceph CSI 상태를 확인한다.

사용 명령어:

```bash
kubectl get sc
kubectl get sc team2-rbd-block -o yaml
kubectl get csidrivers
kubectl get pods -n ceph-csi-rbd -o wide
kubectl get ds -A | grep -i ceph
```

Ceph CSI 설정 확인:

```bash
kubectl get cm -n ceph-csi-rbd ceph-csi-config -o jsonpath='{.data.config\.json}' | python3 -m json.tool
kubectl get secret -n ceph-csi-rbd -o name
```

PVC, PV, VolumeAttachment 확인:

```bash
kubectl get pvc -A
kubectl get pv
kubectl get volumeattachment -o wide
```

## 5. Redis 배포 및 장애 복구 테스트

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    ├── _applications/
    │   └── redis.yaml
    └── redis/
        ├── application.yaml
        └── values.yaml
```

목적:

- Bitnami Redis Helm chart 기반 배포 상태와 StatefulSet Pod 재생성을 확인한다.

상태 확인:

```bash
kubectl -n argocd get app redis
kubectl get pods -n redis -o wide
kubectl get pvc -n redis
```

장애 복구 테스트:

```bash
kubectl delete pod -n redis kosa-redis-node-0 --grace-period=0 --force
kubectl get pods -n redis -w
```

Pod 상세 확인:

```bash
kubectl describe pod -n redis kosa-redis-node-0 | tail -20
```

## 6. Monitoring 배포 및 Grafana 이슈 해결

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    ├── _applications/
    │   └── monitoring.yaml
    └── monitoring/
        ├── application.yaml
        └── values.yaml
```

목적:

- `kube-prometheus-stack` 배포와 Grafana PVC 관련 문제를 해결한다.

상태 확인:

```bash
kubectl -n argocd get app monitoring
kubectl get pods -n monitoring
kubectl get pvc -n monitoring
```

Sync 실패 상세 확인:

```bash
kubectl -n argocd get app monitoring -o jsonpath='{.status.operationState.message}'
kubectl -n argocd get app monitoring -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

Grafana RWO PVC 충돌 해결:

```bash
vi ~/kosa-gitops/apps/_applications/monitoring.yaml

cd ~/kosa-gitops
git add apps/_applications/monitoring.yaml
git commit -m "Use Recreate strategy for Grafana to avoid RWO PVC contention"
git push
```

강제 sync:

```bash
kubectl -n argocd patch app monitoring --type=merge -p '{"operation":{"sync":{"syncOptions":["Replace=true"]}}}'
kubectl -n argocd get app monitoring
```

## 7. Ticket App 배포 확인

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    ├── _applications/
    │   └── ticket-app.yaml
    └── ticket-app/
        ├── deployment.yaml
        ├── ingress.yaml
        ├── namespace.yaml
        └── service.yaml
```

목적:

- Harbor 이미지 기반 ticket-app 배포, Service, Ingress, TLS 연결을 확인한다.

상태 확인:

```bash
kubectl -n argocd get app ticket-app
kubectl get pods -n kosa-tickets -o wide
kubectl get svc -n kosa-tickets
kubectl get ingress -n kosa-tickets
```

Ingress 및 접속 확인:

```bash
kubectl describe ingress -n kosa-tickets ticket-app
curl -k -I https://ticket.kosa.team2
```

앱 Pod 확인:

```bash
kubectl describe pod -n kosa-tickets <ticket-app-pod>
kubectl logs -n kosa-tickets <ticket-app-pod> --tail=50
```

## 8. Harbor 배포 구현

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    └── _applications/
        └── harbor.yaml
```

목적:

- Harbor registry를 Helm chart로 배포하고, Ceph RGW S3 backend를 사용하도록 구성한다.

Application 추가:

```bash
vi ~/kosa-gitops/apps/_applications/harbor.yaml

cd ~/kosa-gitops
git add apps/_applications/harbor.yaml
git commit -m "Add Harbor registry with Ceph RGW S3 backend"
git push
```

Argo CD 동기화:

```bash
kubectl -n argocd get app harbor
kubectl -n argocd annotate app harbor argocd.argoproj.io/refresh=hard --overwrite
kubectl -n argocd patch app harbor --type=merge -p '{"operation":{"sync":{}}}'
kubectl get pods -n harbor -o wide
```

PVC 및 VolumeAttachment 확인:

```bash
kubectl get pvc -n harbor
kubectl get pv | grep harbor
kubectl get volumeattachment | grep harbor
kubectl get events -n harbor --sort-by=.lastTimestamp | tail -20
```

## 9. Harbor 오류 해결

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    └── _applications/
        └── harbor.yaml
```

목적:

- VolumeAttachment stuck, S3 bucket 미생성, Ingress routing 문제를 해결한다.

VolumeAttachment stuck 해결:

```bash
kubectl patch volumeattachment <VA_NAME> -p '{"metadata":{"finalizers":null}}' --type=merge
kubectl delete volumeattachment <VA_NAME> --grace-period=0 --force
```

Harbor S3 bucket 생성:

```bash
AWS_ACCESS_KEY_ID='<MASKED>' AWS_SECRET_ACCESS_KEY='<MASKED>' \
aws --endpoint-url http://10.10.10.11:7480 --region us-east-1 \
s3 mb s3://harbor-registry
```

Harbor Ingress 확인:

```bash
kubectl get ingress -n harbor
kubectl describe ingress -n harbor harbor-ingress
curl -k -I https://harbor.kosa.team2
```

HAProxy 라우팅 반영:

```bash
ssh -i ~/.ssh/kosa_iac ubuntu@<HAPROXY_NODE> "sudo vi /etc/haproxy/haproxy.cfg"
ssh -i ~/.ssh/kosa_iac ubuntu@<HAPROXY_NODE> "sudo haproxy -c -f /etc/haproxy/haproxy.cfg"
ssh -i ~/.ssh/kosa_iac ubuntu@<HAPROXY_NODE> "sudo systemctl reload haproxy"
```

Docker push 테스트:

```bash
docker login harbor.kosa.team2
docker pull nginx:alpine
docker tag nginx:alpine harbor.kosa.team2/library/nginx:test
docker push harbor.kosa.team2/library/nginx:test
```

## 10. Jenkins 배포 구현

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    └── _applications/
        └── jenkins.yaml
```

목적:

- Jenkins를 Helm chart로 배포하고, Ingress 및 Kubernetes dynamic agent 구성을 적용한다.

Application 추가:

```bash
vi ~/kosa-gitops/apps/_applications/jenkins.yaml

cd ~/kosa-gitops
git add apps/_applications/jenkins.yaml
git commit -m "Add Jenkins CI with K8s dynamic agents"
git push
```

root-app 동기화:

```bash
kubectl -n argocd patch app root-app --type=merge -p '{"operation":{"sync":{}}}'
kubectl -n argocd get app jenkins
kubectl get pods -n jenkins -w
```

상태 확인:

```bash
kubectl -n argocd get app jenkins
kubectl get pods -n jenkins -o wide
kubectl describe pod -n jenkins jenkins-0 | tail -40
kubectl logs -n jenkins jenkins-0 -c init --tail=50
```

## 11. Jenkins 오류 해결

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    └── _applications/
        └── jenkins.yaml
```

목적:

- Jenkins chart API 변경, admin 설정, 이미지 및 플러그인 호환성 문제를 해결한다.

Chart API 및 admin 설정 수정:

```bash
vi ~/kosa-gitops/apps/_applications/jenkins.yaml

git add apps/_applications/jenkins.yaml
git commit -m "Fix Jenkins admin to new chart API"
git push
```

이미지 및 플러그인 호환성 수정:

```bash
vi ~/kosa-gitops/apps/_applications/jenkins.yaml

git add apps/_applications/jenkins.yaml
git commit -m "Bump Jenkins image to compatible version"
git push
```

강제 재기동:

```bash
kubectl -n argocd patch app jenkins --type=merge -p '{"operation":{"sync":{}}}'
kubectl delete pod jenkins-0 -n jenkins --grace-period=0 --force
kubectl get pods -n jenkins -w
```

접속 확인:

```bash
curl -k -I https://jenkins.kosa.team2
```

관리자 계정 확인:

```bash
kubectl get secret jenkins -n jenkins -o jsonpath='{.data.jenkins-admin-user}' | base64 -d
kubectl get secret jenkins -n jenkins -o jsonpath='{.data.jenkins-admin-password}' | base64 -d
```

## 12. cert-manager 및 ClusterIssuer 구성

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    ├── _applications/
    │   ├── cert-manager.yaml
    │   └── cert-manager-issuer.yaml
    ├── cert-manager/
    │   └── cert-manager.yaml
    └── cert-manager-issuer/
        ├── ca-secret.yaml
        └── clusterissuer.yaml
```

목적:

- Ingress TLS 인증서 발급을 위해 cert-manager와 CA 기반 ClusterIssuer를 구성한다.

상태 확인:

```bash
kubectl -n argocd get app cert-manager
kubectl -n argocd get app cert-manager-issuer
kubectl get pods -n cert-manager
kubectl get clusterissuer
```

인증서 확인:

```bash
kubectl get certificate -A
kubectl describe certificate -n harbor harbor-tls
kubectl describe certificate -n jenkins jenkins-tls
```

CA Secret 확인:

```bash
kubectl get secret -n cert-manager kosa-ca-secret
```

## 13. demo-nginx 및 auto-demo 테스트

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    ├── _applications/
    │   ├── auto-demo.yaml
    │   └── demo-nginx.yaml
    ├── auto-demo/
    │   └── all.yaml
    └── demo-nginx/
        └── all.yaml
```

목적:

- 간단한 샘플 앱으로 Argo CD 자동 sync, prune, selfHeal 동작을 확인한다.

상태 확인:

```bash
kubectl -n argocd get app demo-nginx
kubectl -n argocd get app auto-demo
kubectl get pods -n demo
kubectl get pods -n auto-demo
```

강제 sync:

```bash
kubectl -n argocd patch app demo-nginx --type=merge -p '{"operation":{"sync":{}}}'
kubectl -n argocd patch app auto-demo --type=merge -p '{"operation":{"sync":{}}}'
```

selfHeal 테스트 예시:

```bash
kubectl scale deployment -n demo <deployment-name> --replicas=0
kubectl -n argocd get app demo-nginx
kubectl get pods -n demo
```

## 14. 노드 장애 및 Pod 복구 테스트

관련 디렉토리 구조:

```text
kosa-gitops/
└── apps/
    └── _applications/
        ├── harbor.yaml
        ├── jenkins.yaml
        ├── monitoring.yaml
        └── redis.yaml
```

목적:

- worker 노드 재기동 시 Stateful workload와 CSI volume attach 복구를 확인한다.

노드 cordon:

```bash
kubectl cordon k8s-w2
kubectl get nodes
```

VM 재기동:

```bash
ssh root@<PROXMOX_HOST> 'qm shutdown <VM_ID> && sleep 15 && qm start <VM_ID>'
sleep 60
kubectl get nodes
```

Unknown Pod 정리:

```bash
kubectl get pods -A -o wide | awk '$8=="k8s-w2" && $4=="Unknown" {print $1, $2}' | \
while read ns name; do
  kubectl delete pod -n "$ns" "$name" --grace-period=0 --force
done
```

노드 복구:

```bash
kubectl uncordon k8s-w2
kubectl get nodes
kubectl get pods -A -o wide | grep k8s-w2
```

## 15. 최종 상태 점검

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

전체 Application 확인:

```bash
kubectl -n argocd get app
```

비정상 Pod 확인:

```bash
kubectl get pods -A --no-headers | awk '$4!="Running" && $4!="Completed" {print}'
```

Service 및 Ingress 확인:

```bash
kubectl get svc -A
kubectl get ingress -A
kubectl get ipaddresspool -A
```

노드 리소스 확인:

```bash
kubectl get nodes
kubectl top nodes
```

외부 접속 확인:

```bash
curl -k -I https://argocd.kosa.team2
curl -k -I https://ticket.kosa.team2
curl -k -I https://harbor.kosa.team2
curl -k -I https://jenkins.kosa.team2
```

## 16. 핵심 요약

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
    ├── cert-manager/
    ├── cert-manager-issuer/
    ├── demo-nginx/
    ├── monitoring/
    ├── redis/
    └── ticket-app/
```

정리:

- `bootstrap/root-app.yaml`로 Argo CD App-of-Apps 구조를 구성했다.
- `root-app`은 `apps/_applications/` 하위 Application들을 자동 관리한다.
- Redis, Monitoring, Harbor, Jenkins는 Helm chart 기반 Argo CD Application으로 배포했다.
- Ticket App, cert-manager issuer, demo app은 repo 내부 manifest 기반으로 배포했다.
- 공통 스토리지는 `team2-rbd-block` StorageClass를 사용했다.
- Harbor는 Ceph RGW S3 backend를 사용하도록 구성했다.
- Jenkins는 Kubernetes dynamic agent, Ingress, PVC 기반으로 구성했다.
- cert-manager와 `kosa-ca-issuer`를 통해 Harbor, Jenkins, Ticket App Ingress TLS를 처리했다.
- 주요 오류는 `VolumeAttachment stuck`, Grafana RWO PVC 충돌, Harbor S3 bucket 미생성, Ingress class 미반영, Jenkins chart/image 호환성 문제였다.
- 오류 해결은 YAML 수정, Git push, Argo CD hard refresh/sync, VolumeAttachment finalizer 제거, HAProxy 설정 반영, S3 bucket 생성, Jenkins 이미지 버전 수정 순서로 진행했다.
- 최종적으로 Argo CD 구성 Pod는 모두 `Running` 상태이며, GitOps 기반 배포 흐름이 동작하는 것을 확인했다.
