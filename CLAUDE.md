# KOSA 인프라 프로젝트 기본 구성

> **이 문서의 정체**
> KOSA Team2 4인 프로젝트(온프레미스 + Ceph + AWS 하이브리드)의 **운영 reference + AI 컨텍스트**.
> 명령어/IP/credential 등 lookup용 정보 + 트러블슈팅 메모 + 학습 가이드를 한 파일에 모음.
> 깊이 있는 narrative와 "왜"는 `docs/onprem/` 챕터별 문서를 참고.

---

## 이 문서 읽는 법

### 누구를 위한 문서인가
- **4인 팀원**: 본인 담당 외 영역 빠르게 따라잡기, 명령어 복붙, 장애 시 회복 절차
- **AI 어시스턴트**: 팀원이 본인 Claude/ChatGPT에 컨텍스트로 첨부 → 프로젝트 맥락 위에서 질문 가능
- **신규 합류자**: 학습 경로 따라가면서 docs/ 챕터와 병행 독해

### 다른 문서와의 관계
| 문서 | 성격 | 보는 시점 |
|---|---|---|
| **CLAUDE.md** (이 문서) | 운영 메모 + AI 컨텍스트 (dense, lookup용) | 작업 중·장애 중 빠른 참조 |
| `docs/onprem/` | 학습용 챕터 문서 (narrative, "왜" 중심) | 개념 학습·신규 합류·발표 자료 |
| `terraform/` | IaC 코드 (state는 별도) | AWS 인프라 변경 작업 시 |
| `~/kosa-gitops/` (bastion) | ArgoCD가 watch하는 K8s manifest | 앱 배포·이미지 태그 갱신 |

### AI에게 이렇게 물어봐 (예시 프롬프트)
- "이 프로젝트의 K8s API VIP가 죽으면 어떤 컴포넌트가 영향받아? 회복 절차도 알려줘"
- "내가 새로운 마이크로서비스를 배포하려면 sys1과 production worker 중 어디에 둬야 해? 판단 기준은?"
- "Harbor에 이미지 push할 때 `x509: certificate signed by unknown authority` 나오는 이유와 해결법은?"
- "지금 아키텍처에서 cert-manager가 발급한 cert와 자체 CA wildcard cert는 각각 어디서 쓰여?"
- "ArgoCD가 OutOfSync인데 Healthy면 어떤 상태고 뭘 확인해야 해?"
- "지금 인프라에서 가장 SPoF(단일 장애 지점)일 가능성이 큰 곳 3가지는?"
- "온프레-AWS burst가 트리거되려면 어떤 조건이 만족돼야 해? 현재 구현된 부분은?"

---

## 학습 경로 (Learning Path)

### 초심자 코스 (이 분야 처음 — 추천 순서)
1. **프로젝트 개요** — 이 문서 `## 프로젝트 개요` + `docs/onprem/01-overview.md`
2. **물리/네트워크** — `docs/onprem/02-physical-network.md` (VLAN, Spine-Leaf, pfSense)
3. **가상화 (Proxmox)** — `docs/onprem/03-proxmox.md` (왜 KVM? cloud-init?)
4. **분산 스토리지 (Ceph)** — `docs/onprem/04-ceph.md` (OSD/MON/RGW, BlueStore)
5. **Kubernetes** — `docs/onprem/05-kubernetes.md` (HA CP, CNI, Ingress, MetalLB)
6. **보안/TLS** — `docs/onprem/06-security-tls.md` (자체 CA, 이중 TLS)
7. **GitOps** — `docs/onprem/07-gitops-argocd.md` (App-of-Apps)
8. **레지스트리/CI/CD** — 08, 09, 10 챕터 (Harbor → Jenkins → 파이프라인)
9. **관측성** — `docs/onprem/11-observability.md`
10. **운영** — `docs/onprem/12-operations.md`

### 빠른 작업 코스 (이미 알고 있고, 특정 작업하러 옴)
| 하고 싶은 것 | 바로 가기 |
|---|---|
| 새 도메인 추가 | `## 도메인` → `### DNS 해결` → `### Edge HAProxy ACL` |
| Harbor에 이미지 push | `## Harbor` → `### Docker / containerd CA 신뢰 등록` |
| Jenkins 파이프라인 새로 만들기 | `## CI/CD 파이프라인` 의 Groovy 템플릿 |
| 노드 추가/교체 | `docs/onprem/12-operations.md` (예정) |
| 장애 회복 | `## 운영 노하우 (장애 회복 메모)` |
| 새 ArgoCD Application 추가 | `## ArgoCD 워크로드` → `### 일반적 ignoreDifferences` |

---

## 핵심 용어 사전 (Glossary)

> 이 프로젝트에서 자주 등장하지만 검색이 귀찮은 약어/개념만 모음. 상세 설명은 `docs/onprem/` 해당 챕터 참고.

### 네트워크
- **VLAN (Virtual LAN)**: 동일 물리 스위치에서 L2 트래픽을 논리적으로 분리. 우리 환경은 10/20/30/40으로 관리/DMZ/Internal/Guest 분리.
- **Spine-Leaf**: 데이터센터 패브릭 토폴로지. Spine 2 / Leaf 5 구성, 모든 Leaf가 모든 Spine과 연결 (ECMP). Ceph 트래픽 격리에 사용.
- **CARP (Common Address Redundancy Protocol)**: pfSense HA에서 VIP를 두 노드가 공유. VRRP의 BSD 변형.
- **pfsync**: pfSense 두 노드 간 firewall state 테이블 동기화 (TCP 연결을 한 쪽에서 다른 쪽으로 이어받음).
- **XMLRPC Sync**: pfSense 설정 (NAT 룰, Host Override 등) 자동 미러링.
- **VRRP (Virtual Router Redundancy Protocol)**: Keepalived가 사용하는 VIP HA 프로토콜. 우리는 K8s API/Edge HAProxy에 사용.
- **LACP (802.3ad)**: 여러 NIC을 묶어 대역폭/HA 확보. 미사용 (10G 단일 링크).
- **MetalLB L2 mode**: 베어메탈 K8s에서 LoadBalancer Service IP를 ARP/NDP로 인근 네트워크에 advertise. 우리는 172.16.23.50.
- **NAT 1:1 vs PAT (Port Address Translation)**: pfSense WAN(192.168.21.109)이 인터넷에서 들어오는 트래픽을 내부 VIP(172.16.22.5)로 변환.
- **OPT 인터페이스**: pfSense에서 WAN/LAN 외 추가 인터페이스 (OPT1, OPT2, ...). VLAN 게이트웨이로 사용.

### 가상화 (Proxmox)
- **KVM (Kernel-based Virtual Machine)**: 리눅스 커널 내장 하이퍼바이저. Proxmox의 기반.
- **virtio**: VM ↔ 호스트 간 고성능 paravirt 디바이스 드라이버 (디스크/NIC).
- **cloud-init**: VM 첫 부팅 시 hostname/SSH key/네트워크 자동 설정. 우리 워커 VM 전부 cloud-init 템플릿 기반.
- **qemu-guest-agent**: VM 내부에서 동작하는 데몬, Proxmox에 메모리/IP 정확히 보고. 없으면 메모리 표시가 100%+로 오류.
- **vmbr0/vmbr1**: Proxmox bridge (Linux bridge). 물리 NIC + tap을 묶음. vmbr0=관리망, vmbr1=10G.
- **fwbr/fwln/fwpr**: Proxmox firewall 인터페이스 자동 생성 (VM당). 보통 신경 안 써도 됨.

### Ceph
- **OSD (Object Storage Daemon)**: 디스크 1개 = OSD 1개 (보통). 우리는 6 OSD (노드당 1TB HDD × 6).
- **MON (Monitor)**: 클러스터 상태/맵 보관. 홀수 개 권장 (3 or 5).
- **MGR (Manager)**: 메트릭/대시보드.
- **RGW (RADOS Gateway)**: S3/Swift 호환 객체 스토리지 게이트웨이. Harbor 백엔드.
- **RBD (RADOS Block Device)**: 가상 블록 디바이스. K8s PV (RWO).
- **CephFS**: POSIX 호환 분산 파일시스템 (RWX 가능). 현재 미사용.
- **BlueStore**: 최신 OSD 백엔드 (XFS+FileStore 대체). 디스크에 직접 쓰기 → 성능/일관성 ↑.
- **PG (Placement Group)**: 데이터 분배 단위. PG 수 = OSD 수 × ~100 / replica.
- **CRUSH (Controlled Replication Under Scalable Hashing)**: Ceph의 데이터 배치 알고리즘 (룩업 테이블 없이 위치 계산).
- **replica vs EC (Erasure Coding)**: 3-replica는 공간 1/3, 빠름. EC는 공간 효율 ↑, CPU/latency ↑.
- **HEALTH_OK / HEALTH_WARN / HEALTH_ERR**: 클러스터 상태. WARN은 즉시 조치 권장, ERR은 데이터 위험.
- **watcher**: RBD 이미지에 attach된 클라이언트 흔적. stale watcher → CSI 마운트 실패의 원인 1순위.

### Kubernetes
- **kubeadm**: K8s 클러스터 부트스트랩 표준 도구. 우리는 v1.30.
- **Control Plane (CP) vs Worker**: API/etcd/scheduler/controller-manager는 CP, 워크로드는 Worker. 우리는 CP×3 (stacked etcd) + Worker×4.
- **stacked etcd**: etcd가 CP 노드와 같은 머신에 있음 (separate etcd 대비 단순). 3 CP면 etcd quorum 만족.
- **CNI (Container Network Interface)**: Pod 네트워크 플러그인. 우리는 Calico (BGP/IPIP/VXLAN 가능, BGP는 미사용).
- **CSI (Container Storage Interface)**: 스토리지 플러그인. 우리는 ceph-csi-rbd.
- **kubelet**: 노드 에이전트, Pod 생명주기 관리.
- **kube-proxy**: Service ClusterIP → Pod 라우팅 (iptables/IPVS).
- **CoreDNS**: 클러스터 내부 DNS. 노드 resolv.conf upstream 사용.
- **Ingress (Controller)**: L7 라우팅 (Host/Path 기반). 우리는 HAProxy Ingress (`jcmoraisjr/haproxy-ingress` v0.16.1).
- **StorageClass / PVC / PV**: 동적 볼륨 프로비저닝. RWO=한 노드만 mount, RWX=여러 노드.
- **StatefulSet**: 안정적 ID/스토리지가 필요한 워크로드 (DB 등). volumeClaimTemplates는 immutable.
- **NodeSelector / Affinity / Taint·Toleration**: Pod의 노드 배치 제어. 우리는 nodeSelector `workload-type=system`으로 sys1 격리.
- **PDB (PodDisruptionBudget)**: drain 시 최소 가용 Pod 보장.
- **HPA (HorizontalPodAutoscaler)**: CPU/메모리/커스텀 메트릭 기반 replica 조절.

