# Smoke Test 체크리스트

운영 변경 후 (예: 컨테이너 recreate, 폴더 mv, repo 리네임) 정상 동작 검증용 URL 체크리스트.

---

## :material-web: 정적 사이트 (homepage-nginx, 5개)

| URL | 기대 응답 |
|---|---|
| <https://seoseji.site> | 200 |
| <https://www.seoseji.site> | 301 → `seoseji.site` |
| <https://portfolio.seoseji.site> | 200 |
| <https://storkspear.co.kr> | 200 |
| <https://www.storkspear.co.kr> | 301 → `storkspear.co.kr` |
| <https://moojigae.co.kr> | 200 |
| <https://www.moojigae.co.kr> | 301 → `moojigae.co.kr` |
| <https://dev.moojigae.co.kr> | 200 (`<h1>hello, dev.moojigae</h1>` 포함) |

---

## :material-server-network: 백엔드 / 관측

| URL | 기대 응답 |
|---|---|
| <https://server.storkspear.cloud/actuator/health> | 200 / `{"status":"UP", ...}` |
| <https://server.storkspear.cloud/actuator/health/liveness> | 200 / `{"status":"UP"}` |
| <https://server.storkspear.cloud/> | 401 (인증 필요 — 정상) |
| <https://log.storkspear.cloud/> | 302 → Grafana 로그인 |

---

## :material-book-open-page-variant: 문서 사이트 (GitHub Pages)

| URL | 비고 |
|---|---|
| <https://storkspear.github.io/docs-template-spring/> | template-spring 문서 |
| <https://storkspear.github.io/docs-template-flutter/> | template-flutter 문서 |
| <https://storkspear.github.io/docs-infra-homeserver/> | (이 문서) |

!!! note "빌드 시간"
    Push 후 GitHub Actions 가 자동 빌드. 보통 1-3분, 길면 5분. 빌드 중에는 404 가능.

---

## :material-github: GitHub 리포 (16개, 컨벤션 검증)

```
template-*       (4) : template-spring, template-flutter, template-react, legacy-template-spring (📦)
docs-*           (3) : docs-template-spring, docs-template-flutter, docs-infra-homeserver
server-*         (1) : server-factory
app-*            (3) : app-sumtally, app-richandyoung, app-chapchap (📦, public)
site-*           (5) : site-seji-portfolio, site-seji, site-storkspear, site-moojigae, site-moojigae-dev (없음 — dev 는 site-moojigae 의 브랜치)
```

확인 포인트:
- prefix 일관성 (알파벳 정렬 시 같은 그룹끼리 묶임)
- archived 표시 (`legacy-*`, `app-chapchap`)
- Visibility (대부분 private, docs-* 와 `app-chapchap`, `site-seji-portfolio` 는 public)

---

## :material-rocket-launch: 일괄 검증 스크립트

```bash
#!/usr/bin/env bash
# 모든 도메인 한 번에 응답코드 체크

URLS=(
  https://seoseji.site
  https://portfolio.seoseji.site
  https://storkspear.co.kr
  https://moojigae.co.kr
  https://dev.moojigae.co.kr
  https://server.storkspear.cloud/actuator/health
  https://log.storkspear.cloud/
  https://storkspear.github.io/docs-template-spring/
  https://storkspear.github.io/docs-template-flutter/
  https://storkspear.github.io/docs-infra-homeserver/
)

for u in "${URLS[@]}"; do
  printf "%-65s : HTTP %s\n" "$u" "$(curl -sS -o /dev/null -w "%{http_code}" --max-time 8 "$u")"
done
```

기대 결과: 모두 200 (또는 302 — log.storkspear.cloud 의 Grafana redirect).

---

## :material-magnify: 콘텐츠 spot check

placeholder 사이트는 응답코드 외에 본문도 한 번 확인. Mac mini 의 git pull 이 정상 반영됐는지 검증용.

```bash
echo "[main] moojigae.co.kr h1:"
curl -sS https://moojigae.co.kr/ | grep -o '<h1>[^<]*</h1>'

echo "[dev]  dev.moojigae.co.kr h1:"
curl -sS https://dev.moojigae.co.kr/ | grep -o '<h1>[^<]*</h1>'
```

기대:
```
[main] moojigae.co.kr h1: <h1>hello, moojigae</h1>
[dev]  dev.moojigae.co.kr h1: <h1>hello, dev.moojigae</h1>
```

main 과 dev 의 h1 이 다르면 = 브랜치 분기 정상 작동.

---

## :material-chart-timeline: 컨테이너 상태 점검 (Mac mini SSH 후)

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

기대 (6개):
```
homepage-nginx                  Up <duration>
observability-grafana           Up <duration>
observability-loki              Up <duration>
observability-prometheus        Up <duration>
backend-server-web-<sha>        Up <duration>     # 다음 deploy 시 server-factory-web-<sha> 로 교체
kamal-proxy                     Up <duration>
```

!!! warning "alertmanager 가 보이면"
    의도적으로 stop+rm 한 상태가 정상. 다시 떠있다면 누군가 `compose up` 해서 부활시킨 것 — 의도된 활성화면 OK, 아니면 다시 stop.
