# Compliance Specification Registry (`compliance-spec`)

본 저장소는 정보보호 규제(CSAP, ISMS-P 등)의 통제 항목을 기계 가독형(Machine-Readable) 정형 데이터로 통합 관리하는 마스터 레지스트리입니다. 자동화 심사 및 AI 기반 검증 엔진이 오판 없이 일관된 연산을 수행하도록, 신뢰할 수 있는 기준 데이터(Machine-Readable Ground Truth)를 제공하는 것이 목적입니다.

이 문서는 사람과 AI 모두가 본 데이터 체계를 처음부터 이해할 수 있도록 작성된 기본 설명서입니다. 각 구조가 **무엇인지**뿐 아니라 **왜 그렇게 설계되었는지**, 그리고 **특정 상황에서 어떻게 데이터를 작성해야 하는지**를 함께 설명합니다.

---

## 1. 아키텍처 설계 대원칙

데이터 구조는 다음 5가지 공학 원칙에 기반합니다.

1. **기계 중심 설계 (Machine-First):** 사람의 입력 편의보다 파싱의 정밀함, 변환의 결정론(Deterministic), 외부 표준(OSCAL 등) 어댑터의 무손실 왕복(Round-trip)을 우선합니다.
2. **스냅샷 불변성 (Immutability):** 발효된 규제 데이터는 정적 스냅샷으로 동결됩니다. 개정이 일어나면 기존 파일을 고치지 않고 새 버전 스냅샷을 추가하며, 두 판본 사이의 변경은 별도 `changelog`로 잇습니다.
3. **출처성 층위화 (Data Provenance):** 제도 간 연관 관계 데이터는 신뢰 등급(`OFFICIAL_NOTICE`, `EXPERT_REVIEWED`, `AI_SUGGESTED`)과 검증 상태를 행마다 명시합니다. AI가 생성한 미검증 추론이 공식 근거와 뒤섞여 후속 판단의 참값으로 오용되는 것을 막기 위함입니다.
4. **느슨한 결합 (Loose Coupling):** 개별 통제 데이터는 다른 제도의 존재를 모른 채 독립적으로 존재합니다. 제도 간 상호 인정 관계는 오직 외부 레이어(`correlations/`)에서 플러그인처럼 조립됩니다.
5. **하이브리드 검증 (Hybrid Validation):** 스키마는 범용 구조만 강제하고, 가변적인 규제 속성의 허용 명사·필수 항목·일관성 규칙은 CI 단계(`framework-rules.json`)에서 통제합니다.

---

## 2. 디렉터리 토폴로지

물리적 배치는 제도(framework)별 폴더로 구분합니다. 다만 파일이 경로와 분리되어 단독 유통(diff, PR, 첨부, 검색)될 때 자기설명적이도록, **파일명에도 제도명과 버전을 표기**합니다(폴더가 이미 제공하는 정보를 의도적으로 반복). 파일명에서 URN의 콜론(`:`)은 도트(`.`)로 치환하며, 두 판본/제도를 잇는 결합 식별자는 언더바 2개(`__`)를 사용합니다.

```text
compliance-spec/
├── schemas/                                            # [1층] 기계적 규칙을 통제하는 JSON 스키마
│   ├── schema.catalog.v1.json
│   ├── schema.control.v1.json
│   ├── schema.guidance.v1.json                         # 해설(주요확인사항/세부설명/증거자료/결함사례) 레이어
│   ├── schema.correlation.v1.json
│   └── schema.changelog.v1.json
│
├── frameworks/                                         # [2층] 제도(framework)별 독립 데이터
│   ├── csap/
│   │   ├── catalog.csap.2026r1.json                   # 판본 메타데이터 스냅샷
│   │   ├── control.csap.2026r1.json                   # 규범(통제 항목) 데이터
│   │   ├── guidance.csap.2026r1.json                  # 해설 데이터 (control_id로 연결)
│   │   └── changelog.csap.2023r1__2026r1.json        # 판본 간 차분
│   └── isms-p/
│       ├── catalog.isms-p.2023r1.json
│       ├── control.isms-p.2023r1.json
│       └── guidance.isms-p.2023r1.json
│
├── correlations/                                       # [3층] 제도 간 상호 인정 선언
│   └── correlation.csap.2026r1__isms-p.2023r1.json
│
└── framework-rules.json                                # CI 2차 검증용 허용 명사·규칙 사전
```