### 보안 / TLS
- **CA (Certificate Authority)**: 인증서 서명 권한. 우리는 자체 CA (`KOSA Team2 Internal CA`, 10년).
- **X.509**: 표준 인증서 포맷. CN, SAN, NotBefore/NotAfter 등.
- **SAN (Subject Alternative Name)**: cert에 매칭할 도메인 목록. wildcard `*.kosa.team2` 등.
- **wildcard cert**: `*.example.com` 매칭. 1단계 서브도메인만 (2단계는 X).
- **mTLS (mutual TLS)**: 양방향 인증. 우리는 미사용.
- **cert-manager**: K8s에서 cert 자동 발급/갱신. ClusterIssuer = 클러스터 전역 발급자.
- **ClusterIssuer vs Issuer**: 전자는 클러스터 전역, 후자는 namespace 한정.
- **이중 TLS**: Edge HAProxy에서 1차 종료 → 내부에서 2차 TLS로 다시 암호화 → HAProxy Ingress에서 종료. 외부/내부 모두 암호화 보장.
- **containerd certs.d**: containerd가 registry별 cert/credential을 찾는 경로 (`/etc/containerd/certs.d/<host>/hosts.toml`).

### GitOps / CI/CD
- **GitOps**: Git이 desired state의 single source of truth. ArgoCD/Flux가 reconcile.
- **App-of-Apps**: 하나의 ArgoCD Application이 다른 Application들을 정의 → 계층적 관리. 우리 root-app이 이 패턴.
- **Helm chart**: K8s manifest 템플릿 + values. 우리는 대부분 community chart 사용.
- **Sync vs OutOfSync vs Healthy vs Degraded**: 4가지 상태 조합. OutOfSync는 git과 cluster가 다름, Healthy는 워크로드가 정상.
- **selfHeal**: ArgoCD가 drift를 자동으로 git 상태로 되돌림.
- **ignoreDifferences**: 특정 필드는 drift 무시 (예: cert-manager가 갱신하는 caBundle).
- **Kaniko**: rootless 컨테이너 이미지 빌더. K8s Pod 안에서 안전하게 빌드.
- **GHCR (GitHub Container Registry)**: GitHub 무료 레지스트리. 우리는 Harbor로 마이그레이션 완료.
- **Harbor project**: 레지스트리 안의 namespace (`library`, `kosa-team2` 등).
- **imagePullSecret**: 사설 레지스트리 인증. Pod spec 레벨에 둠 (containers 안 X).
- **JCasC (Jenkins Configuration as Code)**: Jenkins 설정을 YAML로 관리.

### 관측 (Observability)
- **Prometheus**: pull 기반 메트릭 수집 + 시계열 DB.
- **scrape**: Prometheus가 target에서 메트릭 가져오는 행위.
- **ServiceMonitor / PodMonitor**: kube-prometheus-stack의 CRD. 무엇을 scrape할지 정의.
- **Grafana**: 대시보드/시각화.
- **Alertmanager**: Prometheus 알림을 그룹/라우팅/silence.
- **Recording Rule vs Alert Rule**: 전자는 쿼리 결과를 새 메트릭으로 저장 (성능), 후자는 조건 만족 시 알림.

---

## 프로젝트 개요

- **팀 규모**: 4인 인프라 프로젝트
- **목표 아키텍처**: 온프레미스 + Ceph + AWS 하이브리드 클라우드 환경
- **메인 워크로드**: Kubernetes (K8s) — 온프레미스 운영의 핵심
- **CI/CD**: ArgoCD 기반 GitOps 파이프라인

## 물리 장비 구성

- 라우터 1대
- 관리형 스위치 1대(port2: proxmox4대, port5: 관리 노트북 4대)
- 비관리형 스위치 2대
- JTCOM JT-S508CL-8S L3 8포트 10G 매니지드 스위치 7대
  - Spine 스위치 2대
  - Leaf 스위치 5대
  - Ceph 클러스터링용 Spine-Leaf 패브릭 구성
- Ceph 노드 6대 (별도 클러스터)
  - 각 노드당 1TB HDD 1개 → 총 6TB Raw 용량
  - OSD 백엔드: BlueStore
  - 10GbE Spine-Leaf 패브릭으로 Public/Cluster Network 연결

## pfSense (방화벽 / 라우터 / VLAN 게이트웨이)

- **역할**: 메인 방화벽/라우터, VLAN 10~40 게이트웨이 및 라우팅
- **이중화**: HA(High Availability) 구성 — CARP + pfsync + XMLRPC Sync
- **실제 배치**: Proxmox 4대 중 2대 위에 pfSense VM으로 얹어서 운영
  - 하드웨어 4대 제약으로 인한 현실적 선택
- **발표용 시나리오**: pfSense를 별도 전용 어플라이언스 2대로 분리 배치한 것처럼 설명
  - 토폴로지 다이어그램은 실제용/발표용 두 버전으로 관리 권장
- **주의점**:
  - pfSense VM이 올라간 Proxmox 노드 2대의 부팅 순서/네트워크 의존성이 핵심
  - 해당 노드는 VM autostart, HA 정책, CPU/메모리 우선순위 설정 필요
  - 관리망(vmbr0)이 pfSense 의존하지 않도록 OOB 경로 확보 필요

## Proxmox 노드 사양 (kosa1 기준, 4대 동일 가정)

- **시스템**: LG B80LV.AP37B7E (TA001), 마더보드 MICRO-STAR MS-BA03L
- **OS/커널**: Debian 13 (Trixie) / Kernel 6.17.13-2-pve (Proxmox VE)
- **CPU**: Intel Core i7-13700 (13세대 Raptor Lake), 16코어 24스레드 (8P+8E), L3 30MiB, 최대 5.2GHz, VMX(가상화) 지원
- **메모리**: 32 GiB (현재 사용률 약 75%)
- **GPU**: Intel UHD Graphics 770 (내장, i915 드라이버) — 헤드리스 운영
- **네트워크**:
  - eno1: Intel I219-V 1GbE (관리/업링크용 추정)
  - enp1s0f0: Intel 82599ES 10GbE SFP+ (up, 10Gbps) — Ceph/스토리지망 추정
  - enp1s0f1: Intel 82599ES 10GbE SFP+ (down, 예비 또는 미연결)
  - 브리지: vmbr0, vmbr1 (10Gbps), 다수의 fwbr/fwln/fwpr/tap 인터페이스 → 이미 VM 6대 정도 운영 중 (VMID 103, 104, 107, 111, 124, 127)
- **스토리지**:
  - NVMe: Solidigm SSDPFKNU512GZ 476.94 GiB (시스템 디스크, LVM `pve-root` 93.93 GiB / EFI / swap 8GiB)
  - HDD: Toshiba DT01ACA100 931.51 GiB (보조)
  - 총 1.38 TiB, 사용 9.5 GiB (0.7%)
- **가동 시간**: 16일 18시간 (안정 운영 중)

## Ceph 클러스터 구성

- **노드 수**: 6대 (Proxmox 4대와 분리된 별도 물리 클러스터)
- **OSD 디스크**: 노드당 1TB HDD × 1 = 총 6 OSD / 6TB Raw
- **OSD 백엔드**: BlueStore (XFS/FileStore가 아닌 차세대 백엔드)
  - WAL/DB는 동일 HDD에 함께 배치된 것으로 가정 (별도 SSD 분리 시 성능 향상 가능)
- **네트워크**: Spine 2 / Leaf 5 의 10GbE 패브릭에 연결
  - Public Network / Cluster(Replication) Network 분리 권장
- **예상 가용 용량**:
  - 3-replica 풀 사용 시: 6TB / 3 ≈ 2TB 가용
  - EC(Erasure Coding) 4+2 구성 시: 약 4TB 가용 (단, 6노드는 EC 최소 권장 수준)
- **활용 형태 (예정)**:
  - Kubernetes PV(PersistentVolume) 백엔드 — RBD(Block) / CephFS(File)
  - S3 호환 오브젝트 스토리지 — RGW(Rados Gateway)
  - Proxmox VM 디스크 백엔드 (선택)

## 주목할 포인트 / 고려사항

- **메모리 제약**: Proxmox 노드 32GB에 사용률 75% — VM 추가 시 여유가 빡빡함. (Ceph는 별도 6대 클러스터에 분리되어 있어 Proxmox 메모리 부담은 줄어듦)
- **Ceph 노드 메모리**: BlueStore OSD는 일반적으로 OSD당 4~8GB RAM 권장 → Ceph 노드당 최소 8GB 이상 확보 필요. (Ceph 노드 사양 별도 확인 필요)
- **네트워크 활용**: 10GbE 포트는 노드당 `enp1s0f0` 1개만 실사용 (enp1s0f1은 스위치 포트 부족 등으로 미활용). Ceph + K8s가 같은 10G NIC 공유. **실측 (2026-05-18)**: Pod-Pod = 5.34 Gbps → Calico 이미 10G 사용 중. 추가 최적화는 Jumbo frame 또는 BGP 모드 (필요 시). 상세: `docs/onprem/13-validation.md` §2.3.1
- **Ceph HDD 한계**: 1TB HDD × 6 = 6TB Raw로 본격적인 대용량 워크로드는 어려움 → SSD 추가 시 BlueStore WAL/DB 분리로 성능 보완 가능. 발표 시 "확장 시 SSD 캐시 티어/DB 분리" 로드맵 명시 권장.
- **중첩 가상화**: CPU에 VMX 플래그 있음 → Proxmox 위에 K8s, OpenStack 등 올릴 때 유용.

## 진행 가능한 다음 단계

### 온프레미스 (완료/안정화)
- [x] Proxmox 4노드 + Ceph 6노드 + pfSense HA
- [x] K8s 1.30 HA (CP×3 + Worker×4) + Calico + MetalLB
- [x] Ceph RBD CSI (PV) + RGW S3 (Harbor 백엔드)
- [x] cert-manager + 자체 CA + 이중 TLS (Edge HAProxy → HAProxy Ingress)
- [x] ArgoCD GitOps (App-of-Apps)
- [x] Prometheus / Grafana / Alertmanager (sys1 분리)
- [x] Harbor (Ceph RGW S3 백엔드, GHCR 마이그레이션 완료)
- [x] Jenkins CI (lts-jdk17, K8s dynamic agent, Kaniko)
- [x] CI/CD: GitHub → Jenkins → Kaniko → Harbor → GitOps → ArgoCD 검증

