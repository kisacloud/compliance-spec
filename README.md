# Compliance Specification Registry (`compliance-spec`)

본 저장소는 다양한 정보보호 규제(CSAP, ISMS-P 등)의 통제 항목을 기계 가독형(Machine-Readable) 정형 데이터로 통합 관리하기 위한 마스터 레지스트리입니다.

이 플랫폼은 현재의 자동 판단 시스템을 구축하는 것에 앞서, **미래의 신뢰 가능한 AI 및 자동화 검증 엔진이 환각(Hallucination) 없이 판단을 내릴 수 있는 결정론적 지면(Ground Truth)의 제공**을 목적으로 설계되었습니다.

---

## 1. 아키텍처 설계 대원칙

본 저장소의 모든 데이터 구조는 다음 4가지 공학적 원칙을 절대 제약 조건으로 따릅니다.

1. **기계 중심 설계 (Machine-First):** 인간의 가독성이나 입력 편의성보다 파싱의 정밀함, 변환의 결정론(Deterministic), 외부 표준(OSCAL 등) 어댑터 구동의 무손실성(Round-trip)을 최우선으로 둡니다.
2. **불변 스냅샷 (Immutability):** 발효된 규제 본체 데이터는 과거 이력과 혼재되지 않는 순수 정적 스냅샷으로 보관됩니다. 개정에 따른 시계열 변동은 전수 차분 파일(`changelog`)로 완벽히 분리 격리됩니다.
3. **출처성 층위화 (Data Provenance):** 정성적 연관 관계 데이터는 신뢰 등급(`OFFICIAL_NOTICE`, `EXPERT_REVIEWED`, `AI_SUGGESTED`)과 검증 상태를 스키마 레벨에서 분리 적재하여, 미검증 AI 추론이 참값으로 오염되는 것을 차단합니다.
4. **느슨한 결합 (Loose Coupling):** 개별 규제 통제 데이터는 상호 간의 존재를 모른 채 독립 가동됩니다. 제도 간 상호 인정 관계는 외부 레이어(`correlations/`)에서 플러그인 형태로 조립됩니다.

---

## 2. 디렉터리 토폴도지 (Directory Topology)

저장소 내부의 물리적 배치는 관할권 격리 및 도메인 응집도 향상을 위해 아래의 트리 구조로 엄격히 통제됩니다. 파일명 명명 시 논리 ID 내의 콜론(`:`)은 도트(`.`)로 치환하며, 개정 버전 간의 연결은 언더바 2개(`__`)를 사용합니다.

```text
compliance-spec/
├── schemas/                                   # [1층] 기계적 규칙을 통제하는 JSON 스키마 레이어
│   ├── schema.catalog.v1.json                 # 규제 정의 원장 스키마
│   ├── schema.control.v1.json                 # 정형 통제 항목 스키마
│   ├── schema.correlation.v1.json             # 제도 간 상호 인정 명세 스키마
│   └── schema.changelog.v1.json               # 단일 제도 시계열 변경로그 스키마
│
├── frameworks/                                # [2층] 국가 관할권별 독립 프레임워크 데이터 레이어
│   └── kr/                                    # 관할권 격리 폴더 (ISO 3166-1 소문자)
│       ├── csap/                              # 제도 도메인 단위 격리 폴더 (소문자)
│       │   ├── catalog.2026r00.json           # 2026년 판본 규제 정의 스냅샷
│       │   ├── catalog.2026r01.json           # 2026년 판본 규제 정의 스냅샷
│       │   ├── control.2026r00.json           # 2026년 판본 순수 통제 항목 배열 데이터
│       │   ├── control.2026r01.json           # 2026년 판본 순수 통제 항목 배열 데이터
│       │   ├── changelog.2026r00__2026r01.json # 2026년(R0)판 대비 2026년(R1)판 증적 효력 전이 규칙 원장
│       │   └── guidelines/                    # [3층] 정성 지식 (마크다운, 그림 구조 고립 폴더)
│       └── isms-p/
│           └── ...
│
└── correlations/                              # [4층] 공식 상호인정 선언 전용 독립 레이어
    └── correlation.kr.csap.2026r01__kr.isms-p.2026r00.json
```

---

## 3. 식별자(URN) 문법 규격

모든 객체의 `id`는 RFC 8141 표준에 기반한 **프라이빗 URN(Uniform Resource Name)** 체계를 따릅니다. 데이터 무결성을 위해 **전체 문자열은 완전 소문자**여야 하며, 정규식(`patternProperties`)을 통해 CI 단계에서 컴파일 타임에 제약됩니다.