> 파일명 규칙: `{타입}.{제도}.{버전}.json`. 폴더가 제도를 구분하더라도 파일명에 제도명·버전을 반복 표기하여, 파일 단독 유통 시 자기설명성을 확보합니다. 두 제도/판본을 잇는 파일(`correlation`, `changelog`)은 양쪽을 `__`로 표기합니다. 버전 숫자는 발효 스냅샷이 확정되어 여러 판본이 공존할 때 비로소 변별값을 가지며, 데이터 내부(`catalog_id`, `catalog_version`)의 버전과 일치시킵니다. `guidance`/`changelog`는 해당 데이터가 존재할 때만 둡니다.

데이터 인스턴스 파일에는 `$schema` 키를 넣지 않습니다. 모든 스키마가 `additionalProperties: false`이므로 인스턴스에 메타스키마 포인터를 넣으면 검증이 실패합니다. 인스턴스와 스키마의 연결은 CI 파이프라인이 파일 경로 규칙으로 수행합니다.

---

## 3. 식별자(URN) 문법 규격

모든 객체 식별자는 프라이빗 URN 체계를 따릅니다. 데이터 무결성을 위해 **전체 문자열은 완전 소문자**여야 하며, 정규식으로 CI 빌드 시 강제됩니다.

### 가. 카탈로그 식별자 (`catalog_id`)
* **포맷:** `urn:grc:{region}:{framework}:{version}`
* **정규식:** `^urn:grc:(?:[a-z]{2}|global):[a-z0-9-]+:[0-9]{4}r[0-9]{2}$`
* **예시:** `urn:grc:kr:csap:2026r1`
* **version 세그먼트**(`2026r1`)는 `{연도 4자리}r{차수}` 형식입니다. 차수는 zero-padding 없이 1부터 자연수로 표기하며(예: `2026r1`, `2026r2`, `2026r10`), `r2`처럼 자연스럽게 참조할 수 있습니다. 차수는 연 단위로 리셋됩니다. **버전 시간순 비교는 문자열 정렬이 아니라 차수를 정수로 파싱하여 수행합니다**(`r` 뒤를 정수로 해석). 따라서 `2026r2 < 2026r10`이 올바르게 판정되며, 자리수 고정에 따른 상한이나 참조 시 패딩 부담이 없습니다.

### 나. 통제 항목 식별자 (`control_id`)
* **포맷:** `{catalog_id}:control:{control_no}`
* **정규식:** `^urn:grc:(?:[a-z]{2}|global):[a-z0-9-]+:[0-9]{4}r[0-9]{2}:control:[a-z0-9._-]+$`
* **예시:** `urn:grc:kr:csap:2026r1:control:1.1.1`

### 다. 참조 무결성 제약
모든 통제·매핑·차분 객체의 `control_id`는 다음을 정확히 만족해야 합니다.

```
control_id ≡ catalog_id + ":control:" + control_no
```

구분자로 콜론(`:`)만 쓰므로 `split(":")` 연산만으로 결정론적 무손실 왕복 파싱이 가능합니다. 이는 OSCAL 등 외부 표준 어댑터가 분기 처리 없이 변환할 수 있게 하기 위한 핵심 제약입니다.

> **식별자 인코딩 철학:** 통제 ID에 의미(제도·버전·번호)를 인코딩하는 것은 의도된 설계입니다. 규제 통제는 고시 원문·감사 증적·법령에서 그 번호로 호명되므로, 불투명 UUID로 대체하면 원문 추적성이 끊깁니다. 이는 NIST SP 800-53(`ac-2` 등)을 비롯한 국제 컴플라이언스 표준의 관행과 정렬됩니다.

