# 새로운 온톨로지 구축 기획 참고 문서

> 작성일: 2026-05-12  
> 상태: Claude CLI 피드백 반영 v1  
> 목적: 새로운 지식베이스/그래프DB/에이전트 메모리 시스템을 설계할 때, “무엇을 분류하고 어떤 관계로 연결할지”를 넘어 “어떻게 조회·추론·운영할지”까지 빠르게 기획하기 위한 실무 체크리스트.

---

## 0. 30분 v0 미니 캔버스

새 온톨로지를 처음 기획할 때는 아래 12칸만 먼저 채운다. 이 캔버스가 비어 있으면 GraphDB, 벡터DB, RDB 중 무엇을 써도 결국 운영 중 다시 사람을 부르게 된다.

```yaml
ontology_name: ""
mission: "이 온톨로지가 더 잘하게 만들 일 1문장"
primary_users:
  humans: []
  agents: []
representative_questions:
  - "반드시 답해야 하는 질문 1"
  - "반드시 답해야 하는 질문 2"
model_choice:
  graph_model: RDF | LPG | Hybrid
  reason: "왜 이 모델인가"
identity_strategy:
  namespace_rule: "예: ai.company:openai"
  external_ids: [wikidata, crunchbase, linkedin, github, ror]
core_entity_types: []
core_relation_types: []
source_and_claim_policy:
  source_tiers: [primary, secondary, social, inferred]
  claim_requires_source: true
query_api:
  named_queries: []
  max_traversal_depth: 2
  blocked_relations_for_agent: [related_to]
governance:
  owner: ""
  type_change_requires_review: true
  breaking_change_rule: ""
quality_metrics:
  question_coverage_target: "80%+"
  orphan_node_target: "<5%"
  constraint_violation_target: 0
```

---

## 0.1 먼저 고르는 기술 모델: RDF vs LPG vs Hybrid

온톨로지 문서는 구현과 분리되어야 하지만, 실무에서는 그래프 모델 선택이 표현 방식을 크게 바꾼다. 따라서 초기에 다음 선택지를 명시한다.

### RDF / Triple Store가 맞는 경우

- 표준 어휘와 호환성이 중요하다.
- OWL, RDFS, SHACL, SPARQL 같은 표준 추론/검증 생태계를 쓰고 싶다.
- URI/IRI 기반의 전역 식별자가 중요하다.
- 기관/학술/공공 데이터처럼 의미 표준화와 교환이 중요하다.

주의:

- 기본 triple은 `subject - predicate - object`라 엣지에 속성을 직접 붙이기 어렵다.
- 금액, 날짜, confidence가 붙는 관계는 reification, named graph, event node 같은 패턴이 필요하다.

### LPG(Labelled Property Graph: Neo4j 등)가 맞는 경우

- 엣지에 속성(`date`, `amount`, `confidence`)을 자주 붙인다.
- Cypher/Gremlin 기반 탐색이 핵심이다.
- 애플리케이션 개발 속도와 디버깅 편의가 중요하다.
- 에이전트가 named query API를 통해 그래프를 탐색한다.

주의:

- 표준 온톨로지/추론 생태계는 RDF보다 약하다.
- 구현체별 쿼리/스키마 방식에 종속되기 쉽다.

### Hybrid가 맞는 경우

- 내부 운영은 LPG로 빠르게 하고, 외부 교환/표준 매핑은 RDF/JSON-LD로 내보낸다.
- 원천 데이터는 RDB/Object Storage에 두고, 의미 계층은 Graph + Vector index로 구성한다.

권장 기본값:

```text
초기 스타트업/에이전트 운영 지식베이스: LPG + Vector + 원문 storage
공공/학술/표준 호환 지식베이스: RDF + SHACL + SPARQL
둘 다 필요한 장기 시스템: 내부 LPG, 외부 RDF/JSON-LD export
```

---

## 1. 한 줄 정의

온톨로지는 단순히 노드와 엣지를 많이 만드는 일이 아니라, **도메인의 개념·관계·제약·조회 방식·해석 규칙을 합의 가능한 형태로 명문화한 의미 모델**이다.

주의할 점은, 온톨로지 자체는 “의미 모델”이고 DB 스키마는 그 구현 중 하나라는 점이다. 실무 문서에서는 보통 둘을 함께 다루지만, 개념 정의(TBox), 인스턴스 데이터(ABox), 저장소 스키마, 쿼리 API를 구분해두면 나중에 변경 비용이 줄어든다.

관계형 DB 관점으로 말하면:

- 테이블/컬럼 설계만이 아니라,
- 엔티티 분류 체계,
- 엔티티 간 관계 의미,
- 허용/금지되는 관계,
- 조회 패턴,
- 추론 규칙,
- 데이터 품질 기준,
- 변경/버전 관리 규칙까지 포함한 설계 문서다.

