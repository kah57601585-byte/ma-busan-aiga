# 부산이 프로젝트 — 테스트 전략

- 문서버전: v0.1
- 작성일: 2026-08-24
- 상위 문서: [requirements-spec.md](requirements-spec.md), [functional-spec.md](functional-spec.md)

1인 개발 프로젝트 규모를 전제로, 과도한 테스트 인프라보다 "핵심 리스크를 검증하는 최소 구성"을 목표로 한다.

## 1. 테스트 레벨별 범위

| 레벨 | 대상 | 도구 | 비고 |
|---|---|---|---|
| 단위 테스트 | 정규화 로직, upsert 판단, 프롬프트 조립, DTO 매핑 | JUnit 5 + Mockito | 외부 API/LLM은 모두 목(mock) 처리 |
| 통합 테스트 | Repository(JPA), VectorStore 연동, `@Scheduled` 배치 플로우 | Spring Boot Test + Testcontainers(PostgreSQL+pgvector) | 실제 DB 스키마로 검증 — 로컬 H2 사용 금지(pgvector 미지원) |
| API 계약 테스트 | F4~F8 REST 엔드포인트 | `@SpringBootTest` + `MockMvc`/`WebTestClient` | 부록 C 공통 오류 응답 형식 준수 여부 포함 |
| E2E (선택) | 홈 화면 → 추천 조회 → 챗 질의 흐름 | Playwright | 프론트엔드 완성 이후, 필수는 아님 |

## 2. FR 수용 기준 → 테스트 매핑

requirements-spec.md 2.1절의 FR 수용 기준을 그대로 테스트 케이스로 변환한다. 예:

| FR | 테스트 방식 |
|---|---|
| FR-01 | 배치 통합 테스트에서 4개 카테고리 모두 `ingestion_runs`에 기록되는지 검증 |
| FR-02 | 동일 `(uc_seq, category, lang)`로 2회 연속 수집 실행 후 row 수 불변 검증 (idempotency 테스트) |
| FR-05 | API 테스트에서 응답 스키마(`courseSummary`, `places[]`) 검증 — 레이턴시는 4절 부하 테스트에서 별도 측정 |
| FR-09 | 3xx/5xx를 반환하는 목 서버로 재시도 횟수·`ingestion_runs.status=failed` 기록 검증 |
| FR-10 | CI에 시크릿 스캔(gitleaks 등) 단계 추가 — 하드코딩 키 커밋 자체를 차단 |

나머지 FR도 동일한 방식으로, 기능 구현 PR에 대응하는 테스트를 함께 추가한다(기능만 있고 테스트가 없는 PR은 지양).

## 3. 성능(NFR-01, NFR-02) 검증

| NFR | 목표 | 검증 방법 |
|---|---|---|
| NFR-01 | `/api/recommendations/today` P95 200ms 이내 | k6 스크립트로 캐시 조회 API에 동시 사용자 10~20 수준 부하 발생, P95 확인 |
| NFR-02 | `/api/chat` P95 5초 이내 | k6로 순차 호출(동시성 낮게 — LLM 비용 고려), 응답 시간 분포 확인. 목표 초과 시 스트리밍(SSE) 전환 검토 |

- 부하 테스트는 로컬 Ollama 기준과 운영 OpenAI 기준을 분리해서 측정한다(모델 특성이 다르므로 목표 재조정 가능).
- k6 스크립트는 `test/load/`에 보관하고, CI에서는 실행하지 않는다(비용/시간 문제) — 배포 전 수동 실행.

## 4. 테스트 커버리지 목표

- 배치 파이프라인(F1~F3)과 서빙 API(F4~F6)의 핵심 분기(정상/실패/폴백)는 필수로 커버.
- 커버리지 수치 목표는 별도로 강제하지 않음(1인 개발 규모에서 수치 강박보다 "핵심 리스크 커버"가 우선) — 단, PRE-10 CI에서 테스트 실패 시 머지 차단은 필수로 설정한다.

## 5. 실행 시점

| 시점 | 실행 항목 |
|---|---|
| PR마다 (CI) | 단위 + 통합 테스트, 린트, 시크릿 스캔 |
| 배포 전 (수동) | k6 부하 테스트, `/actuator/health` 확인 |
| 배치 로직 변경 시 | Testcontainers 기반 배치 통합 테스트 필수 재실행 |
