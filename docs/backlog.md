# 부산이 프로젝트 — 백로그

- 문서버전: v0.1
- 작성일: 2026-08-24
- 상위 문서: [requirements-spec.md](requirements-spec.md), [functional-spec.md](functional-spec.md)

요구사항 명세서(FR)와 기능 명세서(F)의 각 항목을 실제 작업 단위로 추적한다.
1인 개발이라 별도 이슈 트래커 없이 이 문서를 진행 상황판으로 사용하며,
코드 작업 착수 시점에 GitHub Issues로 옮겨도 이 문서의 ID 체계를 그대로 유지한다.

상태값: `todo` (미착수) · `doing` (진행 중) · `done` (완료) · `blocked` (외부 요인 대기)

## 착수 전 선행 작업 (코드 스캐폴딩)

| ID | 작업 | 상태 | 비고 |
|---|---|---|---|
| PRE-01 | `.gitignore` 추가 | todo | 코드 스캐폴딩 전에 우선 추가 |
| PRE-02 | data.go.kr 인증키 발급 신청 | todo | 승인 대기 시간 있을 수 있음 — 최우선 |
| PRE-03 | OpenAI API 키 발급 | todo | |
| PRE-04 | 저장소 구조 결정 (모노레포 vs 멀티레포) | todo | |
| PRE-05 | 백엔드 프로젝트 생성 (Spring Boot 3.x + Java 21 + Spring AI) | todo | |
| PRE-06 | 로컬 `docker-compose.yml` (postgres+pgvector, ollama) | todo | NFR-09 |
| PRE-07 | DB 마이그레이션 도구 도입 (Flyway/Liquibase) | todo | places/daily_recommendations/ingestion_runs DDL |
| PRE-08 | 프론트엔드 프로젝트 생성 (Vite + React + TS) | todo | |
| PRE-09 | Secret 관리 방식 확정 (.env / K8s Secret) | todo | NFR-05 |
| PRE-10 | CI 파이프라인 (GitHub Actions: build/test/lint) | todo | |

## 배치 파이프라인

| ID | 작업 | 매핑 | 우선순위 | 상태 |
|---|---|---|---|---|
| WORK-01 | OpenAPI 4종 수집 클라이언트 + 재시도 로직 | FR-01, FR-09, F1 | 필수 | todo |
| WORK-02 | 응답 정규화 + `places` upsert | FR-02, F1 | 필수 | todo |
| WORK-03 | `ingestion_runs` 로그 기록 | FR-09, NFR-08, F1 | 필수 | todo |
| WORK-04 | 임베딩 대상 선정 + `EmbeddingModel` 연동 | FR-03, F2 | 필수 | todo |
| WORK-05 | `VectorStore` 색인 (metadata: placeId/category/gugunNm) | FR-03, F2 | 필수 | todo |
| WORK-06 | 테마 3종 벡터 검색 + RAG 코스 생성 | FR-04, F3 | 필수 | todo |
| WORK-07 | `daily_recommendations` upsert + 폴백 로직 | FR-04, F3 | 필수 | todo |
| WORK-08 | ShedLock 도입 (배치 중복 실행 방지) | NFR-04, architecture.svg 참고 | 필수 | todo |

## 서빙 API

| ID | 작업 | 매핑 | 우선순위 | 상태 |
|---|---|---|---|---|
| WORK-09 | `GET /api/recommendations/today` (구군/테마 필터, 폴백) | FR-05, FR-06, F4 | 필수 | todo |
| WORK-10 | `POST /api/chat` (실시간 RAG) | FR-07, F5 | 필수 | todo |
| WORK-11 | `/api/chat` Rate Limiting (Bucket4j, IP당 분당 5회) | NFR-11, F5 | 필수 | todo |
| WORK-12 | `GET /api/places` (카테고리/구군 페이지네이션) | FR-08, F6 | 선택 | todo |
| WORK-13 | `GET /api/admin/ingestion-runs` | NFR-08, F8 | 선택 | todo |
| WORK-14 | `/actuator/health` 노출 확인 | NFR-10 | 필수 | todo |
| WORK-15 | 공통 오류 응답(`@ControllerAdvice`) | 부록 C | 필수 | todo |

## 프론트엔드

| ID | 작업 | 매핑 | 우선순위 | 상태 |
|---|---|---|---|---|
| WORK-16 | 홈 화면 (오늘의 추천 코스 카드 + 지도) | F7 | 필수 | todo |
| WORK-17 | 챗 화면 (질의 입력 + 답변/참조 장소) | F7 | 필수 | todo |
| WORK-18 | 장소 탐색 화면 (필터 + 상세 모달) | F7 | 선택 | todo |

## 인프라/배포

| ID | 작업 | 매핑 | 우선순위 | 상태 |
|---|---|---|---|---|
| WORK-19 | K8s Deployment/StatefulSet 매니페스트 (레이블링 전략 적용) | architecture 4.1~4.3 | 필수 | todo |
| WORK-20 | K8s Secret 구성 (data.go.kr 키, OpenAI 키) | NFR-05 | 필수 | todo |
| WORK-21 | 컨테이너 이미지 빌드 파이프라인 | NFR-09 | 필수 | todo |
