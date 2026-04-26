# 인프라 아키텍처

전체 시스템 구성도. 사용자 브라우저부터 Mac mini의 컨테이너까지 어떻게 연결되는지.

## 전체 구성도

```mermaid
flowchart TB
  subgraph internet[" "]
    direction TB
    USER[사용자 브라우저]
  end

  subgraph cloudflare["Cloudflare 엣지"]
    direction TB
    CF_DNS[Authoritative DNS<br/>4 zones]
    CF_PROXY[CDN / TLS 종단 / WAF]
    CF_TUNNEL_EDGE[Tunnel Edge<br/>cfargotunnel.com]
  end

  subgraph macmini["Mac mini (가정용 회선 뒤)"]
    direction TB

    CLOUDFLARED[cloudflared<br/>outbound 연결 유지<br/>tunnel: storkspear-home]

    subgraph docker["Docker 컨테이너들"]
      direction LR

      subgraph kamal_lane["동적 앱 차선"]
        KAMAL[kamal-proxy<br/>:80 / :443]
        BACKEND[backend-server-web<br/>:8080]
        KAMAL --> BACKEND
      end

      subgraph nginx_lane["정적 사이트 차선"]
        NGINX[homepage-nginx<br/>:8088]
      end

      subgraph obs_lane["관측성"]
        GRAFANA[Grafana :3000]
        LOKI[Loki :3100]
        PROM[Prometheus :9090]
      end
    end

    SITES[(~/sites/<br/>nginx mount)]
    NGINX -.read.-> SITES

    TS[Tailscale<br/>관리자 SSH 전용]
  end

  subgraph other["다른 Tailscale 노드"]
    MINIO[MinIO :9000<br/>객체 스토리지]
  end

  ADMIN[관리자 노트북]

  USER -->|HTTPS| CF_DNS
  CF_DNS --> CF_PROXY
  CF_PROXY --> CF_TUNNEL_EDGE
  CF_TUNNEL_EDGE <-.유지된 outbound.-> CLOUDFLARED

  CLOUDFLARED -->|server.cloud| KAMAL
  CLOUDFLARED -->|log.cloud| GRAFANA
  CLOUDFLARED -->|storage.cloud| MINIO
  CLOUDFLARED -->|정적 8개| NGINX

  ADMIN -.SSH via Tailscale.-> TS
```

## 핵심 컴포넌트

### Cloudflare 엣지

- **Authoritative DNS:** 4개 zone (`storkspear.cloud`, `storkspear.co.kr`, `seoseji.site`, `moojigae.co.kr`). 11개 호스트 모두 `<tunnel-id>.cfargotunnel.com`을 가리키는 CNAME (proxied=true).
- **CDN / TLS 종단:** 모든 외부 트래픽은 Cloudflare에서 HTTPS 종단. Universal SSL 자동 발급.
- **Tunnel Edge:** Mac mini에서 들어오는 outbound 연결을 유지하고, 요청이 들어오면 그 연결로 되돌려 보냄.

### Mac mini (집)

- **cloudflared:** 단일 프로세스. CF로 항상 outbound 연결 유지. `~/.cloudflared/storkspear.yml`의 ingress 규칙대로 호스트별로 내부 컨테이너에 분기. launchd가 자동 재시작 관리.
- **Docker:**
    - `kamal-proxy` — Basecamp의 Kamal과 함께 쓰는 무중단 배포용 리버스 프록시. 새 컨테이너 띄워서 트래픽 swap.
    - `backend-server-web` — 실제 API 앱. `ghcr.io/storkspear/backend-server` 이미지.
    - `homepage-nginx` — 단순 정적 서빙 + www → apex 리다이렉트. nginx:alpine.
    - `grafana / loki / prometheus` — 관측성 스택.
- **`~/sites/`:** 정적 파일 마운트. `hello/`(placeholder 4개) + `portfolio/`(git clone). nginx가 read-only로 마운트.
- **Tailscale:** 관리자(노트북)에서 SSH 접속용. 외부 트래픽 라우팅에는 사용 안 함.

### 다른 노드

- **MinIO:** 객체 스토리지. Mac mini와 별도 머신, Tailscale로만 접근. cloudflared가 storage.storkspear.cloud 요청을 Tailscale IP로 직접 forward.

## 의존 그래프 (단일 장애점)

```mermaid
flowchart TD
  CF[Cloudflare] -.문제 시.-> ALL[전 호스트 다운]
  CFD[cloudflared 프로세스] -.죽으면.-> ALL2[전 호스트 다운<br/>launchd가 재시작]
  MAC[Mac mini 전원/네트워크] -.꺼지면.-> ALL3[전 호스트 다운]
  ISP[ISP 회선] -.끊기면.-> ALL4[전 호스트 다운]
  KAMAL[kamal-proxy] -.문제.-> APIONLY[API/log만 다운<br/>정적 사이트는 정상]
  NGINX[homepage-nginx] -.문제.-> STATICONLY[정적 8개만 다운<br/>API/log는 정상]
```

대부분의 일상적 작업(API 배포, 정적 사이트 업데이트)은 마지막 두 박스에 해당해서 영향 범위가 격리돼있음.