### 온프레미스 (선택)
- [ ] GitHub Webhook → Jenkins 자동 트리거 (현재는 수동 빌드)
- [ ] etcd backup CronJob (Ceph RBD PVC에 dump)
- [ ] DNS netplan 영구화 (별도 파일 `/etc/netplan/99-team2-dns.yaml`로 분리, cloud-init 덮어쓰기 방지)
- [ ] 부하 테스트 환경 (k6 / JMeter VM 또는 K8s Job)
- [ ] PXC + ProxySQL HA 클러스터 (현재 데모 앱은 in-memory)

### AWS 하이브리드 (진행 중)
- [x] **Phase 1**: VPC (10.20.0.0/16) + Public/Private Subnet × 2 AZ + NAT GW × 2 + NLB + EC2(HAProxy) × 2
- [x] **Phase 2**: Site-to-Site VPN (CGW + VGW + IPsec 양쪽 터널 UP, bastion↔EC2 ping 6ms 성공)
- [ ] **Phase 3**: AWS RDS MySQL Read Replica (Percona binlog → RDS 복제)
- [ ] **Phase 3**: DB Subnet tier 추가 (private의 하위, RDS 전용)
- [ ] **Phase 4**: EKS + Karpenter (Burst용, 평소 0 노드)
- [ ] **Phase 4**: CloudWatch + Lambda 자동 burst trigger (온프레 부하 임계 시 EKS scale-out)
- [ ] **Phase 4**: ArgoCD multi-cluster 등록 (온프레 → EKS apps 동시 배포)
- [ ] **Phase 5**: VPC Endpoints (S3 Gateway, ECR Interface — NAT 비용 절감)
- [ ] **Phase 5**: AWS WAF (SQL injection / XSS / Bot 차단)
- [ ] **Phase 5**: Route 53 hosted zone + ACM cert (실제 도메인 있을 시)
- [ ] **Phase 5**: `terraform/aws/` 코드화 (현재는 콘솔로 구축, IaC 전환)


라우터 ip: 192.168.21.1
proxmox ip:
192.168.21.2 kosa1.team2 kosa1
192.168.21.3 kosa2.team2 kosa2
192.168.21.4 kosa3.team2 kosa3
192.168.21.5 kosa4.team2 kosa4

ceph(10G):
  10.10.10.12

10G IP(팀원):
- kosa1
    10.10.10.35
- kosa2
    10.10.10.36
- kosa3
    10.10.10.37
- kosa4
    10.10.10.38

pfsense:
- vlan10
    Subnet 172.16.21.0/24
    Subnet Range 172.16.21.1 - 172.16.21.254
- vlan20
    Subnet 172.16.22.0/24
    Subnet Range 172.16.22.1 - 172.16.22.254
- vlan30(dhcp)
    Subnet 172.16.23.0/24
    Subnet Range 172.16.23.1 - 172.16.23.254
    Address Pool Range 172.16.23.100 - 172.16.23.200
- vlan40(dhcp)
    Subnet 172.16.24.0/24
    Subnet Range 172.16.24.1 - 172.16.24.254
    Address Pool Range 172.16.24.100 - 172.16.24.200


- 기술스택
    Proxmox, Kubernetes, calico ,ceph, pfsense, HAproxy+keepalive, redis, proxySQL, percona xtraDB cluster,
    prometeus+grafana, terraform, ansible, AWS, github action, argoCD, jmeter, iperfs, haproxy ingress

AWS 비용 무관하지만 50만원 내외로
현재 데모로 fastapi/python으로 코딩된 회원정보 등록/출력 이 있음(유지 or 확장 or 새로 바이브코딩)

## 현재 배포된 VM 전체 목록

| VM | VMID | PVE 노드 | IP (VLAN) | 10G IP (Ceph) | 역할 |
|---|---|---|---|---|---|
| k8s-cp1 | 210 | kosa4 | 172.16.23.10 (30) | - | K8s Control Plane |
| k8s-cp2 | 211 | kosa2 | 172.16.23.11 (30) | - | K8s Control Plane |
| k8s-cp3 | 212 | kosa3 | 172.16.23.12 (30) | - | K8s Control Plane |
| k8s-w1 | 220 | kosa3 | 172.16.23.20 (30) | 10.10.10.120 | K8s Worker (production, 6GB) |
| k8s-w2 | 221 | kosa4 | 172.16.23.21 (30) | 10.10.10.121 | K8s Worker (production, 6GB) |
| k8s-w3 | 222 | kosa2 | 172.16.23.22 (30) | 10.10.10.122 | K8s Worker (production, 6GB) |
| k8s-sys1 | 223 | kosa1 | 172.16.23.23 (30) | 10.10.10.123 | K8s Worker (system 전용, 16GB) |
| bastion | 230 | kosa3 | 172.16.24.10 (40) | - | Ansible runner, kubectl |
| lb-1 | 240 | kosa2 | 172.16.23.6 (30) | - | K8s API HA (HAProxy + Keepalived MASTER) |
| lb-2 | 241 | kosa4 | 172.16.23.7 (30) | - | K8s API HA (BACKUP) |
| edge-haproxy | 250 | kosa4 | 172.16.22.10 (20) | - | Edge L7 (TLS 종료 + Ingress 진입, MASTER) |
| edge-haproxy2 | 251 | kosa3 | 172.16.22.11 (20) | - | Edge L7 (BACKUP) |

### 노드 워크로드 분리

K8s 워커 노드는 라벨 `workload-type`으로 분리:

- **`workload-type=system`** (k8s-sys1): ArgoCD, 모니터링(Prometheus/Grafana/Alertmanager), Harbor, Jenkins, cert-manager 등 시스템 워크로드
- **`workload-type=production`** (k8s-w1/w2/w3): 비즈니스 워크로드 (ticket-app, PXC, Redis 등)

System 컴포넌트의 helm values에 `nodeSelector: workload-type=system`을 설정하여 sys1에 고정. PVC RWO 충돌이나 운영 노드 영향 최소화.

> **왜 분리하나?**
> 1. **PVC RWO 충돌 회피**: Prometheus/Grafana/Jenkins/Harbor는 RWO 볼륨을 점유함. 워크로드가 다른 노드로 schedule되면 Multi-Attach 에러로 Pending. 단일 노드(sys1) 고정 → 재schedule 무관하게 즉시 mount.
> 2. **운영 영향 격리**: 비즈니스 트래픽 폭증으로 production 노드가 OOM/CPU saturation → ArgoCD/모니터링까지 같이 죽으면 진단 불가. system은 별도 노드에 두면 "감시자"가 살아있음.
> 3. **메모리 풀 분리**: sys1=16GB (Prometheus TSDB 메모리 ↑), production=6GB (앱당 가볍게). 워크로드 특성에 맞춤.
> 4. **장기적 확장성**: production을 늘릴 때 system 영향 X, system을 늘릴 때(sys2 추가) production 영향 X.

### qemu-guest-agent

모든 K8s 워커 VM에 qemu-guest-agent 설치 + Proxmox에서 agent 활성화:
- 설치: `apt install qemu-guest-agent` (이미 cloud-init이 설치)
- Proxmox 활성화: `qm set <VMID> --agent enabled=1,fstrim_cloned_disks=1`
- 효과: Proxmox UI에서 메모리 사용량이 정확히 표시 (없으면 100%+ 잘못 표시됨)

## VIP 매트릭스

| VIP | VLAN | 용도 | Keepalived VRID |
|---|---|---|---|
| 172.16.22.5 | 20 (DMZ) | Edge HAProxy VIP (pfSense NAT 대상, HTTPS/HTTP) | 52 |
| 172.16.23.5 | 30 (Internal) | K8s API VIP (kubectl + kubelet endpoint, 6443) | 51 |

## 외부 트래픽 흐름 (이중 TLS)

```
[Internet] → pfSense WAN (192.168.21.109) → NAT → 172.16.22.5 (Edge VIP)
  → Edge HAProxy (172.16.22.10 또는 .11)
    → 1차 TLS 종료 (자체 CA wildcard *.kosa.team2)
    → Host 헤더 분기
    → 172.16.23.50:443 (K8s Ingress LB, MetalLB L2)
  → HAProxy Ingress Controller (K8s)
    → 2차 TLS 종료 (cert-manager 자동 발급, 같은 자체 CA)
    → Service ClusterIP
  → Pod
```

> **왜 TLS를 두 번 종료하나?** (`docs/onprem/06-security-tls.md`에 상세)
> - **DMZ↔Internal 경계에서 한번**: Edge HAProxy가 DMZ(VLAN 20)에 있으니 일단 받아서 cert/Host 검사 후 내부로 재암호화. "외부 노출 영역"과 "내부망"의 신뢰 경계 분리.
> - **K8s 클러스터 진입 직전 한번**: HAProxy Ingress가 클러스터 cert 정책(cert-manager)에 따라 다시 검증. 외부 cert와 클러스터 cert를 독립적으로 회전 가능.
> - **장점**: 내부 트래픽도 wire에서 cleartext가 아님 (10G NIC tap/감청 방지). 인증서 만료/회전이 한쪽씩 가능.
> - **비용**: TLS 핸드셰이크 2배. 우리 규모(QPS 낮음)에서는 무시 가능.

## 도메인

| 도메인 | 백엔드 | 비고 |
|---|---|---|
| ticket.kosa.team2 | ticket-app | 데모 앱 |
| grafana.kosa.team2 | kube-prom-grafana | 모니터링 UI |
| argocd.kosa.team2 | argocd-server | GitOps UI |
| harbor.kosa.team2 | harbor-core/portal | 컨테이너 레지스트리 (Ceph RGW S3 백엔드) |
| jenkins.kosa.team2 | jenkins | CI 서버 |

### DNS 해결 — pfSense DNS Resolver Host Overrides (정석)

**Services → DNS Resolver → Host Overrides**에 5개 등록 (모두 `kosa.team2` 도메인, IP `172.16.23.50`):
- harbor, jenkins, ticket, grafana, argocd

