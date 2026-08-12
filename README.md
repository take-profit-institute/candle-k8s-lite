# candle-k8s-lite (GitOps, k3s)

`../infrastructure-lite`가 k3s+ArgoCD를 부트스트랩한 뒤, 이 repo의 매니페스트가 나머지를 관리.

## Terraform ↔ candle-k8s-lite 경계

| Terraform (`infrastructure-lite`) | candle-k8s-lite (this repo) |
|---|---|
| VPC + EC2(Spot) + EIP + SG + IAM | ArgoCD Applications (platform + services) |
| Route53 서브존 + 와일드카드 A | Ingress (cert-manager가 host별 TLS 발급) |
| k3s + ArgoCD 부트스트랩 (user_data) | ingress-nginx / cert-manager / external-dns |
| — | postgres / timescaledb / redis (StatefulSet+PVC) |
| — | kafka(KRaft) + kafka-connect(Debezium) |
| — | 13개 마이크로서비스 (chart) |

## 구조

```
candle-k8s-lite/
├── projects/candle-lite.yaml            # ArgoCD AppProject
├── bootstrap/dev.yaml                    # app-of-apps 루트 (Terraform user_data가 apply)
├── platform/
│   ├── applications/                     # 루트가 자동 include (recurse:false)
│   │   ├── ingress-nginx.yaml            # Helm
│   │   ├── cert-manager.yaml             # Helm (Let's Encrypt)
│   │   ├── external-dns.yaml             # Helm (Route53)
│   │   ├── data-stack.yaml               # → base/data
│   │   ├── kafka.yaml                    # → base/kafka
│   │   ├── platform-config-dev.yaml      # → overlays/dev
│   │   └── services-dev.yaml             # ApplicationSet 13종
│   └── manifests/
│       ├── base/
│       │   ├── data/
│       │   │   ├── postgres.yaml         # StatefulSet + PVC(local-path)
│       │   │   ├── timescaledb.yaml
│       │   │   ├── redis.yaml            # 단일 인스턴스 (prod 3개 대체)
│       │   │   ├── init-secrets-job.yaml # DB 자격증명 랜덤 생성 → Secret (git에 평문 없음)
│       │   │   ├── init-db-job.yaml      # 단일 candle DB + schema/role + <svc>-db Secret 자동생성
│       │   │   └── init-jwt-key-job.yaml # auth-service RSA 2048 keypair 1회 생성 → auth-jwt-key Secret
│       │   └── kafka/
│       │       ├── kafka.yaml            # KRaft 단일 브로커
│       │       ├── kafka-connect.yaml    # Debezium 커넥터 런타임
│       │       ├── init-secrets-rbac.yaml # init-secrets가 kafka ns에 debezium-creds 복제하도록 허용
│       │       └── connectors-job.yaml   # 커넥터 config를 REST PUT (7종)
│       └── overlays/dev/                 # namespaces, ClusterIssuer, ingresses
└── services/chart/                       # prod chart 카피 + ExternalSecret 제거 + redis.ssl 옵션
```

## prod chart 대비 변경점 (`services/chart/`)

1. `templates/externalsecret.yaml` **삭제** — ESO 없이 인클러스터 Secret 직접 사용
2. `templates/deployment.yaml`:
   - `SPRING_DATA_REDIS_SSL_ENABLED` 값이 `.Values.redis.ssl` (기본 `false`)
   - wishlist `REDIS_URL` scheme이 `.Values.redis.scheme` (기본 `redis`, prod에선 `rediss`)
3. `values.yaml`에 `redis.ssl` / `redis.scheme` 기본값 추가
4. ApplicationSet values에서 `image`(repo override) / `port` / `servicePort` 지원 — gateway처럼 ECR 밖 이미지(Docker Hub) + 비표준 포트(:8000) 쓰는 서비스 대응

## 부트스트랩 흐름

```
Terraform apply
  → EC2 user_data (k3s + ArgoCD 설치)
  → user_data가 이 repo clone → kubectl apply projects/ + bootstrap/dev.yaml
  → ArgoCD가 platform/applications/*.yaml 을 자동 sync
    ├─ ingress-nginx (LoadBalancer via k3s klipper → EIP :80/:443)
    ├─ cert-manager (Let's Encrypt HTTP-01)
    ├─ external-dns (Route53 자동 레코드)
    ├─ data-stack → init-secrets Job이 DB 자격증명 랜덤 생성 → init-db Job이
    │               단일 candle DB + 서비스별 schema/role + <svc>-db Secret 생성
    ├─ kafka-stack → connectors Job이 Debezium 커넥터 등록
    ├─ platform-config-dev (namespaces, ClusterIssuer, argocd/gateway Ingress)
    └─ candle-lite-services (13개 마이크로서비스)
```

