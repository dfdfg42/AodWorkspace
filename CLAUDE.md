# AOD (All of Dopamine) - Claude 작업 가이드

## 모듈별 상세 가이드
- **백엔드**: [back/](back/) → 하위 `CLAUDE.md` 및 `docs/` 참고
- **프론트엔드**: [front/CLAUDE.md](front/CLAUDE.md)
- **AI**: [ai/](ai/)

---

## 프로젝트 한 줄 요약
영화/OTT/게임/웹툰/웹소설 정보를 N개 플랫폼에서 크롤링·집계하여 AI 맞춤 추천을 제공하는 서비스.
가톨릭대 창업동아리 팀 프로젝트. 팀장이 PM+백엔드 주담당.

## 상세 문서
작업 전 관련 문서를 먼저 읽을 것:
- [back/docs/1_PROJECT_OVERVIEW.md](back/docs/1_PROJECT_OVERVIEW.md) — 전체 아키텍처, 멀티모듈 구조
- [back/docs/2_JOB_QUEUE_ARCHITECTURE.md](back/docs/2_JOB_QUEUE_ARCHITECTURE.md) — Producer-Consumer, SKIP LOCKED
- [back/docs/3_TRANSFORM_ENGINE.md](back/docs/3_TRANSFORM_ENGINE.md) — YAML 매핑 룰(v4), Ingest 파이프라인 (런타임 트레이스는 docs/8 참고)
- [back/docs/4_SCALABILITY_AND_PERFORMANCE.md](back/docs/4_SCALABILITY_AND_PERFORMANCE.md) — GIN 인덱스, Selenium 재사용, tini

---

## 모듈 구조
```
back/
├── -AOD-All-of-Dopamine-shared/   # JPA 엔티티 + 리포지토리 (공유 라이브러리)
├── -AOD-All-of-Dopamine-api/      # REST API 서버 (포트 8080)
└── -AOD-All-of-Dopamine-crawler/  # 크롤링 + 배치 변환 서버 (포트 8081)
```

## 핵심 엔티티 구조 (3-Tier)
```
Content (마스터 테이블, domain: MOVIE/TV/GAME/WEBTOON/WEBNOVEL)
  ├── GameContent / MovieContent / TvContent / WebtoonContent / WebnovelContent  [1:1, @MapsId]
  ├── PlatformData [1:N] — 플랫폼별 URL, attributes(JSONB)
  └── RawItem — 크롤링 원본 payload(JSONB) 스테이징 테이블
```
**주의:** `@MapsId` 자식 엔티티는 반드시 `Persistable<Long>` 구현 + `isNew=true` 플래그 필요 (JPA가 INSERT를 UPDATE로 오판하는 버그 방지).

## 크롤링 아키텍처
- **Producer** → `crawl_job` 테이블에 PENDING 상태로 bulk insert 후 즉시 반환
- **Consumer** `@Scheduled(fixedDelay=10s)` → `SKIP LOCKED`으로 배치 처리
- **JobExecutor 인터페이스** (Strategy Pattern) → 새 플랫폼 추가 시 이 인터페이스만 구현하면 자동 등록, Consumer 코드 수정 불필요
- 배치 사이즈: Selenium 기반(웹툰) 1개/5초, HTTP API(Steam/TMDB) 5개/5초, Jsoup(소설) 33개/5초

## YAML 매핑 룰
- 위치: `-AOD-All-of-Dopamine-crawler/src/main/resources/rules/`
- 새 플랫폼 추가 = yml 파일 하나 추가 (Java 코드 수정 없음)
- `IngestPipeline` → `RuleRegistry`(기동 시 yml 전부 스캔·검증, 오타 = 부팅 실패) → `DraftAssembler`가 RawItem JSON을 `Content + 도메인 엔티티 + PlatformData`로 직접 조립 후 병합/저장
- 목적지 접두사: `master.*`(contents) / `domain.*`(도메인 테이블) / `platform.*`(platform_data) / `attr.*`(JSONB)

## 핵심 설계 원칙 (작업 시 반드시 준수)
1. **OCP**: 새 플랫폼/도메인 추가 시 기존 코드 수정 없이 확장 가능해야 함
2. **Persistable**: `@MapsId` 자식 엔티티 신규 작성 시 반드시 `Persistable<Long>` 구현
3. **페이징**: 대용량 조회는 `Pageable.unpaged()` 절대 금지, `Slice` 사용
4. **Selenium**: `ThreadLocal` 기반 WebDriver 재사용 (최대 30회), 매번 새 인스턴스 금지
5. **인덱스**: 배열 필드 필터링은 GIN 인덱스 + `@>` 연산자 사용

## 주요 기술 스택
- Spring Boot 3.x, Java 17, Gradle 멀티모듈
- PostgreSQL (GIN 인덱스, JSONB, SKIP LOCKED)
- Selenium 4, JSoup, Spring WebFlux
- Prometheus + Grafana, Sentry
- Docker + tini (좀비 프로세스 방지)
- EC2 t3.small 최적화: HikariCP max-pool=5, Tomcat threads max=20