K8s 노드들의 `/etc/resolv.conf`가 pfSense를 가리키도록 netplan 수정:
```yaml
# /etc/netplan/50-cloud-init.yaml 또는 별도 파일
nameservers:
  addresses:
    - 172.16.24.2   # pfSense OPT4 (VLAN 40 게이트웨이)
    - 1.1.1.1       # fallback
  search:
    - kosa.team2
```

CoreDNS는 노드의 resolv.conf를 따라가므로 자동으로 pfSense를 upstream으로 사용 → cluster pod도 `*.kosa.team2` 해결 가능.

외부 노트북도 pfSense를 DNS로 쓰면 자동 작동. (`/etc/hosts` 수동 매핑 불필요)

### Edge HAProxy ACL

각 도메인은 Edge HAProxy의 `acl ... hdr(host) -i ...kosa.team2` 라인에 추가 필요. `/etc/haproxy/haproxy.cfg`:
```haproxy
acl ticket  hdr(host) -i ticket.kosa.team2
acl grafana hdr(host) -i grafana.kosa.team2
acl argo    hdr(host) -i argocd.kosa.team2
acl harbor  hdr(host) -i harbor.kosa.team2
acl jenkins hdr(host) -i jenkins.kosa.team2
use_backend k8s-ingress if ticket or grafana or argo or harbor or jenkins
```

수정 후 `systemctl reload haproxy`. MASTER/BACKUP 둘 다 수정 (172.16.22.10, 172.16.22.11).

## 자체 CA

bastion의 `~/pki/`:
- `ca.crt`, `ca.key` — KOSA Team2 Internal CA (10년)
- `wildcard.pem` — `*.kosa.team2` cert (1년, HAProxy Edge용)
- cert-manager는 K8s Secret `cert-manager/kosa-ca-secret`으로 CA 보유 + ClusterIssuer `kosa-ca-issuer`로 서비스별 cert 자동 발급

### Docker / containerd CA 신뢰 등록

Harbor(`harbor.kosa.team2`)는 자체 CA로 서명된 cert를 사용. Docker/containerd가 cert를 신뢰하지 않으면 push/pull 실패.

**bastion 등 Docker 사용 호스트**:
```bash
sudo mkdir -p /etc/docker/certs.d/harbor.kosa.team2
sudo cp ~/pki/ca.crt /etc/docker/certs.d/harbor.kosa.team2/ca.crt
# 시스템 신뢰도 추가
sudo cp ~/pki/ca.crt /usr/local/share/ca-certificates/kosa-ca.crt
sudo update-ca-certificates
sudo systemctl restart docker
```

**K8s 워커 노드 (containerd)**:
```bash
sudo mkdir -p /etc/containerd/certs.d/harbor.kosa.team2
sudo tee /etc/containerd/certs.d/harbor.kosa.team2/hosts.toml << 'EOF'
server = "https://harbor.kosa.team2"
[host."https://harbor.kosa.team2"]
  capabilities = ["pull", "resolve"]
  ca = "/usr/local/share/ca-certificates/kosa-ca.crt"
EOF
sudo cp ~/pki/ca.crt /usr/local/share/ca-certificates/kosa-ca.crt
sudo update-ca-certificates
# containerd config.toml에 config_path 추가 (없으면)
sudo systemctl restart containerd
```

## Prometheus / Grafana 시스템 메트릭

kubeadm 기본값은 K8s 시스템 컴포넌트 메트릭 포트를 127.0.0.1에만 binding → 외부 Prometheus가 scrape 불가. 다음을 0.0.0.0으로 변경 (Ansible `35-metrics-exposure.yml`로 자동화):

- `kube-controller-manager.yaml` — `--bind-address=0.0.0.0` (10257)
- `kube-scheduler.yaml` — `--bind-address=0.0.0.0` (10259)
- `etcd.yaml` — `--listen-metrics-urls=http://0.0.0.0:2381` (2381)
- `kube-proxy` ConfigMap — `metricsBindAddress: "0.0.0.0:10249"`

검증: `kubectl get --raw /api/v1/namespaces/monitoring/services/kube-prom-kube-prometheus-prometheus:web/proxy/api/v1/targets`

## Harbor (컨테이너 레지스트리) — Ceph RGW S3 백엔드

### 구성

- **Helm chart**: `harbor/harbor 1.16.0` (Harbor v2.12)
- **Namespace**: `harbor`
- **저장 백엔드**:
  - **이미지 blob**: Ceph RGW S3 (bucket `harbor-registry`)
  - **메타데이터 (DB/Redis/Trivy)**: Ceph RBD PVC
- **배치 노드**: k8s-sys1 (`workload-type=system`)
- **관리자**: admin / kosa1004
- **GitOps**: `~/kosa-gitops/apps/_applications/harbor.yaml`

### Ceph RGW 설정

- **RGW endpoint**: `http://10.10.10.11:7480` (ceph1에서만 실행, single daemon)
- **RGW user**: `harbor`
  - 생성: `radosgw-admin user create --uid=harbor --display-name="Harbor Registry"`
  - 키 재생성: `radosgw-admin key create --uid=harbor --gen-access-key`
  - quota: 200GB / 10M objects
- **K8s Secret** (Harbor가 참조): `harbor/harbor-s3-secret`
  - keys: `REGISTRY_STORAGE_S3_ACCESSKEY`, `REGISTRY_STORAGE_S3_SECRETKEY` (+ `accesskey`, `secretkey`)
- **Bucket 생성**: Harbor가 자동 생성 안 함. 수동 생성 필요:
  ```bash
  AWS_ACCESS_KEY_ID=<key> AWS_SECRET_ACCESS_KEY=<secret> \
    aws --endpoint-url http://10.10.10.11:7480 --region us-east-1 \
    s3 mb s3://harbor-registry
  ```

### Helm values 주요 설정

```yaml
expose:
  type: ingress
  ingress:
    hosts:
      core: harbor.kosa.team2
    className: haproxy
    annotations:
      kubernetes.io/ingress.class: haproxy    # 옛 jcmoraisjr/haproxy-ingress 호환 필수
      cert-manager.io/cluster-issuer: kosa-ca-issuer

persistence:
  imageChartStorage:
    type: s3                     # ← blob을 RBD가 아닌 RGW로. scale에 유리 (RGW는 수평 확장)
    disableredirect: true        # ← RGW는 기본적으로 client에 "이 URL로 다시 받으세요" redirect를 보냄.
                                 #    그 URL이 내부 IP(10.10.10.11)라 외부 client는 도달 불가 → 반드시 true.
    s3:
      existingSecret: harbor-s3-secret
      region: default            # ← RGW는 region 개념 없음. "default"가 관례. AWS CLI 호출 시는 us-east-1.
      bucket: harbor-registry    # ← 사전 수동 생성 필요 (Harbor가 자동 생성 안 함)
      regionendpoint: http://10.10.10.11:7480
      v4auth: true               # ← SigV4 강제. RGW는 v2/v4 모두 지원하나 v4가 표준.
      secure: false              # ← RGW 7480은 HTTP. 내부망이라 OK. 외부 노출 시 7443으로 변경 + secure:true

ignoreDifferences:              # StatefulSet volumeClaimTemplates immutable
  - group: apps
    kind: StatefulSet
    jsonPointers:
      - /spec/volumeClaimTemplates
```

### 알려진 함정

- **`adminUser` deprecated**: chart 5.x부터 `controller.admin.username` / `controller.admin.password`로 변경
- **HAProxy Ingress controller (jcmoraisjr/haproxy-ingress v0.16.1)**: `ingressClassName` 필드 무시, `kubernetes.io/ingress.class` 어노테이션만 인식
- **Image push 503 / 11h stuck**: K8s API VIP (172.16.23.5) 다운으로 노드들 unreachable되면 회복 후 cleanup 필요. lb-1/lb-2 keepalived 모니터링.
- **Image push 시 `s3aws: connection refused`**: Harbor registry pod의 connection state 캐시 문제. `kubectl delete pod -n harbor -l component=registry`로 재시작
- **`s3aws: NoSuchBucket`**: bucket 미생성. 위 aws cli로 수동 생성

### GHCR → Harbor 마이그레이션 (kosa-tickets 사례)

1. 이미지 가져와서 Harbor에 push:
   ```bash
   docker pull ghcr.io/kosacloudteam2/kosa-tickets:latest
   docker tag  ghcr.io/kosacloudteam2/kosa-tickets:latest harbor.kosa.team2/library/kosa-tickets:latest
   docker push harbor.kosa.team2/library/kosa-tickets:latest
   ```
2. GitOps repo의 deployment.yaml에서 image URL 변경 (`ghcr.io/kosacloudteam2` → `harbor.kosa.team2/library`)
3. private project면 `imagePullSecret` 추가 (pod spec 레벨, 컨테이너 안 X):
   ```yaml
   spec:
     imagePullSecrets:
       - name: harbor-pull-secret
     containers: [...]
   ```
4. 워커 노드 containerd가 Harbor cert 신뢰하도록 위 "Docker / containerd CA 신뢰 등록" 설정

## Jenkins (CI 서버) — Kubernetes 동적 Agent

### 구성

- **Helm chart**: `jenkins/jenkins` (charts.jenkins.io)
- **Namespace**: `jenkins`
- **이미지 태그**: `lts-jdk17` (최신 LTS)
- **배치 노드**: k8s-sys1 (`workload-type=system`)
- **관리자**: admin / kosa1004
- **URL**: https://jenkins.kosa.team2
- **GitOps**: `~/kosa-gitops/apps/_applications/jenkins.yaml`

### Helm values 주요 설정

```yaml
controller:
  admin:
    username: admin
    password: kosa1004
  image:
    tag: "lts-jdk17"
  numExecutors: 2
  nodeSelector:
    workload-type: system
  installPlugins:
    - kubernetes
    - workflow-aggregator
    - git
    - configuration-as-code
    - blueocean
    - credentials-binding
    - ssh-agent
  ingress:
    enabled: true
    hostName: jenkins.kosa.team2
    ingressClassName: haproxy
    annotations:
      kubernetes.io/ingress.class: haproxy
      cert-manager.io/cluster-issuer: kosa-ca-issuer

persistence:
  storageClass: team2-rbd-block
  size: 8Gi
```

### Jenkins Credentials (사전 등록)

Jenkins UI → Manage Jenkins → Credentials → System → Global credentials:

| ID | Type | 용도 |
|---|---|---|
| `harbor-creds` | Username with password | Harbor 로그인 (admin / kosa1004) |
| `kosa-gitops-ssh` | SSH Username with private key | GitHub 푸시용 (deploy key) |