GraphDB 관점으로 말하면:

- 노드 타입을 정하는 것,
- 엣지 타입을 정하는 것,
- 프로퍼티를 정하는 것,
- 어떤 경로를 따라 탐색할지,
- 어떤 경로는 의미 없거나 위험한지,
- 어떤 관계는 자동 유추해도 되는지,
- 어떤 답변에는 출처/시간/확신도를 붙여야 하는지를 정하는 일이다.

---

## 2. 왜 필요한가

온톨로지가 없으면 그래프는 금방 “연결된 잡동사니”가 된다.

초기에는 노드와 링크가 많아 보이지만, 시간이 지나면 다음 문제가 생긴다.

1. 같은 개념이 여러 이름으로 중복된다.
2. 관계 이름은 많은데 의미 차이가 불분명하다.
3. 어떤 쿼리를 해야 원하는 답이 나오는지 매번 사람이 기억해야 한다.
4. 에이전트가 링크를 따라가다가 쓸모없는 경로로 빠진다.
5. 오래된 정보와 최신 정보가 섞인다.
6. 출처·근거·확신도가 없어 답변 품질을 검증하기 어렵다.
7. 새 데이터를 넣을 때마다 기존 구조가 흔들린다.

좋은 온톨로지는 반대로 다음을 가능하게 한다.

- 사람이 봐도 이해되는 지식 구조
- 에이전트가 안전하게 탐색할 수 있는 경로
- 반복 가능한 쿼리 패턴
- 자동 분류/자동 링크/자동 요약의 기준
- 데이터 품질 점검
- 장기적으로 확장 가능한 지식 운영

---

## 3. 핵심 구성 요소

### 3.1 도메인 범위

먼저 “이 온톨로지가 책임지는 세계”를 좁혀야 한다.

정해야 할 것:

- 포함할 도메인
- 제외할 도메인
- 주요 사용 사례
- 주요 사용자/에이전트
- 답해야 하는 질문
- 답하지 않아도 되는 질문

예시:

```text
이 온톨로지는 AI 뉴스/기업/제품/인물/투자/기술 변화를 Threads 콘텐츠 운영 관점에서 해석하기 위한 지식 구조다.
회계, 인사, 법무 계약 원문 관리는 범위 밖이다.
```

### 3.2 엔티티 타입

엔티티 타입은 그래프의 노드 분류다.

기획 시 질문:

- 이 도메인의 주요 명사는 무엇인가?
- 사람이 반복적으로 구분해서 말하는 대상은 무엇인가?
- 검색/필터링/권한/수명주기가 다른 대상은 무엇인가?
- 같은 노드로 합쳐도 되는 것과 분리해야 하는 것은 무엇인가?

예시 타입:

- Person
- Company
- Product
- Model
- ResearchPaper
- Event
- FundingRound
- Regulation
- ContentPiece
- Claim
- Source
- Insight
- Workflow
- Dataset
- Metric

각 타입에는 최소한 다음을 정의한다.

```yaml
entity_type: Company
meaning: 법적/운영적 주체로서의 회사
examples:
  - OpenAI
  - Anthropic
  - NVIDIA
required_properties:
  - name
  - canonical_slug
recommended_properties:
  - website
  - founded_year
  - headquarters
  - status
identity_rule: 같은 공식 웹사이트 또는 신뢰 가능한 외부 ID가 같으면 동일 회사로 본다.
lifetime: 회사가 인수/폐업되어도 과거 이벤트 참조를 위해 유지한다.
```

### 3.2.1 식별자와 네임스페이스

엔티티 타입보다 먼저 ID 전략이 필요하다. ID가 흔들리면 병합/중복/출처 연결이 전부 흔들린다.

정해야 할 것:

- 내부 ID 형식: `namespace:type:slug` 또는 UUID
- 사람이 읽는 slug와 불변 ID를 분리할지
- 외부 ID 매핑: Wikidata QID, Crunchbase, LinkedIn, GitHub, DOI, ORCID, ROR 등
- alias 처리: 이름 변경, 브랜드명, 약어, 한글/영문명
- ID 발급 주체: human / agent / importer
- merge 정책: 어떤 근거가 있어야 같은 엔티티로 병합하는가

예시:

```yaml
entity_id: ai.company:openai
canonical_name: OpenAI
aliases: [OpenAI Inc., 오픈AI]
external_ids:
  wikidata: Q21708200
  website: https://openai.com
identity_rule: 공식 웹사이트 또는 신뢰 가능한 외부 ID가 같으면 동일 회사로 본다. 이름 유사도만으로 자동 병합하지 않는다.
```

### 3.2.2 클래스 계층과 다중 타이핑