### 라. 결합 식별자(`correlation_id`, `changelog_id`)
두 카탈로그 URN을 `__`로 이은 문자열(예: `...csap:2026r1__kr:isms-p:2023r1`)은 **파일을 식별하는 불투명 라벨**일 뿐 파싱 대상이 아닙니다. 실제 관계는 같은 파일의 원자 필드(`source_catalog_id` / `target_catalog_id`)가 담습니다. 따라서 이 라벨은 유일성만 보장하면 됩니다.

---

## 4. 데이터 컴포넌트 명세

### 가. `catalog` — 제도 판본 메타데이터
한 제도의 특정 판본을 나타내는 불변 스냅샷입니다. 발효일, 주관 부처, 직전 판본(`supersedes`)을 담으며, 법적 뿌리를 보장하기 위해 근거 법령 배열 `governing_law`를 최상위 필드로 강제합니다(법적 강제 규제가 아니면 `null`). 총 통제 수·분야 수 같은 집계값은 `control` 데이터에서 파생 가능하므로 본문에 중복 저장하지 않으며, 부득이 `framework_meta`에 둘 경우 CI가 실제 데이터와의 일치를 강제합니다.

### 나. `control` — 통제 항목 (핵심 컴포넌트)

모든 통제는 **트리 중첩이 아니라 단층 평탄 배열**로 적재됩니다. 평탄 구조는 AI가 조인 없이 행 단위로 맥락을 완결할 수 있게 하고, 외부 표준 변환을 단순하게 만듭니다. 평탄화로 인해 표면상 사라지는 정보(분류 계층, 구조 분할, 등급 변형)는 아래 세 개의 **직교(orthogonal)하는 축**으로 명시 복원합니다. 이 세 축을 혼동하지 않는 것이 본 데이터 모델 이해의 핵심입니다.

#### 축 A — 분류 계층 (어디에 속하는가)

규제 해설서의 분류 체계(영역 → 분야 → 통제)는 OSCAL의 `group` 중첩에 해당합니다. 평탄 배열에서는 이를 각 통제의 **반정규화 라벨 필드**로 복원합니다.

* `domain_no` / `domain_title` — 최상위 대분류(영역). 예: `1` / `관리체계 수립 및 운영`
* `group_no` / `group_title` — 중간 분류(분야). 예: `1.1` / `관리체계 기반 마련`

> **중요:** `domain_no`와 `group_no`는 **분류 라벨이지 상위 통제에 대한 참조가 아닙니다.** 따라서 `control_id` 참조 무결성 검사에서 제외됩니다(`group_no`가 가리키는 `1.1`은 통제가 아니라 분야 묶음입니다). 번호는 `control_no`에서 파생 가능하지만 **명칭(title)은 파생 불가**하므로 명칭 반정규화가 이 필드들의 실질적 가치입니다. 반정규화의 약점인 표기 드리프트는 "같은 번호 → 항상 같은 명칭" CI 규칙으로 방어합니다. `group_no`는 향후 분류 계층을 별도 블록으로 정규화할 때 외래 키(FK) 역할을 할 수 있는 안정 키입니다.

#### 축 B — 구조 분할 (한 조항이 여러 확인점인가)

한국 규제 통제는 원자 단위가 아닌 경우가 많습니다. 한 고시 조항이 "A를 하고 **그리고** B를 해야 한다"처럼 여러 독립 확인점을 담으면, 한 문장으로 Pass/Fail을 판정할 수 없습니다. 이때 행을 나누어 각 `statement`를 원자화합니다.

* `base_no` — 행정 고시 원문의 공식 조항 번호. 분할된 형제들을 하나로 묶는 그룹핑 키.
* `split_type: "COMPOSITE"` — 이 행이 구조 분할된 형제 중 하나임을 표시.
* `control_no` — `base_no`에 접미사를 붙여 구분 (예: `1.1.1_a`, `1.1.1_b`).