### K8s Secret (Kaniko용)

`jenkins` 네임스페이스에 `harbor-creds-dockerconfigjson` Secret 사전 생성 — Kaniko가 Harbor에 push할 때 사용:

```bash
kubectl create secret -n jenkins docker-registry harbor-creds-dockerconfigjson \
  --docker-server=harbor.kosa.team2 \
  --docker-username=admin \
  --docker-password=kosa1004
```

### 글로벌 SSH 설정

Manage Jenkins → Security → Git Host Key Verification Strategy → **Accept first connection** (안 그러면 GitHub Host Key Verification Failed)

## CI/CD 파이프라인 (kosa-tickets)

### 전체 흐름

```
[GitHub: kosa-tickets repo] → [Jenkins (수동/Webhook 트리거)]
  → [Kaniko Pod] (rootless 이미지 빌드)
  → [Harbor] (이미지 push)
  → [GitOps repo: image tag 업데이트] (sed로 deployment.yaml 수정 → git push)
  → [ArgoCD auto-sync] (3분 polling 또는 webhook)
  → [K8s Deployment rollout]
  → [HAProxy Ingress → Pod 새 버전]
```

### 레포지토리

- **소스 (Source)**: `git@github.com:kosacloudteam2/kosa-tickets.git` — FastAPI 앱 + Dockerfile
- **GitOps**: `git@github.com:kosacloudteam2/kosa-gitops.git` — ArgoCD가 watch하는 K8s manifest

### Jenkins Pipeline (kosa-tickets-ci job)

```groovy
pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: kaniko
            image: gcr.io/kaniko-project/executor:v1.23.2-debug
            command: ["sleep"]
            args: ["infinity"]
            volumeMounts:
            - name: docker-config
              mountPath: /kaniko/.docker
          - name: git
            image: alpine/git:latest
            command: ["sleep"]
            args: ["infinity"]
          volumes:
          - name: docker-config
            projected:
              sources:
              - secret:
                  name: harbor-creds-dockerconfigjson
                  items:
                  - key: .dockerconfigjson
                    path: config.json
      '''
    }
  }
  environment {
    IMAGE  = "harbor.kosa.team2/library/kosa-tickets"
    TAG    = "${env.BUILD_NUMBER}"
    SOURCE = "git@github.com:kosacloudteam2/kosa-tickets.git"
    GITOPS = "git@github.com:kosacloudteam2/kosa-gitops.git"
  }
  stages {
    stage('Checkout Source') {
      steps {
        container('git') {
          dir('src') {
            git url: "${SOURCE}", credentialsId: 'kosa-gitops-ssh', branch: 'main'
          }
        }
      }
    }
    stage('Build & Push to Harbor') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor \
              --context=src \
              --dockerfile=src/Dockerfile \
              --destination=${IMAGE}:${TAG} \
              --destination=${IMAGE}:latest \
              --skip-tls-verify
          '''
        }
      }
    }
    stage('Update GitOps') {
      steps {
        container('git') {
          withCredentials([sshUserPrivateKey(credentialsId: 'kosa-gitops-ssh', keyFileVariable: 'SSH_KEY')]) {
            sh '''
              mkdir -p ~/.ssh
              ssh-keyscan github.com >> ~/.ssh/known_hosts 2>/dev/null
              export GIT_SSH_COMMAND="ssh -i $SSH_KEY -o StrictHostKeyChecking=no"
              git config --global user.email "jenkins@kosa.team2"
              git config --global user.name "Jenkins CI"
              git clone ${GITOPS} gitops
              cd gitops
              sed -i "s|image: harbor.kosa.team2/library/kosa-tickets:.*|image: harbor.kosa.team2/library/kosa-tickets:${TAG}|" apps/ticket-app/deployment.yaml
              git add -A
              git commit -m "CI: kosa-tickets ${TAG}" || echo "no changes"
              git push origin main
            '''
          }
        }
      }
    }
  }
}
```

### ticket-app deployment 핵심

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: harbor-pull-secret   # ★ pod spec 레벨 (containers 안 X)
      containers:
        - name: ticket-app
          image: harbor.kosa.team2/library/kosa-tickets:4   # Jenkins가 BUILD_NUMBER로 갱신
```

`harbor-pull-secret`은 `kosa-tickets` 네임스페이스에 미리 생성:
```bash
kubectl create secret -n kosa-tickets docker-registry harbor-pull-secret \
  --docker-server=harbor.kosa.team2 \
  --docker-username=admin \
  --docker-password=kosa1004
```

### 파이프라인 함정 / 해결

- **`sshagent` not found**: `ssh-agent` 플러그인 대신 `withCredentials([sshUserPrivateKey(...)])` 사용
- **GitHub Host Key Verification Failed**: Manage Jenkins → Security → "Accept first connection" 또는 파이프라인에서 `ssh-keyscan github.com >> ~/.ssh/known_hosts`
- **Kaniko push x509 (자체 CA)**: `--skip-tls-verify` 플래그 (또는 CA mount). 운영엔 CA mount 권장
- **Jenkins 2.492.x 플러그인 호환성 깨짐**: `image.tag: lts-jdk17`로 LTS 고정
- **Login Failed after PVC 변경**: PVC 완전 초기화 후 재시작이 가장 빠름

### 검증

- Jenkins Build #4 SUCCESS 확인 → Harbor `library/kosa-tickets:4` 이미지 push 확인 → GitOps repo의 deployment.yaml에 `:4` 반영 확인 → ArgoCD ticket-app OutOfSync → Synced → Healthy → Pod rollout 완료
- 향후 GitHub Webhook 연동 시 push 자동 트리거 가능

## ArgoCD 워크로드 (GitOps Apps)

`~/kosa-gitops` 레포의 App-of-Apps 패턴:

- **root-app** → `apps/_applications/` 디렉토리 watch → 아래 Application들 자동 생성
- **Auto sync + selfHeal** 활성화

### 현재 등록된 Application

| Name | Source | 주요 설정 |
|---|---|---|
| `cert-manager` | jetstack helm | TLS cert 자동 발급 |
| `cert-manager-issuer` | apps/ path | `kosa-ca-issuer` ClusterIssuer |
| `monitoring` | prometheus-community helm (kube-prometheus-stack 85.0.2) | sys1 배치, grafana `deploymentStrategy: Recreate` |
| `redis` | bitnami helm | Sentinel HA (3 nodes) |
| `harbor` | helm.goharbor.io 1.16.0 | Ceph RGW S3 백엔드 |
| `jenkins` | charts.jenkins.io | lts-jdk17, Kubernetes 동적 agent, JCasC |
| `ticket-app` | apps/ path | 데모 앱 (Harbor 이미지 사용, Jenkins가 image tag 자동 갱신) |
| `demo-nginx`, `auto-demo` | apps/ path | 데모용 |

### 일반적 ignoreDifferences (Application yaml 공통)

```yaml
ignoreDifferences:
  - group: apps
    kind: StatefulSet
    jsonPointers: [/spec/volumeClaimTemplates]    # immutable
  - kind: Secret
    jsonPointers: [/data]                         # auto-rotated
  - group: admissionregistration.k8s.io
    kind: MutatingWebhookConfiguration
    jsonPointers: [/webhooks]                     # cert-manager가 caBundle 갱신
  - group: admissionregistration.k8s.io
    kind: ValidatingWebhookConfiguration
    jsonPointers: [/webhooks]
```

> **왜 ignore가 필요한가?** ArgoCD는 git의 desired state와 cluster의 실제 state를 비교해서 다르면 OutOfSync로 표시.
> 그런데 K8s API가 자동으로 채우거나 다른 컨트롤러가 변경하는 필드는 git에 없음 → 영원히 OutOfSync.
> - `StatefulSet volumeClaimTemplates`: K8s 자체가 immutable이라 helm upgrade로 변경 불가. 무시해야 sync 가능.
> - `Secret data`: bitnami 등 차트가 random password를 첫 install에 생성 → 이후 git에는 없음.
> - `MutatingWebhookConfiguration webhooks`: cert-manager가 cert 갱신 시 `caBundle` 필드를 새 cert로 자동 patch.

## AWS 하이브리드 (Phase 1 + Phase 2 완료)

상세 narrative + 단계별 절차: `docs/onprem/14-aws-hybrid.md`

### 토폴로지 한눈에

```
온프레 172.16.0.0/12 ──── pfSense ──── ER605 ──── 인터넷 ──── AWS VGW ──── VPC 10.20.0.0/16
                          (IPsec, NAT-T)
                          (Outbound NAT bypass: 172.16.0.0/12→10.20.0.0/16=NO NAT)
```

### 핵심 자원 (lookup)

| 자원 | ID / 값 |
|---|---|
| AWS Region | ap-northeast-2 (서울) |
| VPC | `vpc-03859601c1dd5b658` (10.20.0.0/16) |
| Public Subnet 2a / 2c | 10.20.1.0/24 / 10.20.2.0/24 |
| Private Subnet 2a / 2c | 10.20.10.0/24 / 10.20.20.0/24 |
| NAT GW 2a / 2c | `nat-06228d0a2634bda13` / `nat-0639891e22679e62b` |
| EC2 (HAProxy 2a) | `i-05ad4f4a40f2c7103` (kosa-tickets-haproxy-1a) |
| EC2 (HAProxy 2c) | 10.20.10.121 |
| NLB | `kosa-tickets-nlb-091d28bb8f4ca020.elb.ap-northeast-2.amazonaws.com` |
| CGW (Customer Gateway) | `cgw-0923e106392116cfc` (TP-Link 공인 IP: 125.131.208.229) |
| VGW (Virtual Private GW) | `vgw-0f14a420ce5d30261` |
| VPN Connection | `vpn-0906e8a06bb85a041` |
| VPN Tunnel 1 (AWS 공인) | 43.200.200.229 |
| VPN Tunnel 2 (AWS 공인) | 54.116.133.94 |

### 검증 (양방향 ping)

```bash
# bastion (172.16.24.10)에서
ping -c 5 10.20.10.121
# rtt ~6ms 정상

