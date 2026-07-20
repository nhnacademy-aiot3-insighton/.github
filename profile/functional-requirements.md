# InsightOn 기능/비기능 요구사항 명세서

> 작성일: 2026-07-20
> 목적: Organization README의 요구사항 표가 "구체적이지 않다"는 팀장 피드백에 따라, 기존 요구사항을 검증 가능한 단위로 분해했다. 각 항목은 ID/상세 설명/수용 기준/우선순위/담당/의존성을 갖는다.
> 우선순위 기준: **Must**(MVP 필수, 없으면 서비스 성립 안 됨) / **Should**(MVP에 포함하되 지연 가능) / **Could**(여유 있으면 포함, 없어도 서비스 성립).
> `TBD` 표기 항목은 임의로 확정하지 않았다 — 팀 논의 후 수치를 채워야 한다.

---

## 1. 기능 요구사항

### 1.1 실시간 관제 (담당: CORE)

| ID | 요구사항 | 상세 설명 / 수용 기준 | 우선순위 | 의존성 |
|---|---|---|---|---|
| FR-01 | 공간별 센서 실시간 차트 | CO2/온도/습도 각각 개별 차트로 표시. 신규 패킷 수신 후 화면 반영까지 지연 시간 `TBD`(비기능 NFR-01과 연동) 이내. 차트 축은 최근 `TBD`분/시간 슬라이딩 윈도우 | Must | Core MQTT 수집 파이프라인, InfluxDB 조회 API |
| FR-02 | 대시보드 위젯 커스터마이징 | 위젯 추가/삭제/드래그 배치/크기 조정이 가능하고 새로고침 후에도 배치가 유지(영속화)되어야 함. 위젯 종류: 시계열 차트, 현재값 카드, 기기 상태, 알람 피드 (최소 세트, 확장 가능) | Should | `dashboards`/`widgets` 엔티티 |
| FR-03 | 기기 제어(수동) | 대시보드에서 액추에이터 ON/OFF 및 모드 제어. 명령 전송 후 `simulator_run_logs`에 실행 주체 `USER`로 기록되고, 실제 상태 반영을 대시보드에서 확인 가능해야 함 | Must | Core 제어 API, `device_attributes.current_value_str` |

### 1.2 인프라/기기 관리 (담당: INFRA/CORE)

| ID | 요구사항 | 상세 설명 / 수용 기준 | 우선순위 | 의존성 |
|---|---|---|---|---|
| FR-04 | 게이트웨이/기기 상태 조회 | `gateways.status`(NORMAL/FAULT), `last_heartbeat_at` 기준 목록 조회. 하트비트 임계 시간(`TBD`, 예: 5분) 초과 시 FAULT 자동 전환 | Must | 게이트웨이 헬스체크 스케줄러(ShedLock) |
| FR-05 | 신규 기기 자동 인식 | 미등록 기기의 첫 패킷 수신 시 `devices`/`device_attributes` 자동 생성, 이후 즉시 대시보드에 노출 | Should | Core Auto-Provisioning 로직 |

### 1.3 규칙 기반 자동화 (담당: ENGINE)

| ID | 요구사항 | 상세 설명 / 수용 기준 | 우선순위 | 의존성 |
|---|---|---|---|---|
| FR-06 | 노코드 플로우 빌더 | 트리거 1개 → 필터 0개 이상(직렬 체이닝=AND) → 액션 1개 이상(병렬 가능)을 드래그앤드롭으로 구성. 저장 시 `flows`/`nodes`/`links`로 영속화, `is_active` 토글로 즉시 활성/비활성 | Must | Rule Engine 엔티티 계층 |
| FR-07 | 센서 임계치 트리거 | `SENSOR_TRIGGER` — 특정 공간(선택적으로 특정 기기/메트릭)의 텔레메트리 도착 시 발화. RabbitMQ `telemetry.#` 소비 후 인메모리 인덱스로 O(1) 조회 | Must | Core→RabbitMQ 발행 파이프라인 |
| FR-08 | 스케줄 트리거 | `SCHEDULE_TRIGGER` — cron 표현식 기반 실행(예: 출근 시간 예열). 텔레메트리 이벤트 없이도 다운스트림 실행 | Should | Rule Engine 자체 스케줄러 |
| FR-09 | 조건 필터 | `THRESHOLD_FILTER`(SpEL 조건식), `TIME_WINDOW_FILTER`(시간대 제한) | Must | SpEL 평가기 |
| FR-10 | 반복 제어/알림 빈도 제한 | `TIMER_FILTER` — 지정 주기(`interval_seconds`)당 최초 1회만 통과, 그 사이 도착 메시지는 드롭(몰아서 처리 안 함). 상태는 `(node_uuid, locations_id/devices_id)` 조합별 독립, 인메모리(재시작 시 초기화는 의도된 동작) | Should | 단일 인스턴스 전제 — **다중 인스턴스 확장 시 Redis 공유 상태로 이전 필요(현재 스코프 아님, 별도 이슈로 추적)** |
| FR-11 | 규칙 기반 기기 제어 | `DEVICE_CONTROL_ACTION` — AI 개입 없이 Core 제어 API 직접 호출. `simulator_run_logs.executed_by_type='RULE_ENGINE'`로 기록 | Must | Core 제어 API |
| FR-12 | 즉시 알림 | `ALERT_ACTION` — LLM 없이 즉시 `dashboard_alerts` 생성 | Must | AI 서비스 알람 생성 API |
| FR-13 | AI 위임 | `AI_SUGGESTION_ACTION` — 실행 컨텍스트를 AI 서비스로 이벤트 발행 | Must | AI 서비스 이벤트 리스너 |
| FR-14 | 외부 채널 알림 | `EXTERNAL_NOTIFICATION_ACTION` — Telegram/이메일. 발송은 베스트 에포트(실패 시 재시도/영구 기록 없음, 확정 사항) | Should | Telegram Bot API / SMTP 연동 |