노드 classification은 평면 목록으로 끝나면 안 된다. 최소한 상위 클래스, 역할, 다중 타이핑 기준을 정한다.

예시:

```text
Agent
├── Person
└── Organization
    ├── Company
    ├── University
    └── GovernmentAgency

CreativeWork
├── Source
├── ResearchPaper
└── ContentPiece
```

역할은 타입과 분리하는 편이 안전하다.

- `Person`은 타입이다.
- `Author`, `Investor`, `CEO`, `Speaker`는 특정 맥락의 role이다.
- 한 사람이 동시에 여러 role을 가질 수 있다.

권장 패턴:

```yaml
node_type: Person
roles:
  - role: CEO
    organization: ai.company:openai
    start_date: 2024-01-01
    source: ...
```

### 3.2.3 표준 어휘 재사용 원칙

기존 표준이 있는 개념은 가급적 빌려 쓴다.

- Person / Organization / Event / CreativeWork: schema.org
- 사람/조직 기본 관계: FOAF
- 개념 taxonomy: SKOS
- 문서 메타데이터: Dublin Core
- 출처/파생 관계: PROV-O
- 외부 엔티티 기준점: Wikidata

단, 내부 운영에서 빠른 개발이 중요하면 완전한 표준 준수보다 “표준으로 export 가능한 매핑”을 남기는 정도로 시작해도 된다.

### 3.3 관계 타입

관계 타입은 그래프의 엣지 의미다. 이름만 정하면 부족하고, 방향·의미·제약·증거 수준을 함께 정해야 한다.

각 관계는 다음 형식으로 정의한다.

```yaml
relation_type: acquired
from: Company
to: Company
meaning: from 회사가 to 회사를 인수했다.
direction: acquirer -> acquired_company
allowed_cardinality: many-to-many
required_properties:
  - date
  - source_url
optional_properties:
  - amount
  - currency
  - confidence
inverse_relation: acquired_by
query_use: 특정 회사의 성장 전략, 경쟁 구도, 기술 확보 경로 분석
pitfalls:
  - 투자(invested_in)와 인수(acquired)를 혼동하지 말 것
  - 루머 단계는 acquired가 아니라 rumored_to_acquire로 기록
```

좋은 관계명은 동사처럼 읽혀야 한다.

- `Company developed Product`
- `Person works_at Company`
- `Paper introduces Method`
- `ContentPiece cites Source`
- `Insight derived_from Source`
- `Claim supported_by Source`

나쁜 관계명:

- `related_to`
- `connected_to`
- `has`
- `misc`

이런 관계는 너무 범용적이라 쿼리 품질을 망친다.

### 3.3.1 N항 관계와 이벤트 노드

세 개 이상의 엔티티가 얽히거나, 관계 자체에 중요한 속성이 붙으면 단순 엣지보다 이벤트/상황 노드로 승격한다.

예시:

```text
나쁜 단순화:
CompanyA acquired CompanyB {date, amount, source}

더 안전한 모델:
AcquisitionEvent
- acquirer -> CompanyA
- acquired_company -> CompanyB
- announced_date -> Date
- amount -> Money
- supported_by -> Source
- status -> rumored | announced | completed | cancelled
```

승격 기준:

- 관계에 날짜/금액/상태/출처/confidence가 중요하게 붙는다.
- 동일 관계가 여러 번 발생할 수 있다.
- 여러 주체가 같은 사건에 다른 역할로 참여한다.
- 나중에 그 사건 자체를 검색/인용/콘텐츠화해야 한다.

### 3.3.2 관계 성질 명시

관계마다 다음 성질을 명시하면 추론과 쿼리가 안전해진다.

```yaml
relation_type: competes_with
symmetric: true
transitive: false
inverse_relation: competes_with
time_scoped: true
agent_traversable: true
max_hops_for_agent: 1
```

특히 `inverse_relation`은 저장할지, 쿼리 시 역방향 탐색만 허용할지 결정해야 한다. LPG는 보통 단방향 저장 + 양방향 탐색이 가능하고, RDF는 역관계를 명시하거나 추론 규칙으로 처리하는 경우가 많다.

### 3.4 속성/프로퍼티

프로퍼티는 노드나 엣지에 붙는 구조화 정보다.

주의할 점:

- 자주 필터링할 값은 프로퍼티로 둔다.
- 독립적으로 연결/출처/이력이 필요한 대상은 노드로 승격한다.
- 시간이 지나며 바뀌는 값은 단순 프로퍼티보다 이벤트/타임라인으로 분리한다.

예시:

- 회사의 `current_ceo`는 바뀔 수 있으므로 단순 문자열보다 `Person works_at Company`, `role=CEO`, `start_date` 관계로 관리하는 편이 낫다.
- 모델의 `benchmark_score`는 측정 조건이 중요하므로 `BenchmarkResult` 노드로 분리하는 편이 낫다.