# EC2 (10.20.10.121)에서 (SSM 접속)
ping -c 5 172.16.24.10
# rtt ~6ms 정상
```

### pfSense IPsec 핵심 설정 (재구축 시 cheat sheet)

```
Key Exchange: IKEv1
Remote Gateway: AWS Tunnel 공인 IP (2개)
NAT Traversal: Force ⭐ (NAT 뒤이므로 필수)
Authentication: Mutual PSK
P1: AES256 / SHA1 / DH2 / 28800s
P2: AES256 / SHA1 / PFS2 / 3600s / Local 172.16.0.0/12 / Remote 10.20.0.0/16
```

### ⭐ Outbound NAT Bypass (이게 안 되어 있으면 ping 실패)

```
Firewall → NAT → Outbound → Hybrid Mode
  맨 위 룰:
    Interface: WAN
    Source: 172.16.0.0/12
    Destination: 10.20.0.0/16
    Translation: NO NAT
```

### AWS 측 빠진 설정 자주 까먹는 것

1. **Route Propagation** (Route Table → 해당 RT → Route propagation 탭 → VGW 체크)
2. **EC2 Security Group**에 ICMP from 172.16.0.0/12 허용
3. **VPN Connection의 Static routes**에 `172.16.0.0/12` 등록

### 비용 (월 예상)

VPN $36 + NAT GW × 2 $64 + EC2 t3.micro × 2 $15 + NLB $16 + 데이터 ~$1 = **약 $130 (17만원)**

## 운영 노하우 (장애 회복 메모)

- **노드 unreachable / no route to host**: K8s API VIP (172.16.23.5) 다운 가능성. lb-1/lb-2 keepalived 상태 확인 (`ip addr | grep 172.16.23.5`).
- **Unknown 상태 pod 정리** (노드 재부팅 후):
  ```bash
  kubectl get pods -A -o wide | awk '$4=="Unknown" {print $1, $2}' | \
    while read ns name; do kubectl delete pod -n $ns $name --grace-period=0 --force; done
  ```
- **PVC Multi-Attach 에러**: 노드 마이그레이션 시 RWO 볼륨이 이전 노드에 묶임. VolumeAttachment 삭제:
  ```bash
  kubectl get volumeattachment | grep <pvc>
  kubectl patch volumeattachment <VA> -p '{"metadata":{"finalizers":null}}' --type=merge
  kubectl delete volumeattachment <VA> --grace-period=0 --force
  ```
- **CSINode 등록 깨짐 (`driver name rbd.csi.ceph.com not found`)**: csi-rbdplugin pod 강제 재시작:
  ```bash
  kubectl delete pod -n ceph-csi-rbd -l app=ceph-csi-rbd,component=nodeplugin \
    --field-selector spec.nodeName=<node>
  ```
- **RBD watcher가 stale로 stuck**:
  ```bash
  kubectl exec -n ceph-csi-rbd $POD -c csi-rbdplugin -- \
    rbd -m <MON_IP> --id <USER> --key <KEY> status <pool>/<vol>
  # stale watcher IP를 blocklist
  kubectl exec ... -- ceph osd blocklist add <watcher_ip>
  ```
- **Harbor registry `s3aws: connection refused`** (API VIP 회복 직후): registry pod의 stale connection 캐시. 재시작으로 해결:
  ```bash
  kubectl delete pod -n harbor -l component=registry
  ```
- **Worker DNS NXDOMAIN `*.kosa.team2`**: pfSense Host Overrides 등록 + 워커 netplan `nameservers.addresses: [172.16.24.2, 1.1.1.1]` + `search: [kosa.team2]` → `netplan apply`
- **CoreDNS도 NXDOMAIN** (cluster pod에서): CoreDNS Corefile에 `hosts` 플러그인 추가 (cluster-scope override):
  ```
  hosts {
      172.16.23.50 harbor.kosa.team2 jenkins.kosa.team2 ticket.kosa.team2 grafana.kosa.team2 argocd.kosa.team2
      fallthrough
  }
  ```
  편집: `kubectl -n kube-system edit configmap coredns` → `kubectl rollout restart deploy coredns -n kube-system`
- **Edge HAProxy 404 (새 도메인)**: lb-1/lb-2 둘 다 `/etc/haproxy/haproxy.cfg`에 `acl <name> hdr(host) -i <fqdn>` 추가 + `use_backend k8s-ingress if ...`에 OR로 연결 → `systemctl reload haproxy`
- **AWS CLI `aws-global location constraint not valid`** (Ceph RGW 상대): `--region us-east-1` 명시 (RGW는 default region 사용)
- **bastion SSH `Permission denied (publickey)`** (모든 노드): 키 이름이 default가 아니면 `~/.ssh/config`에 `IdentityFile` 지정. 키 자체가 없으면 `ssh-keygen` 후 `ssh-copy-id` 또는 Proxmox 콘솔로 `authorized_keys` 직접 등록. 상세는 `docs/onprem/13-validation.md` §1.2.1
- **bastion `Host key verification failed`** (VM 재생성 후): `ssh-keygen -R <ip>` 로 옛 키 제거
- **pip `externally-managed-environment`** (Ubuntu 23.04+ / Debian 12+): PEP 668 lock. `sudo apt install <pkg>` 또는 `pipx install <pkg>` 또는 (비추) `pip install --break-system-packages`
- **외부 노트북에서 `https://*.kosa.team2` timeout** (ping은 OK인데 TCP timeout): pfSense Port Forward "Destination: WAN address"가 노드 개별 WAN IP만 catch함. CARP WAN VIP(192.168.21.109) 가는 트래픽은 무시됨. 해결: Port Forward → Destination을 "WAN_VIP" 명시 + (필요 시) System → Advanced → Firewall & NAT → "NAT Reflection mode" = "Pure NAT" 활성. 노트 노드 WAN: Primary `.110`, VIP `.109` (`pfsense-01` 콘솔에서 확인).
- **bitnami chart specific image tag `not found`** (2025년 이후): `bitnami/redis:8.6.3` 같은 specific tag가 docker hub에서 제거됨, `:latest`만 동작. helm values에 `image.tag: "latest"` 명시 (sentinel/metrics 각각도). 장기적으론 Harbor에 mirror 권장. 상세: `docs/onprem/13-validation.md` §4 "Bitnami specific image tag 함정"
- **StatefulSet rolling 진행 중 Pod 일부만 Running** (예: `kosa-redis-node-0 Terminating`, 1/2 Running): `OrderedReady` 정책으로 역순 1개씩 재생성. 1~2분 기다리면 모두 새 spec으로. 멈춰있으면 `kubectl rollout status sts/<name>`로 진행 확인.
- **`argocd app sync` Unauthenticated / token expired**: ArgoCD CLI 토큰 만료. `argocd login argocd.kosa.team2 --username admin --password "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)" --insecure --grpc-web`. 또는 secret 없어졌으면 admin.password bcrypt 해시 patch + `argocd-server` 재시작. 또는 K8s API 직접: `kubectl patch app <name> -n argocd -p '{"operation":{"sync":{"revision":"HEAD"}}}' --type merge`
- **Edge HAProxy VIP timeout인데 개별 IP는 OK** (split-brain 의심): 양쪽 노드 다 MASTER → ARP 충돌 → TCP 비대칭 → timeout. 진단: `ssh ubuntu@172.16.22.{10,11} "ip addr show eth0 | grep 172.16.22.5"` 양쪽 다 보이면 split-brain 확정. 복구: `lb-2 keepalived stop` → `lb-1 restart` → `lb-2 start` (순서 중요). 영구 예방: `net.ipv4.ip_nonlocal_bind=1` sysctl + keepalived check_script을 단순 `pkill -0` 대신 `curl -ksf https://localhost/healthz`로 강화.
- **MetalLB가 IP 할당했지만 어디서도 ARP 응답 X** (`kubectl get svc`엔 EXTERNAL-IP 있는데 노드엔 없음, curl timeout): speaker가 L2 announce 멈춤. 진단: `kubectl get pods -n metallb-system` 의 speaker RESTARTS 잦으면 memberlist 통신 불안정. 복구: `kubectl rollout restart ds -n metallb-system metallb-speaker`. ⚠️ MetalLB L2 모드는 IP를 NIC에 secondary로 add 안 함 (ARP만 응답) — `ip addr | grep 172.16.23.50`로 진단 X. 대신 `arping -I eth0 172.16.23.50` 또는 단순히 `curl -kI -H "Host: ..." https://172.16.23.50/`.
- **MetalLB 다운 → Edge HAProxy backend 죽음 → VIP까지 timeout cascade**: K8s Ingress가 ARP 응답 멈추면 Edge HAProxy의 backend(172.16.23.50)도 unreachable. Edge HAProxy는 backend down으로 표시하고 connection state cache stale. `systemctl reload haproxy`로 안 풀리고 `systemctl restart haproxy` 필요. 복구 순서: ① MetalLB speaker restart → ② Edge HAProxy 양쪽 restart (reload 아님) → ③ keepalived 양쪽 stop→start 순환.

### AWS VPN / 하이브리드 트러블슈팅 (발생 순서대로)

- **EC2 SSM Session Manager 접속 불가**: ① IAM Role `AmazonSSMManagedInstanceCore` 누락 또는 ② Public Subnet에 두면서 Public IP 안 줌 → SSM agent가 인터넷 outbound 못함. 해결: Private Subnet + IAM Role + Private RT에 `0.0.0.0/0 → NAT GW` 라우트 확인. (상세: `docs/onprem/14-aws-hybrid.md` §5.1)

- **NLB Target unhealthy**: EC2 Security Group이 NLB(또는 VPC CIDR `10.20.0.0/16`)에서 오는 healthcheck를 막음. SG inbound에 healthcheck port 허용 추가.

- **VPN 터널 한쪽만 UP** (AWS Console에서 Tunnel 1=UP, Tunnel 2=DOWN): pfSense에 P1/P2를 한 터널만 만듦. AWS는 항상 2개 터널 제공 → pfSense에 두 번째 P1/P2 추가 + Status → IPsec → "Connect P1 and P2s" 클릭. ⚠️ pfSense는 NAT 뒤라 initiator 역할 — AWS가 먼저 connect 안 함, 반드시 pfSense에서 trigger.

- **VPN 터널 양쪽 다 UP인데 ping 안 됨** (가장 까다로운 케이스): 진단 순서로 4가지 확인 — ① AWS Route Propagation (Route Table → 해당 RT → Route propagation 탭 → VGW 활성화), ② **pfSense Outbound NAT bypass 룰** (Firewall → NAT → Outbound → Hybrid mode → 맨 위에 `172.16.0.0/12 → 10.20.0.0/16 = NO NAT` 추가), ③ EC2 Security Group에 ICMP from `172.16.0.0/12` 허용, ④ pfSense state table 비우기 (`pfctl -F state` 또는 Diagnostics → States → Reset). 90%는 ② Outbound NAT가 원인. (상세: `docs/onprem/14-aws-hybrid.md` §5.4)

