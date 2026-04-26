# Stork Spear 홈 서버 인프라

Mac mini 한 대로 11개 호스트네임을 서빙하는 홈 서버 셋업의 구조와 흐름을 문서화한 곳입니다.

## 개요

서울 어느 책상 위의 Mac mini가:

- 4개 도메인 (`storkspear.cloud`, `storkspear.co.kr`, `seoseji.site`, `moojigae.co.kr`)
- 11개 호스트네임
- 백엔드 API + 정적 사이트 + 관측성(Grafana/Loki/Prometheus) + 객체 스토리지

를 단일 Cloudflare Tunnel로 외부에 노출하고 있어요. **포트 포워딩 없음, 공인 IP 노출 없음, 동적 DNS 클라이언트 불필요.**

## 빠르게 보기

<div class="grid cards" markdown>

- :material-dns:{ .lg .middle } **[서빙 중인 DNS](dns-list.md)**
  ---
  도메인별 호스트네임 11개와 각각의 라우팅 대상

- :material-sitemap:{ .lg .middle } **[인프라 아키텍처](architecture.md)**
  ---
  전체 시스템 구성도 — 사용자 → CF → cloudflared → 컨테이너

- :material-arrow-decision:{ .lg .middle } **[요청 처리 플로우](request-flow.md)**
  ---
  API 요청 / 정적 사이트 / www 리다이렉트 / 배포의 시퀀스 다이어그램

- :material-stack-overflow:{ .lg .middle } **[기술 스택](tech-stack.md)**
  ---
  Cloudflare Tunnel · Tailscale · Docker · Kamal · nginx · GitHub Actions

- :material-compare:{ .lg .middle } **[대안 비교](alternatives.md)**
  ---
  DuckDNS / 직접 포트 노출 / Tailscale Funnel / ngrok 과의 차이, 왜 안 썼는지

</div>

## 핵심 한 줄 요약

> **outbound 터널** 한 개로 모든 트래픽을 받기 때문에, 가정용 회선의 동적 IP·NAT·방화벽이 무력화된다.

## 마지막 업데이트

이 문서는 GitHub Actions로 자동 빌드되어 GitHub Pages에서 서빙됩니다.
저장소: [storkspear/storkspear-homeserver-infra-docs-viewer](https://github.com/storkspear/storkspear-homeserver-infra-docs-viewer)
