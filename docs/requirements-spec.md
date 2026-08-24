# 부산이 프로젝트 — 요구사항 명세서

- 문서버전: v0.1 (아키텍처 확정 단계 기준)
- 작성일: 2026-08-24
- 관련 산출물: 아키텍처 다이어그램(Excalidraw), 기능 명세서(functional-spec.md)

## 1. 프로젝트 개요

### 1.1 목적
부산시 공공데이터포털(data.go.kr)이 제공하는 부산 관광 OpenAPI 4종을 수집·정제하여,
매일 1회 RAG(검색증강생성) 기반으로 "오늘의 추천 여행코스"를 자동 생성하고,
웹(React)에서 조회 및 대화형 질의가 가능한 여행 추천 서비스를 만든다.

### 1.2 배경
- SKALA 4기 AI/ML 전환 과정의 실습/포트폴리오 프로젝트
- 기존 stock-rag-pipeline(주식 분석 RAG) 프로젝트의 경험을 살리되, 백엔드는 기존 Java/Spring 역량을
  적극 활용하는 방향으로 스택을 재구성함 (Python/FastAPI/Airflow 대신 Spring Boot + Spring AI 단일 스택)

### 1.3 범위
포함:
- 부산 관광 OpenAPI 4종(도보여행/맛집/명소/테마여행, 국문) 수집·적재
- 임베딩 색인 및 벡터 검색(pgvector)
- 일 1회 배치로 "오늘의 추천 코스" 생성 및 캐싱
- 캐시 조회 API + 실시간 RAG 대화형 질의 API
- React 웹 프론트엔드
- Kubernetes 기반 배포

제외(향후 검토):
- 영문/일문/중문 등 다국어 콘텐츠 (1차는 국문만)
- 네이티브 모바일 앱 (아키텍처상 자리만 마련)
- 사용자 계정/로그인, 개인화 이력 기반 추천 (1차는 비로그인 익명 추천)
- 결제/예약 연동

### 1.4 이해관계자
| 구분 | 설명 |
|---|---|
| 서비스 이용자 | 부산 여행 코스를 추천받고 싶은 일반 사용자 |
| 개발/운영자 | 본인(1인 개발) — 기획/개발/배포/운영 전부 담당 |
| 데이터 제공기관 | 부산광역시 관광진흥과 (data.go.kr OpenAPI 제공) |

## 2. 기능 요구사항 (Functional Requirements)

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-01 | 시스템은 4개 부산 관광 OpenAPI(도보여행/맛집/명소/테마여행)에서 국문 콘텐츠를 매일 1회 수집한다 | 필수 |
| FR-02 | 시스템은 수집한 콘텐츠를 정규화하여 관계형 DB(PostgreSQL)에 upsert한다 (중복 없이 최신 상태 유지) | 필수 |
| FR-03 | 시스템은 신규/변경된 콘텐츠에 대해 임베딩을 생성하고 벡터스토어(pgvector)에 색인한다 | 필수 |
| FR-04 | 시스템은 매일 1회, 벡터 검색 + LLM을 이용해 "오늘의 추천 코스"(장소 조합 + 설명글)를 생성하고 캐시에 저장한다 | 필수 |
| FR-05 | 사용자는 웹에서 "오늘의 추천 코스"를 별도 로딩 지연 없이 즉시 조회할 수 있다 (캐시 조회) | 필수 |
| FR-06 | 사용자는 원하는 구군(區郡)이나 테마를 지정해 그에 맞는 추천 코스를 조회할 수 있다 | 선택 |
| FR-07 | 사용자는 자연어로 여행 질의를 입력하면 실시간 RAG 응답(온디맨드)을 받을 수 있다 | 필수 |
| FR-08 | 사용자는 장소 목록을 카테고리(도보여행/맛집/명소/테마)·구군별로 탐색할 수 있다 | 선택 |
| FR-09 | 시스템은 API 호출 실패 시 재시도하고, 반복 실패 시 실패 이력을 기록한다 | 필수 |
| FR-10 | 시스템은 data.go.kr 인증키, OpenAI API 키 등 민감정보를 코드에 하드코딩하지 않고 별도 관리한다 | 필수 |

**FR-04/05/07 수용 기준 (예시 — 나머지 FR도 구현 착수 전 동일하게 구체화 필요)**
- FR-04: 매일 05:10(KST, 배치 시작 10분 후) 기준으로 `daily_recommendations`에 오늘 날짜 row가 최소 1건 이상 존재한다
- FR-05: `GET /api/recommendations/today` 호출 시 응답까지 200ms(P95) 이내, 응답에 `courseSummary`와 1개 이상의 `places`가 포함된다
- FR-07: `POST /api/chat` 호출 시 5초(P95) 이내 응답하며, 응답에 `answer`가 비어있지 않다

## 3. 비기능 요구사항 (Non-Functional Requirements)