- **"설정 다 고쳤는데 안 되다가 1~3분 후 갑자기 됨"**: 캐시/상태 만료 대기. pfSense state table은 ICMP 60s TTL, AWS Route Propagation 반영은 1~3분, IPsec SA rekey는 5~10분. 즉시 강제 해결: pfSense Diagnostics → States → Reset states + Status → IPsec → Disconnect → Connect (SA 재협상). (상세: `docs/onprem/14-aws-hybrid.md` §5.5)

- **ER605 라우터 뒤 IPsec NAT-T 실패**: pfSense IPsec 로그에 "no NAT-T support" 또는 Phase 1 timeout. ER605 → Advanced → ALG → **IPsec ALG disable** (라우터 ALG가 NAT-T 패킷을 잘못 mangle). 일부 라우터는 UDP 500/4500 명시 포워딩도 필요.

- **pfSense IPsec 로그에 "no proposal chosen"** (Phase 1 negotiation 실패): 양쪽 encryption/hash/DH group 미스매치. AWS Configuration 다운로드 파일 기준으로 pfSense P1을 정확히 일치 (보통 AES256/SHA1/DH2 또는 AES256/SHA256/DH14).

- **AWS Console에서 VPN UP인데 pfSense Status → IPsec은 Disconnected**: pfSense가 NAT 뒤일 때 자동 connect가 안 됨. Status → IPsec → 각 터널 옆 "Connect P1 and P2s" 수동 클릭. 또는 P1 설정에서 "Child SA Start Action: None" → "Initiate"로 변경.

- **CGW IP를 pfSense WAN IP로 잘못 등록**: AWS CGW의 IP는 **NAT-T를 통과한 최외곽 공인 IP** (예: TP-Link ER605의 ISP 공인 IP, `125.131.208.229`). pfSense WAN IP(192.168.21.110, 사설)를 등록하면 절대 매칭 안 됨. 확인: `curl ifconfig.me`로 외부에서 보는 본인 IP.

---

## FAQ — 자주 묻는 질문

### Q1. API VIP(172.16.23.5)가 죽으면 어떤 일이 일어나?
**바로**: `kubectl` 명령이 멈춤(timeout), 워커 노드의 kubelet → API 통신 실패 → 노드 NotReady로 표시되기 시작 (eviction은 기본 5분 후), 신규 Pod scheduling 불가.
**오래 지속되면**: HPA scaling 멈춤, ArgoCD reconcile 멈춤, Webhook(cert-manager 등) 호출 실패. 단, **이미 running 중인 Pod의 트래픽은 영향 없음** (kube-proxy iptables 규칙은 노드 로컬). Pod ↔ Pod 통신, Pod ↔ Service 통신 모두 그대로 동작.
**확인**: lb-1/lb-2에서 `ip addr | grep 172.16.23.5`로 VIP 위치 확인 → `systemctl status keepalived` → `journalctl -u keepalived -n 50`.

### Q2. Ceph가 HEALTH_WARN인데 어디부터 봐?
1. `ceph -s` (전체 상태)
2. `ceph health detail` (어떤 WARN인지)
3. 자주 보는 케이스:
   - `OSDs down`: `ceph osd tree` → 어느 노드/OSD?
   - `slow ops`: 디스크 SMART 검사 (`smartctl -a /dev/sdX`)
   - `mons clock skew`: 노드 NTP 동기화 (`chronyc tracking`)
   - `pgs degraded/undersized`: replica 부족, `ceph pg dump_stuck`

### Q3. 새 마이크로서비스를 sys1과 production 중 어디에 둘지 어떻게 결정?
**system (sys1)에 두는 기준**:
- 클러스터 자체 운영용 (모니터링/CI/CD/cert/registry)
- 비즈니스 트래픽 폭증 시에도 살아있어야 진단 가능한 것
- RWO PVC 사용 + 다른 노드 마이그레이션 시 문제 발생

**production (w1~w3)에 두는 기준**:
- 사용자 트래픽이 흐르는 비즈니스 워크로드
- HA replica로 분산 가능 (Deployment, ReplicaSet)
- 부하 변동에 따라 HPA scaling 필요

**애매하면 production**. sys1은 single node라 부하 견딜 한계 있음.

### Q4. Harbor 인증서는 어떻게 갱신?
Harbor cert는 cert-manager가 ClusterIssuer `kosa-ca-issuer`로 자동 발급/갱신. 90일 전쯤 자동 renew (cert-manager 기본).
**수동 강제 갱신**: `kubectl delete certificate -n harbor harbor-ingress-cert` → cert-manager가 재발급 → 새 Secret 생성.
**자체 CA (`~/pki/ca.crt`) 자체가 만료**: 10년이라 신경 안 써도 됨. 만료 1년 전쯤 새 CA 발급 + 모든 노드 trust store 교체 + cert-manager Secret 교체 필요.

### Q5. ArgoCD가 OutOfSync인데 Healthy면 무슨 의미? 뭘 확인?
- **Healthy**: 워크로드가 정상 동작 중 (Pod Running, Service Endpoint 있음)
- **OutOfSync**: git의 manifest와 cluster의 실제 spec이 다름

대부분의 경우 정상이고, 다음 케이스:
1. **방금 git commit/push**: 곧 sync됨 (selfHeal 켜져 있으면 자동, 아니면 수동 sync)
2. **manual로 cluster 변경**: `kubectl edit/patch`한 것을 git이 모름 → selfHeal이 되돌림
3. **자동 생성 필드 누락**: 위 `ignoreDifferences` 섹션 참고

확인: ArgoCD UI에서 해당 App 클릭 → "App Diff" → 어떤 필드가 다른지 확인.

### Q6. 지금 아키텍처에서 SPoF는?
1. **RGW (ceph1 단일 데몬)**: Harbor 이미지 storage. ceph1 노드/RGW down → Harbor push/pull 불가 (이미 running Pod는 영향 X). 해결: RGW를 2개 이상 노드에 배포.
2. **sys1 노드**: ArgoCD/Harbor/Jenkins/모니터링 전부 여기. sys1 down → CI/CD/관측 전부 정지. 해결: sys2 추가 + HA 가능한 컴포넌트(ArgoCD, Prometheus)는 replica.
3. **NAT Gateway (AWS)** (활성화 시): private subnet의 인터넷 outbound가 AZ1 NAT 하나에 묶임. 해결: AZ별 NAT (비용 ↑).
4. **pfSense MASTER 노드의 Proxmox**: VM으로 올라간 pfSense → 호스트 down 시 BACKUP으로 failover. 정상 동작 중이지만 부팅 순서 의존성 주의.

### Q7. 온프레-AWS burst는 언제 트리거? 현재 구현 상태?
**설계**: 온프레 K8s가 부하 임계(CPU >80% 5분 등) 도달 → CloudWatch + Lambda → EKS Karpenter scale-out → 동일 ArgoCD가 EKS에도 배포 → 트래픽 일부를 AWS로.
**현재 상태**: AWS Terraform은 VPC/NLB/HAProxy 인스턴스까지 완성. **VPN, EKS, RDS replica, Lambda trigger, multi-cluster ArgoCD는 미구현** — 다음 phase.

### Q8. 데모 앱(kosa-tickets)에 새 기능 배포하려면 끝까지 어떻게 흘러가?
1. `git@github.com:kosacloudteam2/kosa-tickets.git`에 코드 push
2. Jenkins UI에서 `kosa-tickets-ci` job → "Build Now" (수동, 향후 webhook)
3. Jenkins Pod 안에서 Kaniko가 이미지 빌드 → Harbor에 `harbor.kosa.team2/library/kosa-tickets:<BUILD_NUMBER>` push
4. Jenkins가 `kosa-gitops` repo의 `apps/ticket-app/deployment.yaml` image tag를 `<BUILD_NUMBER>`로 sed → commit/push
5. ArgoCD가 git 변경 감지 (3분 polling) → ticket-app Application sync
6. K8s Deployment rolling update → 새 Pod 뜸 → Service Endpoint 갱신
7. HAProxy Ingress → 새 Pod로 트래픽

### Q9. cert-manager가 발급한 cert와 자체 CA wildcard cert는 각각 어디서 쓰여?
- **자체 CA wildcard `*.kosa.team2`** (`~/pki/wildcard.pem`): **Edge HAProxy** (lb-1/lb-2)에서 1차 TLS 종료. 모든 `*.kosa.team2` 도메인 커버. 1년 만료, 수동 회전.
- **cert-manager 자동 발급 cert** (per-service): HAProxy Ingress가 2차 TLS 종료할 때 service별로 cert 발급. 같은 자체 CA로 서명 (`kosa-ca-issuer`). 90일 자동 회전.
- 둘 다 같은 root CA로 서명되어 있으므로 client 입장에서는 동일하게 신뢰.

### Q10. 노트북에서 `https://harbor.kosa.team2`가 안 열려요
체크리스트:
1. pfSense를 DNS로 사용 중인가? `nslookup harbor.kosa.team2` → 172.16.23.50 나와야 함
   - 안 나오면 pfSense Host Override 확인 또는 노트북 DNS 설정
2. 노트북이 자체 CA (`~/pki/ca.crt`)를 trust하나? 브라우저 cert 경고 → CA를 system/브라우저에 import
3. Edge HAProxy ACL에 추가됐나? lb-1/lb-2에서 `grep harbor /etc/haproxy/haproxy.cfg`
4. K8s Ingress가 살아있나? `kubectl get ingress -A | grep harbor`
5. Harbor Pod 상태? `kubectl get pods -n harbor`

### Q11. 새 도메인 (예: `wiki.kosa.team2`) 추가하려면?
1. **pfSense**: Services → DNS Resolver → Host Overrides에 `wiki` / `kosa.team2` / `172.16.23.50` 추가 → Apply
2. **Edge HAProxy** (lb-1, lb-2 모두): `/etc/haproxy/haproxy.cfg`에 `acl wiki hdr(host) -i wiki.kosa.team2` 추가 + `use_backend k8s-ingress if ...` 줄에 OR 추가 → `systemctl reload haproxy`
3. **K8s**: 해당 앱의 Ingress에 `host: wiki.kosa.team2` 추가 + cert-manager annotation
4. **(필요시) CoreDNS**: `kubectl -n kube-system edit cm coredns` → hosts 블록에 추가
5. 검증: 브라우저에서 접속