### 3.5 제약 조건

온톨로지는 “무엇을 허용하지 않을지”가 중요하다.

정해야 할 제약:

- 어떤 타입끼리 연결 가능한가?
- 필수 속성은 무엇인가?
- 유일성 기준은 무엇인가?
- 시간 순서가 말이 되는가?
- 출처 없는 주장을 허용하는가?
- 같은 관계가 중복 생성될 때 병합 기준은 무엇인가?

예시:

```yaml
constraint: Claim requires Source
rule: Claim 노드는 최소 1개 이상의 supported_by 또는 contradicted_by Source 관계를 가져야 한다.
severity: error
reason: 출처 없는 주장은 운영 콘텐츠에 사용하면 안 된다.
```

### 3.5.1 시간 모델링

시간은 단순 `date` 하나로 끝내면 안 된다. 최소한 다음을 구분한다.

- valid time: 사실이 실제 세계에서 유효한 기간
- transaction time: 시스템에 기록된 시점
- observed time: 수집기가 관측한 시점
- published time: 출처가 공개된 시점

예시:

```yaml
fact: Person works_at Company
role: CEO
valid_from: 2024-01-01
valid_to: null
recorded_at: 2026-05-12T10:00:00Z
source_published_at: 2024-01-02T09:00:00Z
```

대표 질문에 “그 시점에는 어땠나?”가 있으면 반드시 시간 슬라이스 쿼리를 설계한다.

### 3.5.2 출처/증거 메타모델

Source는 URL 하나가 아니라 provenance 체계다.

권장 속성:

```yaml
source_id: source:...
source_type: official_blog | paper | news | social | api | internal_note
tier: primary | secondary | tertiary | social | inferred
url: ...
archived_url: ...
published_at: ...
fetched_at: ...
author_or_org: ...
language: ...
license: ...
```

권장 관계:

- `Claim supported_by Source`
- `Claim contradicted_by Source`
- `Insight derived_from Claim`
- `Source cites Source`
- `Source archived_as Snapshot`

운영 규칙:

- 공개 콘텐츠에 쓰는 Claim은 최소 1개 이상의 Source가 필요하다.
- LLM이 만든 Insight는 원천 Claim과 분리한다.
- URL은 사라질 수 있으므로 중요한 Source는 스냅샷/아카이브를 남긴다.

### 3.5.3 권한과 민감정보

Person, 내부 노트, 고객 데이터가 들어가는 순간 접근 제어가 필요하다.

정해야 할 것:

- 노드/엣지/속성 단위 공개 범위
- PII 속성 분리 여부
- 삭제 요청 대응
- 외부 공개 콘텐츠에 사용할 수 없는 관계/출처
- 에이전트가 조회해도 되는 필드와 금지 필드

예시:

```yaml
property: Person.email
sensitivity: pii
allowed_contexts: [internal_crm]
blocked_contexts: [public_content_generation]
```

### 3.6 추론 규칙

모든 관계를 직접 저장할 필요는 없다. 일부는 유추할 수 있다.

예시:

```text
Source contains Claim + Claim supported_by Source
=> Claim confidence can be raised only within the same claim/evidence cluster.
```

위험한 추론 예시도 명시적으로 금지한다.

```text
Person works_at Company + Person authored Paper
=> Company has_research_activity_in Paper.topic
```

이 추론은 소속 시점, 논문 작성 시점, 개인 연구와 회사 연구의 경계를 무시할 수 있으므로 자동 확정하면 안 된다. 필요하면 `weak_signal` 또는 `candidate_insight`로만 생성하고 사람 검토를 거친다.

하지만 추론은 위험하다. 따라서 규칙마다 다음을 정한다.

- 유추해도 되는가?
- 유추 결과를 저장할 것인가, 쿼리 시 계산할 것인가?
- 확신도는 얼마인가?
- 사람이 검토해야 하는가?
- 답변에 “추정”이라고 표시해야 하는가?

---

## 4. 기획 순서

### Step 1. 대표 질문 20개를 먼저 쓴다

온톨로지는 쿼리에서 역산해야 한다.

예시 질문:

1. 특정 회사가 최근 어떤 제품/모델/논문으로 방향을 바꾸고 있는가?
2. 이 이벤트는 기존 어떤 흐름의 연장선인가?
3. 이 뉴스에서 콘텐츠화할 수 있는 운영 인사이트는 무엇인가?
4. 특정 인물은 어떤 회사/프로젝트/논문과 연결되는가?
5. 어떤 주장이 어떤 출처에 의해 지지되거나 반박되는가?
6. 오늘 나온 신호 중 과거에 반복된 패턴은 무엇인가?
7. 경쟁사 대비 기술 포지션은 어떻게 달라졌는가?
8. 이 정보는 얼마나 신뢰할 수 있는가?
9. 어떤 링크 경로를 따라가면 맥락 설명이 가능한가?
10. 어떤 정보는 오래되어 폐기해야 하는가?