**작성 예시 (구조 분할):**
```json
{ "control_no": "1.1.1_a", "base_no": "1.1.1", "split_type": "COMPOSITE",
  "overlay_of": null, "statement": "보고 체계를 수립하여야 한다." }
{ "control_no": "1.1.1_b", "base_no": "1.1.1", "split_type": "COMPOSITE",
  "overlay_of": null, "statement": "수립한 보고 체계를 운영하여야 한다." }
```
같은 `base_no`를 가진 `COMPOSITE` 형제들은 **AND로 함께 평가**됩니다(OSCAL의 control 내 다중 `part`에 대응). 분할이 필요 없는 일반 통제는 `split_type: "ATOMIC"`이며 `base_no == control_no`입니다.

#### 축 C — 등급 변형 / 오버레이 (같은 조항이 등급별로 내용이 다른가)

같은 조항이 인증 등급·적용 범위에 따라 **요구사항 문안 자체가 달라지는** 경우가 있습니다(예: 상위 등급은 "물리적으로 분리", 하위 등급은 "물리적 또는 논리적으로 분리"). 이는 적용 여부 on/off가 아니라 내용 변형이므로, 별도 행으로 분리하고 변형 관계를 명시합니다.

* `base_no` — 동일 조항임을 나타내는 공유 키.
* `split_type: "OVERLAY"` — 이 행이 등급 변형임을 표시.
* `overlay_of` — 이 변형이 파생된 원본 통제의 `control_id`(명시적 참조).
* 적용 등급은 `framework_meta`의 등급 플래그(`level_low: true` 등)로 표기.

**작성 예시 (등급 변형):**
```json
{ "control_no": "12.3.1", "base_no": "12.3.1", "split_type": "ATOMIC",
  "overlay_of": null, "statement": "...암호화 관련 법적 요구사항을 반드시 반영하여야 한다." }
{ "control_no": "12.3.1_low", "base_no": "12.3.1", "split_type": "OVERLAY",
  "overlay_of": "urn:grc:kr:csap:2026r1:control:12.3.1",
  "statement": "...정책을 마련하여야 한다.",
  "extensions": { "framework_meta": [ { "name": "level_low", "value": "true" } ], "tenant_meta": [] } }
```

> **왜 B와 C를 분리하는가:** 구조 분할(B)과 등급 변형(C)은 완전히 다른 차원입니다. 하나의 통제가 "여러 확인점으로 쪼개지면서(B) 그중 일부가 등급별로 또 달라지는(C)" 복합 상황이 가능하기 때문에, 두 개념을 한 필드에 섞으면 표현이 불가능해집니다. 따라서 등급 변형은 `split_type=OVERLAY` 축으로, 구조 분할은 `framework_meta` 플래그로 분리하여 직교성을 유지합니다.

#### 다차원 적용 여부 — 직교 boolean 플래그

인증제도 멤버십, 보안 등급, 서비스 유형 등 "이 통제가 어디에 적용되는가"는 기계적 이진 판별이 가능하도록 `framework_meta` 내 **직교 boolean 플래그**(value는 `"true"`/`"false"`)로 적재합니다. 이는 새 제도·등급이 생겨도 스키마 변경 없이 데이터 추가만으로 확장됩니다(허용 명사를 `framework-rules.json`에 등재).

* 인증제도 멤버십: `applies_to_isms`, `applies_to_isms-p`, `applies_to_simplified`

> **플래그 이름 표기 규약:** 멤버십 플래그의 제도 식별 부분은 `catalog_id`의 framework 세그먼트와 표기를 일치시킵니다(예: `urn:grc:kr:isms-p` ↔ `applies_to_isms-p`). 따라서 하이픈을 포함할 수 있습니다. 이 `name` 값들은 **문자열 동등 비교로만** 사용하며, 코드의 변수명·속성 식별자로 직접 승격하지 않습니다(하이픈은 다수 언어에서 변수명에 쓸 수 없기 때문입니다).
* 보안 등급: `level_low`, `level_standard`, `level_simple`
* 서비스 유형: `service_iaas`, `service_paas`, `service_saas`, `service_daas`