### 1.4 AI 지능형 제어 (담당: CORE, AI, API)

| ID | 요구사항 | 상세 설명 / 수용 기준 | 우선순위 | 의존성 |
|---|---|---|---|---|
| FR-15 | 공간별 운영 모드 선택 | `locations.auto_control_mode` = `SUGGESTION` \| `AI_DIRECT` 토글 | Must | Core `locations` 스키마 보정 |
| FR-16 | 상황 인지형 제안 생성 | Core 시간별 통계 + 기상청 현재/1시간 예보/미세먼지를 결합해 LLM이 인과관계 있는 자연어 가이드 생성 | Must | Issue 07(프롬프트 조립), Issue 11(기상청 캐시) |
| FR-17 | SUGGESTION 모드 수락 흐름 | 제안 생성 시 `is_accepted=null` 저장 + 대시보드 배너 노출 → 유저 수락 클릭 시 `action_payload` 기반 Core 제어 API 호출 및 `is_accepted=true` 갱신 | Must | `suggestion_logs`, Issue 08 |
| FR-18 | AI_DIRECT 모드 즉시 집행 | 생성과 동시에 `is_accepted=true` 자동 적재 + 유저 확인 없이 즉시 Core 제어 API 호출 | Must | 동일 |

### 1.5 알림 (담당: CORE, ENGINE)

| ID | 요구사항 | 상세 설명 / 수용 기준 | 우선순위 | 의존성 |
|---|---|---|---|---|
| FR-19 | 대시보드 실시간 알람 표시 | 신규 `dashboard_alerts` 생성 시 대시보드에 즉시 반영 | Must | **전송 방식(WebSocket/SSE/폴링) 미확정 — Issue 05, 팀 확인 필요** |
| FR-20 | Telegram/이메일 알림 | FR-14와 동일 채널 — 알람 발생 시 외부 채널로도 발송 | Could | FR-14 |

### 1.6 리포트 (담당: CORE→AI 정정 필요)

> README 표에는 리포트 담당이 "CORE"로 되어 있으나, `smart-office-iot-architecture-spec.md` 4장 기준 `hourly_telemetry_stats`/`reports`는 AI 서비스 소유다. README 담당 컬럼 오탈자로 보이며, 최종 조립 시 AI로 정정했다.

| ID | 요구사항 | 상세 설명 / 수용 기준 | 우선순위 | 의존성 |
|---|---|---|---|---|
| FR-21 | 정각 통계 집계 | 매시 정각 InfluxDB 원시 데이터로 평균/최고/최저 산출 + 액추에이터별 가동 분(JSONB) 산출. 데이터 없는 location은 행 미생성(의도된 동작) | Must | Issue 01, 02 |
| FR-22 | 주간/월간 리포트 | `hourly_telemetry_stats` 결산 데이터를 컨텍스트로 LLM이 마크다운 리포트 본문 생성 | Should | Issue 09, 10 |

### 1.7 이력/감사 (담당: CORE)

| ID | 요구사항 | 상세 설명 / 수용 기준 | 우선순위 | 의존성 |
|---|---|---|---|---|
| FR-23 | 실행 주체별 제어 이력 조회 | `executed_by_type`(`USER`/`AI_SYSTEM`/`RULE_ENGINE`) 기준 필터 조회 | Should | `simulator_run_logs` 스키마 보정(3.5절) |

