# 부산이 프로젝트 — 기능 명세서

- 문서버전: v0.1
- 작성일: 2026-08-24
- 상위 문서: requirements-spec.md (요구사항 명세서)

각 기능은 요구사항 명세서의 FR 항목과 매핑된다.

---

## F1. 관광 콘텐츠 수집 배치 (FR-01, FR-02, FR-09)

| 항목 | 내용 |
|---|---|
| 트리거 | Backend 내부 `@Scheduled` 크론 (매일 05:00 KST, 설정: `busan.batch.cron`) |
| 입력 | data.go.kr 4개 OpenAPI × 국문(Kr) — WalkingService, FoodService, AttractionService, RecommendedService |
| 처리 | 1) 카테고리별로 `numOfRows` 페이지네이션 전체 순회 호출 → 2) 응답 아이템을 공통 스키마로 정규화(정제) → 3) `(uc_seq, category, lang)` 기준 upsert |
| 출력 | `places` 테이블 반영, `ingestion_runs`에 실행 로그(수집 건수/성공/실패) 1건 기록 |
| 예외 처리 | - API 5xx/타임아웃: 지수 백오프로 최대 3회 재시도<br>- `resultCode != 00`: 해당 카테고리만 실패 처리하고 나머지 카테고리는 계속 진행<br>- 3회 재시도 후에도 실패: `ingestion_runs.status = failed` 기록, 배치 전체는 중단하지 않음(부분 성공 허용) |
| 완료 조건 | 4개 카테고리 모두 시도 완료 (일부 실패해도 배치는 다음 단계로 진행) |

## F2. 임베딩 색인 (FR-03)

| 항목 | 내용 |
|---|---|
| 트리거 | F1 완료 후 이어서 실행 (같은 배치 파이프라인의 다음 단계) |
| 대상 선정 | `places` 중 `embedded_at IS NULL OR embedded_at < updated_at` (신규/변경분만) |
| 처리 | 1) `Place.toEmbeddingText()`로 제목/부제/장소/구군/대표메뉴/상세내용을 합친 텍스트 생성 → 2) Spring AI `EmbeddingModel`(OpenAI text-embedding-3-small, 1536차원)로 임베딩 → 3) `VectorStore.add(Document)`로 색인 (metadata: `placeId`, `category`, `gugunNm`) → 4) `places.embedded_at` 갱신 |
| 출력 | Spring AI가 관리하는 `vector_store` 테이블에 임베딩 반영 |
| 예외 처리 | 임베딩 API 실패 시 해당 배치는 스킵하고 다음 배치에서 재시도 (embedded_at이 갱신되지 않으므로 자동으로 다시 대상에 포함됨) |

## F3. 오늘의 추천 코스 생성 (RAG) (FR-04)

| 항목 | 내용 |
|---|---|
| 트리거 | F2 완료 후 이어서 실행 |
| 검색 조건 | **고정 테마 목록 3개를 매일 전부 생성한다** (원본 데이터가 년 1회 갱신이라 데이터 자체는 거의 안 바뀌므로, 굳이 "오늘의 테마를 선택"하는 로직 없이 매일 3개 다 만드는 쪽이 더 단순하고 비용도 낮음):<br>① `도심 산책과 로컬 맛집` (WALKING+FOOD 중심)<br>② `가족과 함께하는 명소 나들이` (ATTRACTION 중심)<br>③ `부산 대표 테마 코스` (THEME 중심)<br>각 테마별로 `SearchRequest`(topK=8, 카테고리별 최소 1건 필터링)로 벡터 검색 수행 |
| 생성 | 검색된 장소 후보들을 컨텍스트로 `ChatClient`에 전달 → **장소 4곳(고정값, `busan.recommendation.places-per-course`로 설정화)**을 묶어 하루 코스로 구성하고, 이동 동선과 한 줄 소개를 포함한 코스 설명을 생성하도록 프롬프트 구성 |
| 출력 | `daily_recommendations`에 `(rec_date, theme, gugun_nm)` 단위로 upsert — `place_ids`(코스 순서), `course_summary`(LLM 생성 설명), `generated_by`(모델 식별자) |
| 예외 처리 | LLM 호출 실패 시 해당 테마만 스킵, 다른 테마는 계속 생성. 당일 추천이 하나도 생성되지 못하면 F4 조회 API는 "가장 최근 성공한 날짜"의 캐시를 폴백으로 반환 |

## F4. 오늘의 추천 코스 조회 API (FR-05, FR-06)

```
GET /api/recommendations/today?gugun={구군}&theme={테마}
```

