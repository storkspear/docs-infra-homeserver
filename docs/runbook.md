# 런북 (Runbook)

운영 일상 작업 절차. 일이 닥쳤을 때 빠르게 펼쳐보는 용도.

---

## :material-key: SSH 접속

```bash
ssh -i ~/.ssh/macmini -o IdentitiesOnly=yes -o IdentityAgent=none storkspear@home-mac-mini-m1
```

!!! note "왜 옵션이 많나"
    - 사용자명 `storkspear` (현재 로컬 사용자와 다를 수 있어 명시 필수)
    - `IdentitiesOnly=yes` + `IdentityAgent=none` — 다른 키들이 시도되어 "Too many auth failures" 발생 방지
    - Tailscale 호스트네임 `home-mac-mini-m1` (앱스토어 Tailscale 이라 `tailscale ssh` 미지원)

---

## :material-folder-network: 폴더 구조

```
~/workspace/
├── moojigae-frontend/      # Bitbucket (storkspear 무관)
├── server-factory/         # Spring 백엔드 (Kamal deploy)
│   └── infra/
│       ├── docker-compose.observability.yml
│       └── grafana-data/, loki-data/, prometheus-data/
└── sites/
    ├── nginx.conf                      # homepage-nginx 컨테이너 mount
    ├── site-seji-portfolio/            # portfolio.seoseji.site
    ├── site-seji/                      # seoseji.site
    ├── site-storkspear/                # storkspear.co.kr
    ├── site-moojigae/        (main)    # moojigae.co.kr
    └── site-moojigae-dev/    (dev)     # dev.moojigae.co.kr
```

홈 직속에는 운영 git repo 없음 (의도). `~/.kamal/`, `~/.cloudflared/`, `~/duckdns/` 등 dotfile/스크립트만.

---

## :material-docker: 컨테이너 정의 위치