> **활용 예 — ISMS와 ISMS-P:** 두 제도는 별도 카탈로그가 아니라 **하나의 통제 풀(`urn:grc:kr:isms-p`)에 대한 적용 범위 차이**입니다. 제도의 본질은 ISMS가 기본 모집단이고, **ISMS-P는 여기에 개인정보 처리 영역 통제를 더해 확장한 집합**입니다(역사적으로도 ISMS가 먼저 운영되다 개인정보 영역이 추가되어 ISMS-P가 되었습니다). 통제 본문(statement)은 양쪽이 동일하고 포함 여부만 다르므로, 통제를 복제하거나 카탈로그를 분리하지 않고 **단일 통제 풀에 멤버십 플래그**로 표현합니다. 1·2영역 통제는 `applies_to_isms: true` + `applies_to_isms-p: true`, 개인정보 처리(3영역) 통제는 `applies_to_isms: false` + `applies_to_isms-p: true`로 둡니다 — 개인정보 통제가 `applies_to_isms: false`라는 사실 자체가 "ISMS에는 없고 ISMS-P에서 추가된 확장"이라는 방향을 데이터로 증명합니다. 'ISMS 80개'와 'ISMS-P 101개'는 각각 별도 파일이 아니라 이 플래그로 필터링하여 소비 시점에 얻습니다. ISMS 간편인증처럼 부분집합을 적용하는 제도도 같은 방식(`applies_to_simplified`)으로 확장됩니다.

#### `extensions` 레이어
* `framework_meta` — 정부 고시·기관 지정에서 비롯된 행정/적용 속성(위 플래그 포함). 허용 명사는 `framework-rules.json`이 통제. **해설성 자료는 여기에 넣지 않습니다**(아래 규범/해설 분리 참조).
* `tenant_meta` — 기준 데이터가 아닌, 개별 도입 기업의 운영/인프라 매핑용 임의 속성. 공식 기준값에는 비워 둡니다(`[]`).

#### 규범(norm)과 해설(guidance)의 분리

`control`은 **규범만** 담습니다 — 판정 대상인 `statement`와 식별·분류·적용 메타(플래그)뿐입니다. 안내서가 제공하는 해설성 자료(주요 확인사항, 세부 설명, 증거자료 예시, 결함사례)는 **별도 `guidance` 레이어**에 두고 `control_id`로 연결합니다.

분리하는 이유는 셋입니다. (1) **기계 가독성** — 자동 판정·매핑은 `statement`만 필요하므로, 수십 배 무거운 해설을 함께 로드하지 않도록 분리합니다. (2) **개정 추적의 순수성** — `statement`(규범)는 그대로인데 해설만 다듬어지는 변경을 통제 변경과 구분하기 위함입니다. (3) **출처 분리** — 규범(고시 원문)과 해설(안내서 설명, 향후 AI 보충 등)은 신뢰 출처가 다르므로 섞지 않습니다. 이는 `correlation`/`changelog`가 그렇듯 `control_id`를 외래 키로 참조하는 plug-in 레이어이며, '느슨한 결합' 원칙을 따릅니다.

### 다. `guidance` — 통제 해설 (control_id로 연결)
한 통제에 대한 안내서의 해설을 담는 레이어입니다. 각 행은 `control_id`(어느 통제에), `guidance_type`(checklist=주요 확인사항 / detail=세부 설명 / evidence=증거자료 / defect=결함사례 중 하나), `content_md`(해설 본문), `provenance`(출처 등급)로 구성됩니다. 본문은 **Markdown**(`content_md`)으로 보존하여 안내서 원문의 문단·불릿·중첩 위계를 그대로 유지합니다 — 항목을 인위적으로 배열 분해하지 않으므로 구조 손실과 분리 검수 부담이 없고, 사람이 보는 제3 도구는 Markdown 렌더러로, AI는 Markdown 구조 그대로 활용할 수 있습니다. 한 통제는 여러 `guidance` 행(타입별)을 가질 수 있습니다(1:N).