| 항목 | 내용 |
|---|---|
| 파라미터 | `gugun`(선택), `theme`(선택) — 둘 다 없으면 오늘 생성된 추천 중 기본 테마 반환 |
| 처리 | `daily_recommendations`에서 `rec_date=오늘`(+조건) 조회 → 없으면 가장 최근 날짜로 폴백 → `place_ids`로 `places` 상세 조인 |
| 응답 예시 | ```json\n{\n  \"recDate\": \"2026-08-24\",\n  \"theme\": \"도심 산책과 로컬 맛집\",\n  \"gugunNm\": null,\n  \"courseSummary\": \"...\",\n  \"places\": [ { \"id\": 58, \"title\": \"...\", \"category\": \"WALKING\", \"lat\": 35.116, \"lng\": 129.038, ... } ]\n}\n``` |
| 상태 코드 | 200 정상 / 200(폴백, `stale: true` 필드 포함) / 404 폴백 캐시조차 없을 때(최초 배치 이전) |

## F5. 실시간 RAG 질의 API (FR-07)

```
POST /api/chat
Body: { "message": "가족이랑 갈만한 하루 코스 추천해줘" }
```

| 항목 | 내용 |
|---|---|
| 처리 | 1) 사용자 메시지를 쿼리로 벡터 검색(topK=6) → 2) 검색 결과를 컨텍스트로 `ChatClient` 프롬프트 구성 → 3) 응답 생성 |
| 응답 | `{ "answer": "...", "referencedPlaces": [ {id, title, category} ] }` |
| 비고 | 캐시 없이 매 요청 LLM 호출(온디맨드) — NFR-02(5초 이내) 목표. 추후 스트리밍(SSE) 응답으로 체감 속도 개선 검토 |
| Rate Limit | NFR-11에 따라 **IP당 분당 5회**로 제한 (Bucket4j 등 사용). 초과 시 429 반환 |
| 예외 처리 | LLM/벡터스토어 장애 시 503 반환 + 사용자에게 "잠시 후 다시 시도해주세요" 메시지 |

## F6. 장소 목록 조회 API (FR-08)

```
GET /api/places?category={WALKING|FOOD|ATTRACTION|THEME}&gugun={구군}&page={n}
```

단순 페이지네이션 목록 조회. 정렬 기준은 최신 수집순(`fetched_at desc`).

## F7. 프론트엔드 화면 (React)

| 화면 | 설명 | 연동 API |
|---|---|---|
| 홈 | 오늘의 추천 코스 카드(장소 리스트 + 지도 마커 + 코스 설명) | F4 |
| 챗 | 자유 질의 입력창 + 답변/참조 장소 카드 | F5 |
| 장소 탐색 | 카테고리/구군 필터가 있는 목록 + 상세 모달 | F6 |

## F8. 배치 실행 이력 조회 (NFR-08, 운영자용)

```
GET /api/admin/ingestion-runs?limit=20
```
최근 배치 실행 이력(성공/실패, 수집 건수, 소요시간)을 반환. 1차는 인증 없이 내부망 전용으로 두고,
운영 노출 시 별도 인증 추가 검토.

---

## 부록 A. API 엔드포인트 요약

| 메서드 | 경로 | 기능 |
|---|---|---|
| GET | `/api/recommendations/today` | F4 오늘의 추천 코스 조회 |
| POST | `/api/chat` | F5 실시간 RAG 질의 |
| GET | `/api/places` | F6 장소 목록 조회 |
| GET | `/api/admin/ingestion-runs` | F8 배치 이력 조회 |
| GET | `/actuator/health` | NFR-10 K8s liveness/readiness 프로브용 헬스체크 |

## 부록 B. 배치 파이프라인 순서

```
[Scheduler 트리거]
   ↓
F1 콘텐츠 수집 (카테고리별 순차 또는 병렬)
   ↓
F2 임베딩 색인 (변경분만)
   ↓
F3 오늘의 추천 코스 생성 (테마별)
   ↓
[완료 — ingestion_runs 로그 기록]
```

F1 실패가 F2/F3를 막지 않도록, 각 단계는 "가능한 만큼 처리하고 계속 진행"하는 방어적 설계를 기본으로 한다.

## 부록 C. 공통 오류 응답 형식

모든 API는 실패 시 아래 형식으로 통일한다 (스프링 `@ControllerAdvice`로 공통 처리):

```json
{
  "code": "RECOMMENDATION_NOT_FOUND",
  "message": "오늘 생성된 추천 코스가 없습니다.",
  "timestamp": "2026-08-24T05:10:00+09:00"
}
```

| HTTP 상태 | 사용 예 |
|---|---|
| 400 | 잘못된 요청 파라미터 (예: 존재하지 않는 category 값) |
| 404 | 리소스 없음 (F4 폴백 캐시조차 없을 때) |
| 429 | F5 Rate Limit 초과 |
| 503 | LLM/벡터스토어/DB 등 외부 의존성 장애 |