| 컨테이너 | 정의 위치 | 비고 |
|---|---|---|
| **homepage-nginx** | **manual `docker run`** (compose 미관리) | 정의 파일 없음 — 아래 recreate 명령 외우세요 |
| observability-{grafana,loki,prometheus} | `~/workspace/server-factory/infra/docker-compose.observability.yml` | volume path 가 상대 (`./`) |
| observability-alertmanager | 동일 compose | **의도적 미실행** ([활성화 절차](#alertmanager) 참조) |
| backend-server-web-* | Kamal 자동 생성 | 직접 안 건드림 |
| kamal-proxy | Kamal 자동 생성 | mount: `~/.kamal/proxy/apps-config` |

---

## :material-restart: homepage-nginx recreate

!!! danger "정의 미관리 — 이 명령을 잃으면 복구 어려움"
    `homepage-nginx` 는 docker compose 가 아닌 manual `docker run` 으로 떠있어서, 정의가 어디에도 적혀있지 않습니다. 반드시 이 페이지를 통해 보존하세요.

```bash
# stop + remove
docker stop homepage-nginx && docker rm homepage-nginx

# recreate (mount source 가 ~/workspace/sites)
docker run -d \
  --name homepage-nginx \
  --restart unless-stopped \
  -p 8088:80 \
  -v ~/workspace/sites:/sites:ro \
  -v ~/workspace/sites/nginx.conf:/etc/nginx/conf.d/default.conf:ro \
  nginx:alpine
```

검증:
```bash
docker exec homepage-nginx nginx -t
# 그 후 도메인 5개 응답 확인 — smoke-test.md
```

---

## :material-plus-network: 새 도메인 추가 절차

예: `newsite.example.com` 정적 사이트 추가.

1. **GitHub repo 생성**: `site-newsite` (private)
2. **워크스페이스에서 셋업**:
   ```bash
   mkdir ~/workspace/site-newsite && cd $_
   echo '<h1>hello, newsite</h1>' > index.html
   git init -b main && git add -A
   git commit -m "chore: initial placeholder"
   gh repo create storkspear/site-newsite --private --source=. --push
   ```
3. **Cloudflare DNS** — `newsite.example.com` 을 cloudflared 터널로 매핑
4. **`~/.cloudflared/storkspear.yml`** 의 ingress 에 추가:
   ```yaml
   - hostname: newsite.example.com
     service: http://localhost:8088
   ```
5. **Mac mini 에서 clone**:
   ```bash
   cd ~/workspace/sites
   gh repo clone storkspear/site-newsite
   ```
6. **`~/workspace/sites/nginx.conf` 에 server 블록 추가**:
   ```nginx
   server { listen 80; server_name newsite.example.com; root /sites/site-newsite; index index.html; }
   ```
7. **homepage-nginx restart**:
   ```bash
   docker restart homepage-nginx
   ```
8. **launchctl 로 cloudflared reload**:
   ```bash
   launchctl unload ~/Library/LaunchAgents/site.server-factory.cloudflared.plist
   launchctl load ~/Library/LaunchAgents/site.server-factory.cloudflared.plist
   ```
9. **검증**: `curl https://newsite.example.com/`

---

## :material-bell-alert: alertmanager 활성화 { #alertmanager }

기본 상태: **의도적으로 미실행**. 활성화하려면 Discord webhook URL 이 필요합니다.

```bash
# 1. webhook URL 환경변수 설정 (영속성 위해 ~/.zshrc 같은 곳에)
export DISCORD_WEBHOOK_URL='https://discord.com/api/webhooks/...'

# 2. 해당 환경변수가 보이는 셸에서 compose up
cd ~/workspace/server-factory/infra
docker compose -f docker-compose.observability.yml up -d alertmanager

# 3. fail 안 하는지 확인
docker logs observability-alertmanager --tail 20
docker ps --filter name=alertmanager
```

비활성화로 돌아가려면:
```bash
docker compose -f docker-compose.observability.yml stop alertmanager
docker compose -f docker-compose.observability.yml rm -f alertmanager
```

---

## :material-alert: 알려진 함정

### OrbStack/Docker bind mount sync 지연

호스트에서 mount 된 파일을 수정한 직후 컨테이너 안에서 보면 **파일이 잘려보이는** 경우 있음. nginx config 수정 후 reload 시 syntax error 가 자주 나는 원인.

**해결**: `docker restart <container>` (mount 다시 잡힘) 또는 `docker exec ... cp` 로 직접 갱신.

### macOS BSD `sed` 가 `\b` (word boundary) 미지원

```bash
# ❌ Linux GNU sed 패턴 — macOS 에선 silent fail
sed -i "" "s|/sites/portfolio\b|/sites/seji-portfolio|g" file

# ✅ 단순 치환
sed -i "" "s|/sites/portfolio|/sites/seji-portfolio|g" file
```

### Mac mini 가 GitHub https credential 비대화형 SSH 에서 못 받음

macOS Keychain 이 GUI 세션 잠금이라 `git clone https://...` 가 SSH 세션에서 fail.

**해결**: `gh repo clone` 사용. gh CLI 가 storkspear 로 인증돼있어 자동 처리.

```bash
gh repo clone storkspear/<repo-name>
```

### GitHub repo rename 시 GHCR 이미지 자동 이전 안 됨

`github.com/owner/old-name` → `github.com/owner/new-name` 으로 repo rename 해도 `ghcr.io/owner/old-name` 이미지는 자동으로 새 namespace 로 가지 않음.

**해결**: 새 namespace `ghcr.io/owner/new-name` 은 다음 push 때 자동 생성됨. 옛 namespace 의 패키지는 web UI 에서 수동 삭제.

### GitHub Free + Private repo = Branch protection / Ruleset 불가

`Upgrade to GitHub Pro` 메시지로 차단됨. Public repo 는 무료. 솔로 인디는 protection 없이도 무방.

### GitHub Actions 분 한도 초과 시 즉시 fail (step 0개)

월 분 한도 초과되면 모든 workflow 가 step 진입 전 즉시 abort. annotation 도 비어있어 진단 어려움. **매월 1일 충전**.

확인:
```bash
gh run list --repo storkspear/<repo> --limit 5 --json status,conclusion
# 모든 run 이 즉시 failure 면 quota 의심
```