### Q12. Proxmox 노드 1대가 죽으면?
- 그 노드에 있던 VM은 down. HA 활성화돼 있으면 다른 노드에서 자동 시작 (단 우리는 Ceph 백엔드가 아니라 local disk라 자동 마이그레이션 X — 정확히는 일부 VM만).
- **pfSense VM이 있던 노드면**: 다른 노드의 pfSense가 CARP MASTER로 승격 (HA 동작).
- **K8s 노드가 있던 노드면**: 해당 노드 down → 5분 후 Pod eviction → 다른 워커로 reschedule.
- **bastion이 있던 노드면 (kosa3)**: SSH/kubectl 접근 불가. → 다른 노트북에서 kubeconfig 가지고 직접 K8s API VIP로 접근 가능.
- **lb-1/lb-2 중 하나가 있던 노드면**: Keepalived BACKUP이 MASTER로 승격 (수 초 내).

### Q13. Ceph 디스크가 가득 차면?
- 85% (`mon_osd_nearfull_ratio`): WARN
- 95% (`mon_osd_full_ratio`): 모든 쓰기 차단 → 클러스터 read-only
- 대응: OSD 추가, 데이터 정리, EC 풀로 전환, 임시 ratio 상향 (`ceph osd set-full-ratio 0.97`)

### Q14. RBD PVC가 "Multi-Attach" 에러로 Pod이 안 뜨면?
RWO 볼륨이 이전 노드에 attach된 상태로 남아있음. 해결:
```bash
kubectl get volumeattachment | grep <pvc-name>
kubectl patch volumeattachment <VA> -p '{"metadata":{"finalizers":null}}' --type=merge
kubectl delete volumeattachment <VA> --grace-period=0 --force
# 그래도 안 되면 stale watcher blocklist
kubectl exec -n ceph-csi-rbd $POD -c csi-rbdplugin -- ceph osd blocklist add <prev-node-ip>
```

### Q15. ArgoCD Application 추가하려면 어디에 commit?
- `~/kosa-gitops/apps/_applications/<name>.yaml` (Application 정의)
- root-app이 자동으로 감지해서 새 Application 생성
- Helm chart는 `source.repoURL` + `source.chart` + `source.helm.values` 형태
- raw manifest는 `source.repoURL: <kosa-gitops>` + `source.path: apps/<name>`

### Q16. Jenkins 빌드가 자꾸 실패해. 어디부터 봐?
1. Pipeline log: 어느 stage에서?
2. **Checkout 단계**: SSH key/GitHub 접근 권한 → `kosa-gitops-ssh` credential 확인
3. **Build 단계 (Kaniko)**: Harbor 인증 → `harbor-creds-dockerconfigjson` Secret 확인, `--skip-tls-verify`
4. **Push 단계**: Harbor 상태, bucket 존재, registry pod 정상
5. **GitOps update 단계**: `kosa-gitops` write 권한, conflict, branch protection
6. K8s 측: Jenkins agent Pod이 뜨는가? `kubectl get pods -n jenkins`

### Q17. Pod-to-Pod 트래픽은 1G인가 10G인가?
**실측 (2026-05-18): 5.34 Gbits/sec** — Calico가 10G NIC(`enp1s0f0`) 데이터 평면 사용 중. 1G의 5배 → 의도대로 10G fabric 통과.
이상 9.4G 대비 5G로 떨어지는 이유: IPIP encapsulation 오버헤드 + Calico tunl0 단일 스레드 + Proxmox virtualization 오버헤드 + MTU 1500.
**우리 워크로드(Redis Sentinel sync 50MB/s, Prometheus federation 10MB/s, 일반 HTTP API < 100MB/s) 모두 충분.** 추가 짜내기 옵션:
- Jumbo frame (MTU 9000) — 양쪽 일치 시 7~8G 가능, 미스매치면 단절
- IPIP → BGP — 외부 BGP 라우터 필요 (우리 환경엔 없음)
- eBPF dataplane — Calico Operator 재구성 필요
상세: `docs/onprem/13-validation.md` §2.3.1

### Q18. Ceph 성능이 느린 거 같은데 네트워크 문제?
**아니, 네트워크는 멀쩡 — HDD가 병목.**

실측 (2026-05-18, rbd-team2 풀):
- RBD 4K randwrite: 1,700 IOPS (cache 포함), `--rbd-cache=false` 시 ~100~200
- RBD 1M seqwrite: **35 MB/s** ← 이게 진짜 한계치
- RADOS 4K randwrite: 99 IOPS (cache 우회 정직 수치)
- 같은 NIC iperf3: 9.4 Gbps = 1,175 MB/s (네트워크 사용률 3%)

계층별 한계:
```
10G NIC          → 1,250 MB/s
6 HDD 합          → 600 MB/s (HDD 100 MB/s × 6)
3-replica        → 200 MB/s (각 write가 3 디스크에)
WAL/DB 같은 HDD  → 70~100 MB/s (seek thrashing)
실측             → 35 MB/s
```

**개선 1순위 (가성비)**: SSD WAL/DB 분리. 노드당 100GB SSD ~5만원 × 6 = 30만원. seq write 4~8배, randwrite 5~15배.
**진단 명령**: 13장 §3 + §9.2 참고.

### Q19. Ceph pool 이름이 문서와 다름
문서 다수에 `team2-rbd-block`이라 적혀있지만 실제 클러스터엔:
- `team2-k8s-pvc-rbd` — K8s CSI용 (이게 진짜)
- `rbd-team1~4` — 4팀 분리용
- `cephfs_metadata`, `cephfs_data` — CephFS도 깔려있음 (문서엔 미사용이라 했으나)
- `default.rgw.*` — RGW 백엔드

확인: `ceph osd pool ls` (ceph 노드) + `kubectl get sc -o yaml | grep pool` (K8s).
docs (04, 08, 09, 11, 13) 정정 필요 — 다음 검증 사이클에 일괄 처리.

### Q20. Redis Sentinel HA가 진짜 동작하나?
**검증 (2026-05-18)**: 3 노드 + quorum 2 구성으로 1대 죽어도 자동 failover 보장.

```bash
# 현재 master 확인
kubectl exec -n redis kosa-redis-node-0 -c sentinel -- \
  redis-cli -p 26379 -a kosa1004 sentinel get-master-addr-by-name mymaster

# Quorum 체크
kubectl exec -n redis kosa-redis-node-0 -c sentinel -- \
  redis-cli -p 26379 -a kosa1004 sentinel ckquorum mymaster
# 응답: "OK 3 usable Sentinels"

# Failover 시험 — master 죽이기
MASTER_POD=kosa-redis-node-0  # 또는 위 명령 결과의 master
kubectl delete pod -n redis $MASTER_POD
sleep 30
# 새 master 확인 — 다른 노드로 바뀌어야
```

**주의**: 이전 2 노드 + quorum 2 구성은 **SPoF** (1대 죽으면 quorum 깨짐). 3 노드가 진정한 HA 최소 구성.

**Redis 실사용 확인 (2026-05-18)**: ticket-app에 대기열 endpoint (`/api/queue/enter`, `/status`, `/next`, `/health`) 추가. LPUSH/RPOP/LLEN으로 FIFO 대기열 동작. 검증: `db0:keys=1`, `keyspace_hits` 증가, `total_commands` 2.8K → 31K. 상세: `docs/onprem/13-validation.md` §4.6

### Q21. AWS와 온프레가 VPN으로 연결됐다는 게 정확히 무슨 뜻?
온프레 사설망(172.16.0.0/12)과 AWS VPC(10.20.0.0/16)가 **마치 같은 LAN인 것처럼** 통신 가능. 인터넷을 통하지만 IPsec으로 암호화되어 있어 안전.
**할 수 있는 것**: bastion에서 EC2(10.20.10.121)로 ssh, ping. K8s Pod에서 AWS RDS endpoint로 DB 쿼리. AWS Lambda가 온프레 API 호출 (단, 보안 그룹/방화벽 허용 시).
**못 하는 것**: VPN은 트래픽 통로만 제공. 라우팅/방화벽/DNS는 별도 설정 필요. 예: AWS EC2가 `harbor.kosa.team2` 도메인을 알아야 하면 별도 DNS 설정 또는 `/etc/hosts` 매핑 필요.

### Q22. VPN이 끊기면 어떤 영향?
**평소**: 영향 없음 (현재 Phase 3+ 미구축이라 실제 운영 트래픽 없음).
**Phase 3 이후** (RDS Replica 연결 시): 온프레 PXC → RDS replication 끊김. RDS는 stale 데이터. 복구 후 자동 catch-up.
**Phase 4 이후** (EKS Burst 활성 시): EKS Pod이 온프레 PXC에 read 불가 → fail. 트래픽 처리 못 함. 복구 필요.
**대응**: VPN 자체는 양쪽 터널 (43.200.200.229, 54.116.133.94) 중 하나만 살아있어도 동작. 둘 다 죽으면 ER605/pfSense/AWS 중 하나 장애.

### Q23. AWS Free Tier로 얼마나 가능?
- **VPN Connection**: Free tier 없음 ($36/월)
- **NAT Gateway**: Free tier 없음 ($32/월 × 2 = $64)
- **EC2 t3.micro**: 750h 무료 (1대만 24시간), 2대 운영 시 한대 분만 무료
- **NLB**: Free tier 없음 (~$16/월)

→ 우리 구성(VPN + NAT GW 2개 + EC2 2대 + NLB) 실비 ~$130/월 (17만원). 데모 후 stop/delete 권장. **NAT GW는 stop 안 됨, 삭제만 가능** → 안 쓸 때는 NAT GW만이라도 삭제하면 60% 비용 절감.

### Q24. VPN 위에 뭘 얹어야 진짜 "하이브리드"인가?
VPN은 그저 통로. 진짜 하이브리드는 **데이터/워크로드의 양방향 흐름**:
1. **데이터 복제** (Phase 3): 온프레 PXC → AWS RDS Read Replica. 읽기 부하 분산.
2. **컴퓨트 burst** (Phase 4): EKS Karpenter로 트래픽 폭증 시 노드 자동 생성, 평소 0.
3. **자동 trigger** (Phase 4): CloudWatch (온프레 메트릭은 어떻게? Prometheus remote_write to AMP) + Lambda → EKS scale-out.
4. **GitOps 통합** (Phase 4): ArgoCD multi-cluster 등록 → 온프레 + EKS 동시 배포.
5. **트래픽 분산** (Phase 5): Route 53 weighted routing 또는 NLB cross-region.