## 도메인 레이아웃

와일드카드 `*.lite.candle.io.kr` → EIP (Terraform 소유).
Ingress host별로 cert-manager가 TLS 발급:

| Host | 대상 |
|---|---|
| `argocd.lite.candle.io.kr` | ArgoCD UI |
| `api.lite.candle.io.kr` | gateway (Spring Cloud Gateway, :8000) → auth/BFF |
| `app.lite.candle.io.kr` | bff (SPA) |

새 host 추가는 `platform/manifests/overlays/dev/gateway-ingress.yaml` 확장 → external-dns가 A 레코드 자동 생성.

## Repo URL 치환

모든 매니페스트의 `https://github.com/take-profit-institute/candle-k8s-lite.git` 을 실제 org/repo로 변경:
```bash
# lite 폴더 안에서
grep -rl "take-profit-institute/candle-k8s-lite" | xargs sed -i 's|take-profit-institute/candle-k8s-lite|<YOUR_ORG>/<YOUR_REPO>|g'
```

Terraform variables.tf 의 `gitops_repo_org` / `gitops_repo_name` 도 함께 변경.

## 커넥터 추가 (선택)

`platform/manifests/base/kafka/connectors-job.yaml` 의 ConfigMap `debezium-connectors` 에
- 현재: auth / user / trading 3개
- 추가하려면 같은 JSON 패턴 (ranking, notification, stock, wishlist 등)

수정 후 push → ArgoCD가 재sync → Job은 `Recreate` 정책이 아니라서 재실행 필요 시
```bash
kubectl -n kafka delete job debezium-connectors-apply
kubectl apply -f platform/manifests/base/kafka/connectors-job.yaml
```

## 서비스 이미지 tag 갱신

`platform/applications/services-dev.yaml` 의 `tag: <sha>` 를 CI가 bump.
현재 초기값은 prod `candle-k8s/platform/applications/services-dev.yaml` 시점의 SHA — 이미 새 빌드가 있다면 최신 SHA로 교체 필요.

## 알려진 제약

1. **관측 스택 없음** — Prometheus/Grafana/Loki/Jaeger 제거. 로그는 `kubectl logs`, 메트릭은 없음. 필요하면 이후 `platform/applications/observability.yaml` 추가.
2. **mTLS 없음** — Istio 미설치. 서비스간 통신은 평문(cluster network 격리에만 의존).
3. **시크릿** — git에 평문 비밀 없음. `init-secrets-job`이 postgres superuser / timescaledb(market) / debezium 비번을 클러스터 안에서 랜덤 생성(`openssl rand -hex 16`)해 Secret에 저장하고, 서비스별 role 비번은 `init-db-job`이 `<svc>-db` Secret에 생성·보존한다. 이미 있으면 재사용하므로 Job을 다시 돌려도 살아있는 서비스가 깨지지 않는다.
   - **로테이션**: 해당 Secret 삭제 후 Job 재실행 → `ALTER ROLE`로 DB에 반영. 서비스는 `kubectl -n candle rollout restart deployment` 필요(env로 주입되므로).
   - debezium 비번은 connectors-job이 읽어야 해서 `kafka` 네임스페이스에도 같은 값으로 복제된다.
   - prod 스타일이 필요하면 ESO + AWS Secrets Manager로 교체.
4. **JWT** — RS256 + JWKS로 통일 (`GatewaySecurityConfig` 주석: "대칭키 공유 폐지"). auth-service가 자체 keypair로 서명 + `/.well-known/jwks.json` 노출:
   - **gateway** → `GATEWAY_JWT_JWK_SET_URI=http://auth-service/.well-known/jwks.json`
   - **bff/chatting** → `AUTH_JWKS_URI=http://auth-service/.well-known/jwks.json`
   - 셋 다 `issuer=https://api.lite.candle.io.kr`, `audience=candle-api` 로 일치해야 함
   - **RSA 개인키**: `init-jwt-key-job`이 최초 1회 openssl로 2048 keypair 생성 → `auth-jwt-key` Secret 저장 → auth-service가 마운트 (`AUTH_JWT_PRIVATE_KEY`). 재기동해도 동일 키 유지 → 세션 지속. 로테이션 원하면 Secret 삭제 후 Job 재실행.
5. **콜드 스타트** — 13개 JVM 순차 기동 20~30분. 부팅 순서 강제 없음(k3s CNI 준비 → ArgoCD sync → 이미지 pull → JVM warmup).
6. **CPU throttling** — t3.xlarge 4vCPU에 13개 서비스가 붙으면 CPU가 병목. HPA 꺼놨고 replica=1로 눌러놨음. 부하 테스트는 어려움.
