# docs-infra-homeserver

Stork Spear 홈 서버(Mac mini) 인프라 문서.

🌐 **Live:** https://storkspear.github.io/docs-infra-homeserver/

## 다루는 내용

- 서빙 중인 DNS 11개 (4개 도메인) — 각 호스트가 어디로 라우팅되는지
- 인프라 아키텍처 — Cloudflare Tunnel · Tailscale · Docker · Kamal · nginx
- 요청 처리 플로우 — API / 정적 / www 리다이렉트 / 배포 시퀀스 다이어그램
- 기술 스택 — 각 컴포넌트의 역할과 선택 이유
- 대안 비교 — DuckDNS / Tailscale Funnel / ngrok 등과의 차이

## 로컬에서 보기

```bash
pip install mkdocs-material
mkdocs serve
# → http://127.0.0.1:8000
```

## 배포

`main`에 push하면 GitHub Actions가 mkdocs build → Pages 배포.
