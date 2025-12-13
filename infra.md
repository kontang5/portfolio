# macOS 기반 컨테이너 인프라: Observability 중심 홈서버 설계

> 단일 Mac mini에서 개발·배포·모니터링을 통합 운영하는 경량 인프라

`Docker` `Nginx` `PostgreSQL` `OpenTelemetry` `OpenObserve` `Cloudflare` `Node.js`

## 1. 프로젝트 개요

### 배경

클라우드 서비스 비용 부담 없이 개인 프로젝트를 실제 운영 환경에서 테스트하고 싶었습니다. 단순히 "돌아가는" 서버가 아니라, 실무에서 요구하는 보안·모니터링·유지보수성을 갖춘 인프라를 직접 설계하고 운영해보는 것이
목표였습니다.

### 핵심 목표

- 단일 장비로 통합 환경 구성: Mac mini 한 대에서 DB, API, 모니터링 스택 운영
- 보안 계층화: 네트워크 분리와 최소 권한 원칙 적용
- Observability 확보: 로그/메트릭/트레이스를 단일 대시보드에서 확인

코드베이스: [github.com/kontang5/playbook](https://github.com/kontang5/playbook)

## 2. 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                        Cloudflare Proxy                         │
│                   DDoS 방어 · CDN · Origin 인증서                  │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Nginx (80/443)                          │
│         Reverse Proxy · HTTPS Termination · File serving        │
└─────────────────────────────────────────────────────────────────┘ 
         │                       │                      │
         ▼                       ▼                      ▼
┌─────────────┐         ┌─────────────┐        ┌─────────────┐
│   Express   │         │  LM Studio  │        │   (Other)   │
│    :3000    │         │    :1234    │        │  Services   │
└─────────────┘         └─────────────┘        └─────────────┘
         │                       
         ▼                       
┌─────────────┐
│ PostgreSQL  │
│    :5432    │
└─────────────┘

         ┌──────────────────────────────────────────────────┐
         │               OTel Collector                     │
         │  Nginx · LM Studio · PostgreSQL · API            │
         │  Logs / metrics / traces                         │
         └──────────────────────────────────────────────────┘
                                 │
                                 ▼
                       ┌─────────────────┐
                       │   OpenObserve   │
                       │      :5080      │
                       └─────────────────┘
```

## 3. 주요 구현내용

### 3.1 네트워크 3분리 설계

| 네트워크            | 연결된 서비스                       | 설계 의도              |
|-----------------|-------------------------------|--------------------|
| public-network  | Nginx Only                    | 외부 노출 최소화 (단일 진입점) |
| private-network | Nginx, App, OTel, OpenObserve | 내부 서비스 간 통신        |
| db-network      | App, PostgreSQL, OTel         | DB 접근을 필요한 서비스로 제한 |

> 네트워크를 분리해 공격 표면을 최소화

### 3.2 PostgreSQL

#### Role 분리

| Role     | 권한           | 용도               |
|----------|--------------|------------------|
| observer | pg_monitor   | 모니터링 전용 (읽기만 가능) |
| demo_app | demo DB CRUD | 애플리케이션 전용        |

> 애플리케이션이 탈취되어도 다른 DB나 시스템 테이블에 접근 불가하도록 권한 최소화

#### DB 최적화

| 원칙      | 주요 설정                                                              | 의도                                                            |
|---------|--------------------------------------------------------------------|---------------------------------------------------------------|
| 메모리 효율  | `work_mem=32MB`, `max_connections=50`                              | 32MB × 50 = 1.6GB로 제한, 나머지는 OS 캐시(`effective_cache_size`)로 활용 |
| API 응답성 | `statement_timeout=30s`, `idle_in_transaction_session_timeout=60s` | 느린 쿼리와 좀비 연결을 빠르게 정리                                          |
| SSD 최적화 | `random_page_cost=1.1`                                             | 랜덤 I/O 비용을 낮춰 인덱스 스캔 적극 활용                                    |

> 스케일아웃이 불가능한 단일 장비 환경에서 timeout과 연결 제한으로 단일 요청이 전체 시스템을 블로킹하지 않도록 방어적으로 설계

### 3.3 OTel Collector 모듈화

```
otel-collector/
├── base.yaml           # 공통설정
├── docker.yaml         # Docker log collection (filelog receiver)
├── routing.yaml        # Log classification, filtering, routing
├── application.yaml    # App telemetry pipelines (OTLP receiver)
├── nginx.yaml          # Nginx metrics/logs
└── database.yaml       # PostgreSQL metrics/logs
 ```   

> 서비스별로 분리하여 유지보수가 편하게 구성하였습니다.

### 3.4 Observability 파이프라인 구축

| Signal  | Source                       | 수집 방식                      |
|---------|------------------------------|----------------------------|
| Logs    | Nginx, PostgreSQL, LM Studio | 파일 → OTel filelog receiver |
| Logs    | API (Express)                | OTLP 직접 전송                 |
| Metrics | 전체 컴포넌트                      | OTel receiver / OTLP       |
| Traces  | API (Express)                | OTLP 직접 전송                 |

#### Retention Policy

| Location    | Retention | Method                           |
|-------------|-----------|----------------------------------|
| Docker logs | 30MB      | 컨테이너당 용량 제한 (log rotation)       |
| OpenObserve | 30일       | `ZO_COMPACT_DATA_RETENTION_DAYS` |

> OpenObserve에서 trace → 관련 로그 → DB 쿼리 시간까지 단일 화면에서 추적 가능

### 3.5 로컬 LLM 서비스

| 항목    | 설정                   |
|-------|----------------------|
| 런타임   | LM Studio            |
| 모델    | Qwen3 8B (4-bit 양자화) |
| 엔드포인트 | `:1234`              |
| 프록시   | Nginx reverse proxy  |

> macOS Docker 환경에서는 Metal GPU 패스스루가 불가능하여 호스트에서 직접 실행

## 4. Trouble Shooting

### 4.1 OTel Collector가 Docker 로그에 접근하지 못하는 문제

#### 증상

- filelog receiver가 `/var/lib/docker/containers/` 경로의 로그 파일 읽기 실패
- `permission denied` 에러 발생

#### 원인

- Docker 로그 디렉토리는 root 소유 (0700 권한)
- OTel Collector 컨테이너는 non-root 유저로 실행

#### 해결

```yaml
# docker-compose.yml
otel-collector:
  user: root
  volumes:
    - /var/lib/docker/containers:/var/lib/docker/containers:ro
```

- Collector를 root로 실행하되, 볼륨은 read-only로 마운트하여 권한 최소화

### 4.2 컨테이너 시작 순서로 인한 연결 실패

#### 증상

- `docker-compose up` 시 간헐적으로 API → PostgreSQL 연결 실패
- Nginx → upstream 서비스 502 Bad Gateway

#### 원인

- `depends_on`만으로는 컨테이너 "시작"만 보장, 서비스 "준비 완료"는 보장하지 않음
- PostgreSQL이 컨테이너는 떴지만 연결 수락 전 상태에서 API가 접속 시도

#### 해결

health checker 사용

```yaml
# docker-compose.yml
services:
  postgres:
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres" ]
      interval: 30s
      timeout: 10s
      retries: 3

  api:
    depends_on:
      postgres:
        condition: service_healthy
```

추가로 초기 기동 시 순서 보장을 위한 스크립트 작성:

```bash
#!/bin/bash
# start.sh
docker compose -f database/docker-compose.yml up -d
until docker exec postgres pg_isready -U postgres > /dev/null 2>&1; do
  sleep 2
done
```

## 5. Decision Records

### 5.1 OpenObserve

- 비교 대상: ELK, PLG
- 선택 이유: 단일 바이너리로 log/metric/trace 통합, 압도적으로 낮은 메모리 사용량

### 5.2 Cloudflare Origin 인증서

- 비교 대상: Let's Encrypt
- 선택 이유: 인증서 갱신, 관리 편의성

---

## 6. 장애 경험과 교훈

### 상황

2025년 11월 18일, Cloudflare 서비스 장애 발생. Bot Management 기능의 버그로 인해 프록시 서버가 HTTP 요청을 처리하지 못해 서비스 접속 불가 상태가 되었습니다.

### 문제점

- CDN/프록시를 Cloudflare에 단일 의존 → 우회 경로 없음
- 장애 시 Origin 서버는 정상이어도 사용자 접근 불가

### 이후 조치

- 장애 시 빠른 DNS 전환을 위한 낮은 TTL 설정 (현재 300초)
- DNS 이중화 구성 계획(Cloudflare + fallback dns)

### 배운 점

SPOF는 서비스뿐 아니라 "장애 대응 수단"에도 적용된다는 것을 체감했습니다.

## 7. 향후 계획

| 항목      | 현재            | 목표                                     |
|---------|---------------|----------------------------------------|
| 프로비저닝   | 수동            | Ansible Playbook으로 더 간단한 셋업            |
| 환경 분리   | 단일 환경         | 서비스 구성 분리 (Docker Compose profiles 활용) |
| 백업      | 수동 pg_dump    | 자동 백업 + 스토리지서버 MinIO 연결                |

---

이지훈 | kontang5@icloud.com | [github.com/kontang5](https://github.com/kontang5)