### 가. 카탈로그 식별자 (`catalog_id`)
* **포맷:** `urn:grc:{region}:{framework_name}:{version}`
* **규격 정규식:** `^urn:grc:(?:[a-z]{2}|global):[a-z0-9-]+:[0-9]{4}r[0-9]{2}$`
* **인스턴스 예시:** `urn:grc:kr:csap:2026r01`

### 나. 통제 항목 식별자 (`control_id`)
* **포맷:** `{catalog_id}:control:{control_no}`
* **규격 정규식:** `^urn:grc:(?:[a-z]{2}|global):[a-z0-9-]+:[0-9]{4}r[0-9]{2}:control:[a-z0-9._-]+$`
* **인스턴스 예시:** `urn:grc:kr:csap:2026r01:control:1.1.1`

### 다. 참조 무결성(Referential Integrity) 제약
모든 통제 조항 개별 객체는 소속 파일의 최상위 `catalog_id`와 완벽히 동치되어야 하는 강한 무결성 룰을 가집니다.

$$control\_id \equiv catalog\_id + ":control:" + control\_no$$

구분자로 콜론(`:`)만을 활용하므로 외부 OSCAL 표준과의 상호 변환 시 별도의 분기 처리 없이 `split(":")` 연산만으로 결정론적 무손실 왕복(Round-trip) 파싱이 보장됩니다.

---

## 4. 데이터 컴포넌트 명세

### 가. `catalog` 데이터
규제 제도의 발효일, 주관 부처, 계보상의 전임 카탈로그(`supersedes`) 정보를 담습니다. 제도명 및 기관명이 행정적으로 변경되는 단절 리스크를 최소화하기 위해 식별자 키는 제도명 중심(`csap`)으로 고정하며, 주관 부처 정보는 `"publisher": ["과기정통부", "kisa"]`와 같이 메타데이터 배열로 관리합니다.

### 나. `control` 데이터
부모 분류 조항(`parent_no`), 도메인, 제목, 단일 요구사항 평문(`statement`)만을 담아 원자성을 지킵니다. 표준 이외의 가변 필드 대응을 위해 데이터 하단에 공식 행정용인 `framework_meta`와 사업자 커스텀용인 `tenant_meta`로 분리된 개방형 확장 포트(`extensions`)를 강제 내장합니다.

### 다. `correlation` 데이터
제도 간의 상호 인정 관계 및 면제 조항 팩트를 선언합니다. 미래 AI의 추론 무결성을 위해 매핑 유형(`mapping_type`) 외에 출처 종류(`provenance`)와 검증 상태(`verification_status`)를 개별 행(Row)마다 전수 매립하여 가동 엔진이 신뢰 등급별로 데이터를 분기 연산할 수 있게 합니다.

### 라. `changelog` 데이터
단일 제도의 버전 업 시 발생하는 변동 차분 로그입니다. 단순히 상태 코드를 기록하는 것에 그치지 않고, 과거 구형 URN으로 영속 저장된 레거시 자동화 증적(Evidence Logs)을 신버전 런타임 심사 시 어떻게 법적으로 인정할 것인가에 대한 변경 관리 규칙(`evidence_transition_rule`)을 명시하여 시스템의 감사 추적성(Audit Trail) 단절을 방어합니다.

---

## 5. 기여 가이드 및 데이터 인입 유효성 검증

본 저장소의 모든 데이터 변경(Pull Request)은 기계적 유효성 검사 통과를 전제로 합니다.

1. **스키마 검증:** 인입되는 모든 JSON 파일은 `schemas/` 하위의 v1 마스터 설계도 규칙을 100% 만족해야 합니다.
2. **소문자 강제:** URN 데이터 필드 내에 단 1글자의 대문자나 비표준 구분자(`#`)가 발견될 경우 CI 빌드가 즉각 기각됩니다.
3. **참조 무결성 검사:** `control` 내에 기재된 `catalog_ref` 주소가 실제 동일 경로상의 `catalog.json` 내 `catalog_id`와 실재 동치하는지 교차 룩업(Lookup) 검증이 실행됩니다.
4. **로컬 가용성 지원:** 개발자의 경직성 검증 실패 마찰을 줄이기 위해, 커밋 실행 전 알파벳 대문자를 소문자로 자동 치환 및 정형화 처리를 선제 집행해 주는 헬퍼 스크립트 파일이 `/tools/pre-commit-hook.sh`에 패키징되어 있으므로 로컬 환경에 적용 후 작업을 권장합니다.