이 질문들이 실제 스키마의 기준이 된다.

### Step 2. 답변에 필요한 노드 타입을 뽑는다

질문 안의 명사를 후보 노드로 뽑는다.

예:

```text
회사, 제품, 모델, 논문, 이벤트, 인물, 주장, 출처, 인사이트, 콘텐츠, 기술, 시장, 규제
```

### Step 3. 답변에 필요한 관계를 뽑는다

질문 안의 동사를 후보 관계로 뽑는다.

예:

```text
개발했다, 발표했다, 인용했다, 반박했다, 투자했다, 인수했다, 경쟁한다, 파생됐다, 영향을 준다, 대체한다
```

### Step 4. 관계를 정규화한다

비슷한 관계를 합친다.

예:

- `announced`, `launched`, `released`는 구분할 필요가 있는가?
- `uses`, `built_on`, `depends_on`은 같은가 다른가?
- `mentions`와 `cites`는 다르다. cites는 근거로 삼는다는 뜻이다.

### Step 5. 최소 온톨로지 v0를 만든다

처음부터 완벽한 온톨로지를 만들지 않는다.

v0 기준:

- 핵심 엔티티 타입 8~15개
- 핵심 관계 타입 15~30개
- 필수 속성만 정의
- 대표 쿼리 10개가 답변 가능하면 충분

### Step 6. 샘플 데이터 30~100개로 검증한다

실제 데이터 없이 온톨로지를 설계하면 거의 틀린다.

검증 방식:

- 실제 기사/문서/노트/DB 레코드 30~100개를 넣어본다.
- 중복 엔티티가 생기는지 본다.
- 관계명이 모호한지 본다.
- 대표 질문에 답이 나오는지 본다.
- 에이전트가 이상한 경로로 탐색하는지 본다.

### Step 7. 운영 규칙을 붙인다

온톨로지는 문서가 아니라 운영 체계다.

필요한 운영 규칙:

- 새 타입 추가 기준
- 새 관계 추가 기준
- 중복 병합 기준
- 출처 신뢰도 등급
- 오래된 정보 처리
- 사람이 검토해야 하는 변경
- 자동화 가능한 변경
- 버전 관리 방식

---

## 5. 실무 설계 템플릿

새 온톨로지를 기획할 때 아래 템플릿을 채우면 된다.

```markdown
# [온톨로지 이름] Ontology Spec v0

## 1. 목적
이 온톨로지는 무엇을 더 잘하기 위해 존재하는가?

## 2. 범위
### 포함
- 

### 제외
- 

## 3. 주요 사용자
- 사람 사용자:
- 에이전트/시스템 사용자:

## 4. 대표 질문
1. 
2. 
3. 

## 5. 엔티티 타입
### EntityTypeName
- 의미:
- 예시:
- 필수 속성:
- 선택 속성:
- 동일성 판단 기준:
- 수명주기:
- 생성 주체: human / agent / importer

## 6. 관계 타입
### relation_name
- From:
- To:
- 의미:
- 방향:
- 필수 속성:
- 선택 속성:
- 역관계:
- 허용 조건:
- 금지 조건:
- 대표 쿼리:

## 7. 속성 표준
- 날짜 형식:
- URL 형식:
- 신뢰도 형식:
- slug 규칙:
- 언어/로케일 규칙:

## 8. 제약 조건
- 

## 9. 추론 규칙
- 

## 10. 쿼리 패턴
### QueryPatternName
- 의도:
- 입력:
- 탐색 경로:
- 반환 형식:
- 주의사항:

## 11. 데이터 수집/입력 규칙
- 원천 소스:
- ingest 방식:
- 중복 제거:
- 검수 기준:

## 12. 품질 점검
- orphan node 허용 여부:
- 출처 없는 claim 허용 여부:
- deprecated 관계 처리:
- 정기 lint 항목:

## 13. 버전 관리
- 변경 제안 방식:
- breaking change 기준:
- migration 방식:
```

---

## 6. 대표 쿼리 패턴 설계

온톨로지를 잘 만들려면 “저장 구조”보다 “꺼내는 방식”을 먼저 설계해야 한다.

### 6.1 맥락 확장 쿼리

목적: 어떤 사건/문서/뉴스가 기존 흐름과 어떻게 연결되는지 파악한다.

```text
Event
-> involves Company/Person/Product
-> related_to prior Event
-> supported_by Source
-> derived Insight
```

반환:

- 핵심 사건
- 관련 주체
- 선행 사건
- 근거 출처
- 해석 가능한 인사이트

### 6.2 주장 검증 쿼리

