# ArbiEver Blockscout

ArbiEver L2 (Chain ID `580511`) 전용 [Blockscout](https://github.com/blockscout/blockscout) 풀스택 익스플로러 배포 레포.

5컨테이너 미니멀 구성으로 4코어 12GB 환경에서 안정 가동.

## 접속

- **외부**: https://arbiever2.ever-chain.xyz
- 로컬: http://localhost:4005

## 빠른 시작

```bash
# 1. 시크릿 생성
cat > .env <<EOF
POSTGRES_PASSWORD=$(openssl rand -hex 16)
SECRET_KEY_BASE=$(openssl rand -hex 32)
EOF
chmod 600 .env

# 2. 가동 (재부팅 시 자동 복구 — restart: unless-stopped)
docker compose --env-file .env up -d

# 3. 확인
curl http://localhost:4005/api/v2/stats
```

`proxy` 서비스는 사전 빌드된 **`silverruler/arbiever-blockscout:latest`** (DockerHub) 이미지를 자동 pull. nginx + nginx.conf + arbiicon.png 가 임베디드됨.

## 5개 서비스

| 컨테이너 | 이미지 | 역할 |
|---|---|---|
| `arbiever-bs-redis` | `redis:alpine` | 인덱서 캐시 |
| `arbiever-bs-db-init` | `postgres:17` | DB 디렉토리 권한 초기화 (one-shot) |
| `arbiever-bs-db` | `postgres:17` | 인덱서 DB |
| `arbiever-bs-backend` | `blockscout/blockscout:6.10.1` | Elixir 인덱서 + API |
| `arbiever-bs-frontend` | `ghcr.io/blockscout/frontend:v1.36.4` | Next.js UI |
| `arbiever-bs-proxy` | **`silverruler/arbiever-blockscout:latest`** | nginx — :4005 + favicon 정적 서빙 |

비활성 (메모리 절약): visualizer / stats / sig-provider / user-ops-indexer.

## ArbiEver 브랜드 패치

- 네트워크 이름: **ArbiEver**
- 통화 단위: **ETE**
- 로고 / favicon / 메뉴 아이콘: `arbiicon512.png` → `/static/arbiicon.png`
- 기본 색상 테마: Light
- testnet 배지: 없음

핵심 환경변수 (`frontend.env`):
```
NEXT_PUBLIC_NETWORK_NAME=ArbiEver
NEXT_PUBLIC_NETWORK_CURRENCY_SYMBOL=ETE
NEXT_PUBLIC_NETWORK_LOGO=https://arbiever2.ever-chain.xyz/static/arbiicon.png
NEXT_PUBLIC_IS_TESTNET=false
NEXT_PUBLIC_COLOR_THEME_DEFAULT=light
```

## L2 RPC 연결

```
ETHEREUM_JSONRPC_HTTP_URL=http://host.docker.internal:8449/
```

도커 컨테이너가 `host.docker.internal:host-gateway` 를 통해 호스트의 Nitro RPC (0.0.0.0:8449) 에 접근.

## 도메인 + 인프라

- Cloudflare DNS: `arbiever2.ever-chain.xyz` → OCI VPN 게이트웨이
- OCI stunnel: `accept=4005`, `connect=10.8.0.14:4005`
- 본 서버 (10.8.0.14): `proxy` 컨테이너 :4005 → backend(4000) + frontend(3000)

## 자체 빌드 (선택)

기본 사용은 DockerHub 의 pre-built proxy 이미지면 충분. 직접 빌드하려면:

```bash
docker build -f Dockerfile.proxy -t arbiever-blockscout-proxy:local .
# docker-compose.yml 의 proxy.image 를 arbiever-blockscout-proxy:local 로 변경
```

## 향후 확장

- 컨트랙트 verify: Backend 가 이미 지원. Frontend의 verify 폼.
- 토큰 페이지: ERC20 자동 인덱싱.
- `stats` 컨테이너 활성화: tx/day, gas usage 등 정밀 차트.
- 도메인 매핑 + HTTPS: 이미 Cloudflare 통해 적용됨.

## 관련

- ArbiEver L2 인프라: https://github.com/makewalletfirst/ArbiEver
- 가벼운 Alethio Lite 익스플로러: https://github.com/makewalletfirst/AribiEver-Explorer