| ID | 구분 | 요구사항 |
|---|---|---|
| NFR-01 | 성능 | "오늘의 추천 코스" 조회 API는 캐시 조회이므로 평균 응답 200ms 이내 |
| NFR-02 | 성능 | 실시간 RAG 질의 API는 LLM 호출 특성상 5초 이내 응답 목표 (스트리밍 응답 검토) |
| NFR-03 | 가용성 | 배치(수집/임베딩/추천생성) 실패가 서빙 API 가용성에 영향을 주지 않아야 함 — 배치 실패 시에도 직전 캐시로 서빙 유지 |
| NFR-04 | 확장성 | 트래픽 증가 시 백엔드/프론트엔드 파드를 수평 확장할 수 있어야 함 (단, 배치 스케줄러의 중복 실행 문제는 별도 대응 필요 — 4.2절 참고) |
| NFR-05 | 보안 | 인증키/API 키는 Kubernetes Secret으로 관리하고 Git에 커밋하지 않는다 |
| NFR-06 | 비용 | 개발 단계는 로컬 Ollama로 LLM 비용 없이 개발, 운영 배포 시에만 OpenAI API 비용 발생 |
| NFR-07 | 데이터 신선도 | 관광 콘텐츠는 하루 단위 최신화로 충분 (실시간성 불필요) |
| NFR-08 | 운영성 | 배치 실행 이력(성공/실패, 수집 건수)을 조회할 수 있어야 함 |
| NFR-09 | 이식성 | 로컬(Docker Compose)과 운영(Kubernetes) 환경 모두에서 동일한 컨테이너 이미지로 구동 가능해야 함 |
| NFR-10 | 운영성 | Backend는 K8s liveness/readiness 프로브가 사용할 헬스체크 엔드포인트(`/actuator/health`)를 제공해야 함 — 없으면 Deployment가 파드 상태를 판단할 수 없어 롤링 업데이트/재시작이 불안정해짐 |
| NFR-11 | 보안/비용 | F5(실시간 RAG 질의) API는 인증 없는 공개 엔드포인트로, 무제한 호출 시 OpenAI 비용이 통제 불가능하게 늘어날 수 있음 → IP당 분당 호출 횟수 제한(Rate Limiting)을 필수로 적용 |

## 4. 배포/인프라 요구사항

### 4.1 배포 대상 워크로드

| 워크로드 | K8s 리소스 | 역할 |
|---|---|---|
| Frontend | Deployment | Nginx + React 정적 빌드 서빙 |
| Backend | Deployment | Spring Boot 앱 — 배치 스케줄러 + Spring AI(임베딩/RAG) + REST API가 한 프로세스 |
| Database | StatefulSet + PVC | PostgreSQL + pgvector |

### 4.2 파드(레플리카) 수 권장안

| 워크로드 | 개발(dev) | 운영(prod) 초기 | 비고 |
|---|---|---|---|
| Frontend | 1 | **2** | 정적 콘텐츠라 리소스 부담 적음. 2개면 무중단 배포(롤링 업데이트) 가능. 트래픽 늘면 HPA로 확장 |
| Backend | 1 | **1 (주의 필요)** | 아래 "배치 중복 실행 이슈" 참고 — 복제본을 늘리기 전에 반드시 대응 필요 |
| Database | 1 | 1 | 개인 프로젝트 규모에선 단일 인스턴스로 충분. PVC로 데이터는 파드 재시작에 영향받지 않음 |