### 라. `correlation` — 제도 간 상호 인정
서로 다른 두 제도 사이의 동등·포함·연관 관계를 선언합니다. 통제 데이터 자체는 다른 제도를 모르며(느슨한 결합), 관계는 오직 이 레이어에서 조립됩니다. 매핑 유형(`mapping_type`: EQUIVALENT/SUPERSET/SUBSET/PARTIAL/RELATED/NO_EQUIVALENT)은 NIST OLIR 등 국제 매핑 어휘와 정렬됩니다. 미래 AI의 추론 무결성을 위해 출처(`provenance`)와 검증 상태(`verification_status`)를 매핑 행마다 전수 기재합니다. 통제 참조 시 반드시 `:control:` 네임스페이스를 포함해야 합니다.

### 마. `changelog` — 판본 간 차분
한 제도의 두 판본 사이 통제 단위 변경(`UNCHANGED`/`CHANGED`/`NEW`/`DEPRECATED`)을 기록합니다. 핵심은 단순 상태 기록을 넘어, 구버전 기준으로 축적된 자동화 증적을 신버전 심사에서 어떻게 인정할지를 `evidence_transition_rule`로 선언하여 감사 추적성 단절을 방어하는 것입니다.

> **활용 예 — 고위험 통제 추가 등 개정:** 법 개정으로 고위험 대상에 통제가 신설되면, 기존 판본을 수정하지 않고 새 버전 카탈로그를 만든 뒤(`supersedes`로 직전 판본 지정) changelog로 잇습니다. 신설 통제는 `status_code: NEW`, 기존 통제의 문안이 강화되면 `CHANGED`로 기록하고, 각 변경의 `evidence_transition_rule`로 구증적 승계 여부를 명시합니다. 개정 내용이 "기존 통제를 특정 대상에 한해 의무화"하는 수준이면 통제 신설 없이 적용 플래그(축 C/플래그) 추가만으로 수용될 수도 있습니다 — 실제 고시 문안에 따라 세 축 중 적합한 표현을 선택합니다.

---

## 5. 기여 가이드 및 데이터 인입 유효성 검증

모든 데이터 변경(Pull Request)은 CI 파이프라인의 2중 기계 검증을 통과해야 합니다.

1. **1차 스키마 검증 (구조 제어):** 인입되는 모든 JSON은 `schemas/`의 v1 마스터 규칙을 100% 만족해야 합니다. `framework_meta`/`tenant_meta` 등 확장 필드는 반드시 중립 `name-value` 배열 구조여야 합니다.
2. **2차 룰셋 검증 (내용 제어):** 스키마가 통제하지 못하는 부분은 `framework-rules.json`과 CI 스크립트가 검증합니다.
   * 확장 필드 `name`이 허용 명사 사전에 없으면 빌드 기각.
   * `control_id ≡ catalog_id + ":control:" + control_no` 합성 일치.
   * `domain_no`/`group_no`는 통제 참조가 아니므로 참조 무결성 검사에서 제외.
   * 반정규화 일관성: 같은 `domain_no`/`group_no`는 항상 같은 명칭.
   * `split_type` 일관성: `ATOMIC`이면 `base_no == control_no`; `OVERLAY`이면 `overlay_of`가 동일 `base_no`의 원본을 가리킴; `COMPOSITE`이면 `overlay_of`는 null이고 동일 `base_no` 형제가 둘 이상 존재.
   * `title` 품질: 공백이거나 접속사/문장부호로 끊긴 추출 오류 금지.
   * `catalog`의 집계값은 `control` 데이터에서 계산한 실제 값과 일치.
3. **소문자·구분자 강제:** URN 필드에 대문자나 비표준 구분자(`#`)가 1글자라도 있으면 빌드 거부.