목적: 특정 claim이 근거 있는지 확인한다.

```text
Claim
-> supported_by Source
-> contradicted_by Source
-> made_by Person/Organization
-> about Entity
```

반환:

- claim 본문
- 지지 출처
- 반박 출처
- 출처 신뢰도
- 최종 confidence

### 6.3 콘텐츠 생성 쿼리

목적: 지식 그래프에서 콘텐츠 소재를 뽑는다.

```text
Insight
-> derived_from Source/Event
-> about Company/Product/Workflow
-> connected_to prior ContentPiece
-> has_angle AudiencePain/OperatorLesson
```

반환:

- 핵심 인사이트
- 새로움
- 독자 효용
- 기존 콘텐츠와의 중복 여부
- 사용 가능한 출처 링크

### 6.4 변화 감지 쿼리

목적: 시간이 지나며 포지션이 바뀐 엔티티를 찾는다.

```text
Company
-> announced Event over time
-> launched Product over time
-> hired Person over time
-> invested_in/acquired Company over time
```

반환:

- 변화 전 상태
- 변화 후 상태
- 변화 신호
- 해석
- confidence

### 6.5 에이전트 작업 쿼리

목적: 에이전트가 다음 행동을 결정하도록 한다.

```text
Task/Workflow
-> requires Entity/Source
-> blocked_by MissingInfo
-> produces ContentPiece/Insight
-> reviewed_by Human/Agent
```

반환:

- 다음 작업
- 필요한 정보
- 막힌 이유
- 검수 필요 여부

---

## 6.6 조회 운영 설계

회장님 관점에서 온톨로지의 실전 가치는 “꺼낼 때” 드러난다. 따라서 쿼리 전략은 별도 산출물로 둔다.

### 쿼리 언어 선택

- RDF: SPARQL
- LPG: Cypher / Gremlin
- 앱 API: GraphQL / REST named query
- 에이전트용: 제한된 named query API 권장

에이전트에게 자유 Cypher/SPARQL을 바로 주기보다, 검증된 named query를 제공하는 편이 안전하다.

```yaml
query_name: context_for_event
input:
  event_id: string
traversal:
  - Event -> involves -> Entity
  - Event -> supported_by -> Source
  - Event -> related_prior_event -> Event
limits:
  max_depth: 2
  max_nodes: 50
blocked_relations:
  - related_to
return:
  - event_summary
  - entities
  - sources
  - prior_events
  - confidence
failure_mode: return_partial_with_missing_fields
```

### Hop 폭발 방지

그래프 탐색은 금방 폭발한다.

규칙:

- 기본 max depth는 2~3으로 제한한다.
- `related_to`는 탐색 종료 관계로 둔다.
- high-degree node(예: OpenAI, AI)는 직접 확장하지 않고 필터를 요구한다.
- 시간 범위 없는 쿼리는 기본 최근 90일 등으로 제한한다.
- 결과에는 항상 경로(path)를 함께 반환한다.

### 인덱스와 캐싱

- 자주 찾는 slug/external_id/source_url에는 유니크 인덱스
- 시간축 쿼리를 위한 `published_at`, `valid_from` 인덱스
- 콘텐츠 후보/타임라인처럼 반복 조회되는 결과는 materialized view 또는 cache
- 임베딩은 노드 본문, Source chunk, Claim, Insight 단위로 분리

### Vector + Graph 결합 패턴

권장 순서:

1. Vector search로 후보 Source/Claim/Insight를 찾는다.
2. Graph traversal로 출처, 관련 엔티티, 이전 이벤트를 확장한다.
3. 제약/권한/시간 필터로 위험 후보를 제거한다.
4. named query 형식으로 결과를 반환한다.

피해야 할 방식:

- 벡터 검색 결과를 근거 검증 없이 바로 답변에 사용
- 그래프 전체를 무제한 traversal
- LLM에게 자유롭게 관계 의미를 추측하게 하기

---

## 7. RDB와 GraphDB 관점 비교

### RDB로 충분한 경우

- 관계가 안정적이고 예측 가능하다.
- 조인이 정해져 있다.
- 트랜잭션 정합성이 중요하다.
- 데이터 모델이 자주 바뀌지 않는다.
- 주요 조회가 대시보드/CRUD 중심이다.

### GraphDB가 유리한 경우

- 관계 자체가 핵심 정보다.
- 탐색 경로가 매번 달라진다.
- “A와 B가 어떻게 연결되는가?”가 중요한 질문이다.
- 추천/맥락/영향/계보/지식 연결이 중요하다.
- 에이전트가 링크를 따라 사고해야 한다.

### 실무적으로는 혼합이 많다

권장 구조:

- RDB: 원천 레코드, 권한, 상태, 트랜잭션, 큐, 작업 로그
- GraphDB/지식그래프: 엔티티·관계·맥락·추론·탐색
- Vector index: 비정형 텍스트 검색, 유사도 검색
- Object storage: 원문 문서, 이미지, PDF, 스냅샷

온톨로지는 이 중 GraphDB만이 아니라 **전체 지식 시스템의 의미 계층**으로 보는 편이 좋다.

---

## 8. 설계할 때 자주 하는 실수

### 실수 1. relation을 전부 `related_to`로 만든다

해결:

- `related_to`는 임시/약한 관계로만 허용한다.
- 핵심 쿼리에 쓰이는 관계는 의미 있는 동사로 승격한다.

### 실수 2. 타입을 너무 많이 만든다

해결:

- v0에서는 핵심 타입만 둔다.
- 구분 기준이 쿼리에 영향을 줄 때만 타입을 나눈다.

### 실수 3. 타입을 너무 적게 만든다

해결:

- 수명주기, 권한, 출처, 쿼리 방식이 다르면 별도 타입으로 분리한다.

### 실수 4. 출처와 주장을 분리하지 않는다

해결:

- Source는 원문이다.
- Claim은 원문에서 나온 주장이다.
- Insight는 claim과 맥락을 해석한 결과다.

이 셋을 섞으면 에이전트가 사실과 해석을 구분하지 못한다.

### 실수 5. 현재 상태만 저장한다

해결:

- 변화가 중요한 값은 이벤트/타임라인으로 저장한다.
- `current_status`만 두지 말고 `status_changed_at`, `previous_status`, `source`를 남긴다.

### 실수 6. 쿼리 패턴을 문서화하지 않는다

해결:

- 대표 질문마다 탐색 경로를 정의한다.
- 에이전트가 사용할 수 있는 named query pattern을 만든다.

---

## 9. 온톨로지 v0 산출물 체크리스트

최소 산출물:

- [ ] 목적/범위 문서
- [ ] 대표 질문 20개
- [ ] 엔티티 타입 목록
- [ ] 관계 타입 목록
- [ ] 타입별 필수 속성
- [ ] 관계별 from/to 제약
- [ ] 출처/claim/insight 분리 원칙
- [ ] 동일성 판단 기준
- [ ] 중복 병합 기준
- [ ] 대표 쿼리 패턴 5~10개
- [ ] 샘플 데이터 30~100개
- [ ] 품질 lint 규칙
- [ ] 버전 관리 규칙
- [ ] RDF/LPG/Hybrid 선택과 이유
- [ ] ID/네임스페이스/외부 ID 매핑 규칙
- [ ] 클래스 계층과 role 처리 기준
- [ ] N항 관계를 이벤트 노드로 승격하는 기준
- [ ] 시간 모델(valid/transaction/observed/published)
- [ ] 권한/PII 정책
- [ ] named query API와 traversal 제한
- [ ] Vector + Graph 결합 방식
- [ ] 품질 지표와 lint 규칙

---

## 10. 빠른 의사결정 기준

### 노드로 만들까, 속성으로 둘까?

노드로 만든다:

- 다른 것들과 관계를 맺는다.
- 출처/시간/확신도가 필요하다.
- 독립적으로 검색된다.
- 변경 이력이 중요하다.
- 여러 엔티티가 공유한다.

속성으로 둔다:

- 단순 값이다.
- 독립적으로 탐색하지 않는다.
- 변경 이력이 중요하지 않다.
- 필터링 정도에만 쓰인다.

### 관계를 새로 만들까, 기존 관계를 쓸까?

새 관계를 만든다:

- 기존 관계와 의미가 다르다.
- 쿼리 경로가 달라진다.
- 잘못 합치면 추론이 위험해진다.

기존 관계를 쓴다:

- 같은 질문에 같은 방식으로 쓰인다.
- 차이는 프로퍼티로 표현 가능하다.

### 자동 추론할까, 사람이 검토할까?

자동 가능:

- 규칙이 명확하다.
- 오탐 비용이 낮다.
- 출처가 강하다.
- confidence를 낮게 표시해도 된다.

사람 검토 필요:

- 사업적/법적 리스크가 있다.
- 공개 콘텐츠에 바로 사용된다.
- 해석이 논쟁적이다.
- 출처가 약하다.

---

## 11. 추천 v0 스키마 예시: 지식/콘텐츠 운영용

### Entity Types

- `Source`: 원문 기사, 논문, 영상, 문서, API 응답
- `Claim`: Source에서 추출한 사실 주장
- `Insight`: Claim과 맥락을 해석해 만든 운영 인사이트
- `Company`: 회사/조직
- `Person`: 인물
- `Product`: 제품/서비스
- `Model`: AI 모델
- `Technology`: 기술 개념
- `Event`: 발표, 출시, 투자, 인수, 규제, 사고
- `ContentPiece`: 작성/게시된 콘텐츠
- `Audience`: 독자/고객 세그먼트
- `Workflow`: 반복 운영 프로세스