**배치 중복 실행 이슈 (중요)**: Backend는 `@Scheduled` 크론잡이 애플리케이션 내부에 포함된 구조라,
Backend 파드를 2개 이상으로 늘리면 매일 배치(수집→임베딩→추천생성)가 파드 수만큼 중복 실행된다.
당장은 **Backend replicas=1**로 유지하는 것을 권장하며, 이후 트래픽이 늘어 REST API만 확장하고 싶다면:
- (A) [ShedLock](https://github.com/lukas-krecan/ShedLock) 라이브러리로 분산 락을 걸어 여러 파드 중 하나만 배치를 실행하게 하거나,
- (B) 배치 로직을 별도 K8s **CronJob**으로 분리하고, REST API 서빙 부분만 Deployment로 독립시켜 자유롭게 스케일아웃

둘 중 하나를 도입해야 한다. 1인 개발 초기 단계에서는 (A) ShedLock이 코드 변경이 적어 더 간단하다.

### 4.3 레이블링 전략

Kubernetes 공식 권장 레이블 스킴([app.kubernetes.io](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/))을
기준으로, 모든 리소스에 공통 레이블 + 컴포넌트별 구분 레이블을 부여한다.

**공통 레이블 (모든 리소스)**
```yaml
labels:
  app.kubernetes.io/part-of: busan-trip-rag       # 프로젝트 전체를 묶는 상위 레이블
  app.kubernetes.io/managed-by: kubectl            # 또는 helm 도입 시 helm
  app.kubernetes.io/version: "0.1.0"                # 이미지/릴리즈 버전
  env: dev                                          # dev | prod — 환경 구분(커스텀 레이블)
```

**컴포넌트별 레이블 (Deployment/StatefulSet/Pod/Service마다 추가)**

| 컴포넌트 | `app.kubernetes.io/name` | `app.kubernetes.io/component` | `app.kubernetes.io/instance` |
|---|---|---|---|
| Frontend | `busan-trip-rag-frontend` | `frontend` | `busan-trip-rag-frontend-dev` |
| Backend | `busan-trip-rag-backend` | `backend` | `busan-trip-rag-backend-dev` |
| Database | `busan-trip-rag-postgres` | `database` | `busan-trip-rag-postgres-dev` |

**적용 예 (Backend Deployment)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busan-trip-rag-backend
  labels:
    app.kubernetes.io/name: busan-trip-rag-backend
    app.kubernetes.io/instance: busan-trip-rag-backend-dev
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: busan-trip-rag
    app.kubernetes.io/managed-by: kubectl
    app.kubernetes.io/version: "0.1.0"
    env: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: busan-trip-rag-backend   # Selector는 안정적인 subset만 사용 (version/env 등 자주 바뀌는 값은 제외)
  template:
    metadata:
      labels:
        app.kubernetes.io/name: busan-trip-rag-backend
        app.kubernetes.io/instance: busan-trip-rag-backend-dev
        app.kubernetes.io/component: backend
        app.kubernetes.io/part-of: busan-trip-rag
        app.kubernetes.io/managed-by: kubectl
        app.kubernetes.io/version: "0.1.0"
        env: dev
```

> Selector에는 `version`처럼 배포마다 바뀌는 레이블을 넣지 않는다 — Deployment의 `spec.selector`는
> **불변**이라 값이 바뀌는 레이블을 넣으면 다음 배포에서 롤아웃이 깨진다. 안정적인 `name`(+`instance`)만 selector로 쓰고,
> `version`/`env`는 조회·필터링용 부가 레이블로만 사용한다.

**Service는 대상 Pod의 `name` 레이블로 selector를 건다** (Frontend/Backend/Database 각각 Service 1개씩, ClusterIP).

## 5. 데이터 요구사항

원본: data.go.kr 부산시 공공데이터 OpenAPI 4종 (기관코드 6260000)

| API | 오퍼레이션(국문) | 주요 필드 |
|---|---|---|
| WalkingService (부산도보여행정보) | getWalkingKr | UC_SEQ, MAIN_TITLE, CATE2_NM, LAT/LNG, PLACE, TITLE, SUBTITLE, TRFC_INFO, ITEMCNTNTS |
| FoodService (부산맛집정보) | getFoodKr | 상동 + GUGUN_NM, ADDR1/2, CNTCT_TEL, USAGE_DAY_WEEK_AND_TIME, RPRSNTV_MENU |
| AttractionService (부산명소정보) | getAttractionKr | 상동 + GUGUN_NM, ADDR1/2, CNTCT_TEL, TRFC_INFO |
| RecommendedService (부산테마여행정보) | getRecommendedKr | 상동 + GUGUN_NM |

내부 저장: `places`(통합 테이블), `daily_recommendations`(추천 캐시), `vector_store`(Spring AI 관리 임베딩), `ingestion_runs`(배치 로그)
— 상세 스키마는 아키텍처 문서 및 기능 명세서 참고.

## 6. 제약사항 및 가정

- data.go.kr 인증키는 발급까지 승인 대기 시간이 있을 수 있음 (활용신청 후 즉시/수일 내 승인)
- 4개 API 모두 "년 1회" 갱신 주기로 명시되어 있어, 실제로는 매일 배치를 돌려도 원본 데이터 자체는 자주 바뀌지 않음
  → 배치는 "매일 갱신을 시도"하되, RAG 추천 코스의 다양성은 동일 데이터 내에서의 조합/테마 로테이션으로 확보
- OpenAI API는 유료이므로 운영 단계 비용을 모니터링해야 함 (임베딩은 상시 OpenAI, 채팅 생성만 운영에서 OpenAI)
- 1인 개발 프로젝트 규모를 전제로 하며, 대규모 트래픽 대응은 범위 밖

## 7. 용어 정의

| 용어 | 설명 |
|---|---|
| RAG | Retrieval-Augmented Generation, 검색증강생성 — 벡터 검색 결과를 근거로 LLM이 응답을 생성하는 방식 |
| pgvector | PostgreSQL에서 벡터 유사도 검색을 지원하는 확장(extension) |
| UC_SEQ | data.go.kr 응답의 콘텐츠 고유 ID |
| 구군(GUGUN_NM) | 부산광역시의 구/군 행정구역 |