### 1.8 조직/계정 관리 (담당: AUTH)

| ID | 요구사항 | 상세 설명 / 수용 기준 | 우선순위 | 의존성 |
|---|---|---|---|---|
| FR-24 | 초대 토큰 셀프 가입 | 토큰 입력 가입 시 항상 `group_role='MEMBER'`로 고정. 관리자는 토큰 재발급(로테이션) 가능, 재발급 즉시 이전 토큰 무효화 | Must | Auth↔Core 내부 API(`/internal/groups/join-by-token`) |
| FR-25 | 역할 기반 접근 제어 | OWNER/ADMIN/MEMBER 3단계 최소 구현, 화면/API별 접근 권한 매트릭스는 별도 문서화 필요(`TBD`) | Must | Auth 권한 모델 |
| FR-26 | 이메일/소셜 로그인 | 이메일+비밀번호, OAuth(제공자 `TBD` — 예: Google) | Must | Auth `users_credentials`/`oauth` |

---

## 2. 비기능 요구사항

| ID | 구분 | 요구사항 | 수용 기준 / 측정 방법 | 우선순위 |
|---|---|---|---|---|
| NFR-01 | 실시간성 | 패킷 수신~대시보드 반영 지연 | 목표 지연 시간 `TBD`(팀 확인 필요, 예: p95 2초 이내 제안) — 수집 스레드는 논블로킹 즉시 반환, 저장/전파는 비동기 분리 | Must |
| NFR-02 | 장애 격리 | InfluxDB 장애가 실시간 수집을 막지 않음 | Fail-Silent 구조 — Influx 쓰기 실패가 MQTT 리스너/RabbitMQ 발행 경로에 예외 전파되지 않음을 통합 테스트로 검증 | Must |
| NFR-03 | 멀티테넌시 | 그룹 간 데이터 물리적 격리 | 서비스별 독립 DB, 그룹 A의 쿼리가 그룹 B 데이터에 도달 불가함을 테스트로 검증 | Must |
| NFR-04 | 확장성 | 신규 센서/액추에이터 종류 추가 시 스키마 변경 불필요 | `metrics_avg`/`actuator_on_minutes` 등 JSONB 컬럼 사용, 신규 종류 추가가 배포 없이 가능한지 확인 | Should |
| NFR-05 | 확장성 | 멀티 인스턴스 환경에서 배치 중복 실행 방지 | 정각 통계/주간·월간 리포트/기상청 캐시 갱신 배치에 ShedLock 적용, 2개 이상 인스턴스 동시 기동 시 1회만 실행되는지 검증 | Must |
| NFR-06 | 비용 효율 | LLM 호출은 이상 상황 발생 시에만 트리거 | 정기 배치성 LLM 호출 없음(리포트 배치 제외) | Should |
| NFR-07 | 성능 | 반복 조회 매핑 정보 캐싱 | Core `group_id`/`location_id`, Rule Engine `locations_id→flows` 매핑에 인메모리 캐시(Caffeine 등) 적용 | Should |
| NFR-08 | 보안 | 유출된 초대 토큰 즉시 무효화 | 재발급 즉시 이전 토큰으로 가입 시도 시 실패 응답 확인 | Must |
| NFR-09 | 데이터 무결성 | 동일 물리 DB 내 실제 FK 제약 | 각 서비스 ERD 갭 표(3.5/4.5/5장 참고) 반영한 DDL에 FK 제약이 실제로 걸려 있는지 확인 | Must |
| NFR-10 | 오버 엔지니어링 회피 | 불필요한 이력 저장/과도한 트랜잭션 보장 지양 | 게이트웨이/기기 상태 이력 테이블 미생성, 알림 발송 로그 미보관 등 확정 사항 유지 | Should |

---

## 3. 미확정 항목 (임의 결정 금지 — 팀 논의 필요)

| 항목 | 관련 요구사항 | 비고 |
|---|---|---|
| 실시간 알람 푸시 방식 (WebSocket/SSE/폴링) | FR-19 | Issue 05 |
| 실시간성 목표 지연 시간(SLA) | NFR-01 | 구체적 수치 미정 |
| 게이트웨이 하트비트 임계 시간 | FR-04 | 구체적 수치 미정 |
| OAuth 제공자 목록 | FR-26 | 구체적 목록 미정 |
| 역할별 접근 권한 매트릭스 | FR-25 | 화면/API 단위 매핑표 별도 필요 |
