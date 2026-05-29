# Minecraft on Kubernetes (Bedrock + Java)

베어메탈 단일/소규모 클러스터에서 마인크래프트 Bedrock + Java 에디션을
실서버급으로 운영하기 위한 GitOps 친화 매니페스트 번들.

```
minecraft-k8s/
├── kustomization.yaml          # ★ 이미지 버전을 한 곳에서 핀(pin)하는 버저닝 핵심
├── namespace.yaml
├── bedrock.yaml                # Bedrock: ConfigMap+PVC+Deploy(server+backup)+NodePort(UDP)
├── java.yaml                   # Java(Paper): +RCON 기반 백업
├── monitoring.yaml             # mc-monitor exporter + ServiceMonitor + 알림 규칙
├── logging.yaml                # Grafana Alloy -> Loki (minecraft ns 로그)
└── dashboards/
    └── minecraft-logs-dashboard.json   # Grafana 임포트용 로그 대시보드
```

## 사전 요구

- `local-path-provisioner` (StorageClass `local-path`) — 라이브 월드 PVC용
- `kube-prometheus-stack` — Prometheus Operator + Grafana (ServiceMonitor/PrometheusRule CRD)
- `loki` (예: grafana/loki 헬름) — 로그 저장. `loki.monitoring.svc:3100` 가정
- Garage 등 S3 호환 스토어 — 백업 타겟 (MinIO 커뮤니티는 2026년 사실상 EOL, Garage 권장)

## 버저닝 전략 (versioning)

세 층으로 버전을 명시적으로 고정한다. **어디에도 떠다니는 `latest`를 두지 않는 것**이 원칙.

1. **컨테이너 이미지** — `kustomization.yaml`의 `images:` 블록에서만 태그 지정.
   매니페스트 본문은 태그 없이 이미지명만 쓰고, 버전업은 이 블록 한 줄 수정 → commit.
   ```yaml
   images:
     - name: itzg/minecraft-bedrock-server
       newTag: "2026.4.1"
   ```
2. **마인크래프트 서버 버전** — ConfigMap의 `VERSION`.
   - Java: 플러그인 호환 때문에 `1.21.4`처럼 **명시 고정** 권장.
   - Bedrock: `LATEST`도 가능하나, 서버가 클라보다 앞서가면 접속 불가.
     모바일 앱 롤아웃이 느릴 땐 `PREVIOUS`로 한 단계 물려둔다.
3. **Git / 배포 버전** — 이 디렉터리를 Git에 두고 태그(`v1.0.0`)로 릴리스.
   ArgoCD/Flux가 특정 ref를 추적하면, 클러스터 상태 = Git 상태로 재현 가능.
4. **월드 데이터 이력** — 백업이 곧 월드의 버전 히스토리.
   `garage:mc-backups/<edition>/`에 타임스탬프로 쌓이며 `PRUNE_BACKUPS_DAYS`로 보존.

> 롤백: 이미지/VERSION을 이전 값으로 되돌려 commit + 필요한 월드 백업을 PVC로 복원.

## 배포

```bash
# 1) Garage(S3) 자격증명 시크릿 (rclone S3 remote 'garage')
kubectl create ns minecraft
kubectl -n minecraft create secret generic rclone-garage \
  --from-literal=RCLONE_CONFIG_GARAGE_TYPE=s3 \
  --from-literal=RCLONE_CONFIG_GARAGE_PROVIDER=Other \
  --from-literal=RCLONE_CONFIG_GARAGE_ENDPOINT=http://garage.storage.svc.cluster.local:3900 \
  --from-literal=RCLONE_CONFIG_GARAGE_ACCESS_KEY_ID=<KEY> \
  --from-literal=RCLONE_CONFIG_GARAGE_SECRET_ACCESS_KEY=<SECRET> \
  --from-literal=RCLONE_CONFIG_GARAGE_REGION=garage

# 2) 적용
kubectl apply -k .

# 3) Grafana에 dashboards/minecraft-logs-dashboard.json 임포트 (Loki/Prometheus DS 선택)
```

## 네트워킹 (베어메탈, NodePort 방식)

MetalLB 없이 NodePort + 공유기 포트포워딩으로 충분(단일노드 기준).

| 에디션 | 내부 | NodePort | 공유기 포워딩 |
|--------|------|----------|---------------|
| Bedrock | UDP 19132 | 31132 | 외부 19132/udp → 노드IP:31132 |
| Java | TCP 25565 | 30565 | 외부 25565/tcp → 노드IP:30565 |

- 기본 NodePort 범위(30000–32767) 때문에 19132를 직접 못 써서 31132로 받고, 외부에선 공유기가 19132로 노출.
- RCON(25575), mc-monitor(8080), Garage S3(3900)는 **ClusterIP만**. 외부 노출 금지.
- 유동 IP면 DDNS, CGNAT 회선이면 포워딩이 무력화되니 UDP 터널(playit.gg 등) 필요.

## 운영 체크리스트 (우선순위 순)

1. **백업 검증** — 백업이 실제 Garage에 쌓이는지, 복원이 되는지 한 번은 테스트.
2. **그레이스풀 셧다운** — 재배포 시 월드 세이브 완료(터미네이션 120s) 확인.
3. **mc-monitor → Grafana** — 접속 인원/헬스 패널 동작.
4. **알림** — 서버 다운/OOM/백업 지연 Alertmanager 라우팅.
5. **GitOps** — ArgoCD/Flux 연결로 SSOT 확립.

## 알려진 확인 포인트 (이미지 버전 따라 플래그 상이 가능)

- `mc-monitor`의 Bedrock edition 지정 방식(플래그/스킴)은 사용 중인 태그 문서 확인 후 `monitoring.yaml` 조정.
- Bedrock 백업 일관성: Bedrock은 RCON이 없어, 정합 스냅샷이 중요하면
  `PAUSE_IF_NO_PLAYERS=true` 또는 save-hold 연동 동작을 검증할 것.
- Java에서 Bedrock 크로스플레이가 목표면 Paper + GeyserMC + Floodgate(`SPIGET_RESOURCES`)로
  Java 한 서버에 베드락 클라까지 붙일 수 있어 컨테이너를 하나로 합치는 선택지도 있음.
