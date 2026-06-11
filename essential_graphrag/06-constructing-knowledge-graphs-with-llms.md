# 06. Constructing Knowledge Graphs with LLMs

## 챕터 개요 (3줄 요약)
- text embedding만으로는 filtering·counting·aggregation이 어려워, unstructured text를 structured data로 추출해 knowledge graph를 구성한다.
- LLM의 Structured Outputs(Pydantic schema)로 계약서 등에서 일관된 구조화 데이터를 추출한다.
- 추출 결과를 Neo4j graph로 import하고, entity resolution과 unstructured 데이터 결합으로 품질을 높인다.

---

## 1. Extracting structured data from text
> legal 문서처럼 document 경계가 중요한 경우 단순 chunk 검색은 여러 계약의 chunk가 섞여 부정확해진다.

- 문제 1: 여러 계약을 chunk·embed하면 "payment terms" 질의 시 다른 계약 chunk가 섞임.
- 문제 2: "ACME와 active 계약 몇 개?" 같은 filter+count는 embedding으로 불가(BI성 질의).
- 해법: LLM으로 unstructured → structured(table/key-value) 추출 → knowledge graph.
- workflow: contract 문서 → LLM 추출(parties, dates, terms 등) → JSON → Neo4j 저장.

### 1.1 Structured Outputs model definition
- 과거 information extraction은 다중 ML 모델·고비용. LLM이 prompt만으로 진입장벽 대폭 낮춤.
- OpenAI **Structured Outputs**: 출력 schema 사전 정의(function calling/schema). Python은 Pydantic 사용.

```python
class Contract(BaseModel):
    contract_type: str = Field(..., enum=contract_types)
    parties: List[Organization]
    effective_date: str = Field(..., description="yyyy-MM-dd")
    end_date: Optional[str]   # 누락 가능 → Optional (없으면 hallucination 방지)
    total_amount: Optional[float]
```

- attribute별 **description** 필수(format 지정, 예: date yyyy-MM-dd, country ISO 2-letter).
- **enum**으로 허용값 제한(contract_type). 예시만 줄 땐 description에 명시.
- 중첩 object 가능(Contract→Organization→Location)하나 깊은 중첩은 성능 저하.
- mandatory vs Optional 구분 중요: Optional 미표시 시 일부 LLM이 값을 hallucinate.

### 1.2 Extraction request
- system message로 도메인·용도 안내 + user message로 원문 전달.

```python
def extract(document, model="gpt-4o-2024-08-06", temperature=0):
    response = client.beta.chat.completions.parse(
        model=model, temperature=temperature,
        messages=[{"role":"system","content":system_message},
                  {"role":"user","content":document}],
        response_format=Contract)
    return json.loads(response.choices[0].message.content)
```

### 1.3 CUAD dataset
- 실제 계약은 기밀이라 공개 CUAD(Contract Understanding Atticus Dataset) 사용.
- 추출 결과: contract_type, parties(name/location/role), effective_date, term 등. 없는 값은 None.

---

## 2. Constructing the graph
> 추출한 structured output을 graph model로 설계해 Neo4j에 import한다.

- 그래프 모델: `Contract`, `Organization`, `Location` 3개 entity. Organization-[HAS_PARTY]->Contract, Organization-[LOCATED_AT]->Location.

### 2.1 Data import
- best practice: unique constraint·index 정의(무결성+성능).
- Cypher로 dict를 graph schema에 맞게 import.
- 주의: `randomUUID()` 사용 시 query가 idempotent하지 않음 → 반복 실행하면 중복 Contract 생성.

```cypher
MERGE (contract:Contract {id: randomUUID()})
SET contract += {...}
UNWIND $data.parties AS party
MERGE (p:Organization {name: party.name})
MERGE (loc:Location {fullAddress: ...})
MERGE (p)-[:LOCATED_AT]->(loc)
MERGE (p)-[r:HAS_PARTY]->(contract) SET r.role = party.role
```

### 2.2 Entity resolution
- 같은 실세계 entity의 여러 표현(예: "...Company" / "...Company Limited" / "...Company Ltd")을 단일 node로 병합.
- 기법: string matching, clustering, context 기반 ML.
- 매우 도메인·use case 특화 → 일반 해법 드묾. domain ontology/rule + SME + iterative feedback 권장.

### 2.3 Adding unstructured data to the graph
> knowledge graph에 원문(unstructured)과 추출된 structured 데이터를 함께 저장하면 풍부함과 정밀 질의를 동시에 얻는다.

- LLM으로 추출하되 원본 문서도 graph에 보존.
- chunking: 일반은 token/word 길이 기반. legal은 clause 단위 분할이 semantic 구조 보존에 유리.

---

## Summary (핵심 정리)
- 단순 chunk 검색은 document 경계가 중요한 도메인(legal 등)에서 결과가 섞여 부정확할 수 있다.
- filtering·sorting·aggregation은 structured data가 필요하며 embedding만으로는 부적합하다.
- LLM은 unstructured text에서 structured data(table·key-value)를 효과적으로 추출한다.
- Structured Outputs는 schema를 정의해 일관된 출력 형식을 보장하고 후처리를 줄인다.
- contract_type·parties·dates 등 명확한 data model 정의가 정확한 추출의 핵심이다.
- entity resolution은 동일 entity의 여러 표현을 병합해 데이터 일관성을 높인다.
- structured + unstructured 결합은 원본의 풍부함을 보존하며 정밀 질의를 가능케 한다.