### Relation Types

- `Source contains Claim`
- `Claim about Entity`
- `Claim supported_by Source`
- `Claim contradicted_by Source`
- `Insight derived_from Claim/Source/Event`
- `Insight about Entity`
- `Company developed Product/Model`
- `Company acquired Company`
- `Company invested_in Company`
- `Person works_at Company`
- `Person authored Source/Paper`
- `Event involves Entity`
- `Product uses Technology/Model`
- `ContentPiece cites Source`
- `ContentPiece expresses Insight`
- `ContentPiece targets Audience`
- `Workflow produces ContentPiece/Insight`
- `Workflow requires Source/Entity`

### Named Query Patterns

- `context_for_event(event_id)`
- `claims_about(entity_id, time_range)`
- `insights_from_source(source_id)`
- `content_candidates(topic, freshness, audience)`
- `entity_change_timeline(entity_id)`
- `evidence_for_claim(claim_id)`
- `related_prior_content(entity_id, angle)`

---

## 12. 온톨로지 거버넌스와 품질 지표

온톨로지는 한 번 만들고 끝나는 문서가 아니다. 운영 중 계속 바뀌므로 변경 규칙과 측정 지표가 필요하다.

### 변경 등급

```yaml
non_breaking:
  - optional property 추가
  - 새 Source tier 추가
  - 새 alias 추가
review_required:
  - 새 entity type 추가
  - 새 relation type 추가
  - inference rule 추가
breaking:
  - relation 의미 변경
  - entity type 분할/병합
  - required property 삭제/변경
  - identity rule 변경
```

### 마이그레이션 규칙

- breaking change는 migration script 또는 변환 규칙을 함께 작성한다.
- deprecated relation은 즉시 삭제하지 말고 `deprecated_at`, `replacement_relation`을 둔다.
- 오래된 쿼리 패턴이 어떤 영향을 받는지 기록한다.
- 롤백 가능한 단위로 버전 태그를 남긴다.

### 품질 지표

- 대표 질문 커버리지: 대표 질문 중 실제 답 가능한 비율
- 제약 위반 수: 필수 속성/관계 위반 수
- orphan node 비율: 연결 없는 노드 비율
- ambiguous relation 비율: `related_to` 등 약한 관계 비율
- duplicate candidate 수: 병합 후보 엔티티 수
- source coverage: Claim 중 Source가 붙은 비율
- traversal success rate: named query가 제한 시간 내 성공한 비율
- dead relation 수: 실제 쿼리에서 거의 쓰이지 않는 관계 수

### Lint 예시

```text
ERROR: Claim without Source
ERROR: Person.email exposed to public_content_generation
WARN: Entity has only related_to edges
WARN: Company node has no external_id and no official website
WARN: Event has announced_date later than completed_date
INFO: Relation type not used in last 90 days
```

---

## 13. 하지 말아야 할 것 10개

1. 모든 관계를 `related_to`로 때우지 말 것.
2. 이름 유사도만으로 엔티티를 자동 병합하지 말 것.
3. Source, Claim, Insight를 한 노드에 섞지 말 것.
4. LLM 추론 결과를 사실 관계처럼 저장하지 말 것.
5. 시간 정보 없이 현재 상태만 저장하지 말 것.
6. Person 데이터를 다루면서 PII/권한 모델을 빼먹지 말 것.
7. 대표 쿼리 없이 타입 목록부터 만들지 말 것.
8. 표준 어휘가 있는 개념을 무조건 자체 정의하지 말 것.
9. 에이전트에게 무제한 그래프 탐색 권한을 주지 말 것.
10. 온톨로지 변경을 migration 없이 문서 수정으로만 처리하지 말 것.

---

## 14. 최종 관점

회장님이 말한 것처럼, 개발 적용 관점에서는 온톨로지를 이렇게 봐도 된다.

> “GraphDB 기반으로 꺼낼 때, 노드 분류와 관계 의미를 어떻게 정리하고, 조회 경로와 상호작용 규칙을 어떻게 명문화하느냐.”

다만 여기에 반드시 더 붙어야 하는 것이 있다.

1. **제약**: 어떤 연결은 허용하지 않는다.
2. **증거**: 어떤 주장은 어떤 출처에서 왔는지 남긴다.
3. **시간**: 지금 사실과 과거 사실을 구분한다.
4. **추론**: 직접 저장한 것과 유추한 것을 구분한다.
5. **운영**: 새 타입/관계/데이터가 들어올 때 품질을 유지한다.

즉, 온톨로지는 DB 스키마보다 넓고, 지식 시스템의 “의미 운영 체계”에 가깝다.
