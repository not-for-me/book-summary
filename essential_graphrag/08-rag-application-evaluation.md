# 08. RAG Application Evaluation

## 챕터 개요 (3줄 요약)
- RAG 파이프라인의 각 단계(tool selection, retrieval relevance, answer generation)를 benchmark로 평가해야 한다.
- benchmark dataset은 다양한 질의를 포함하며, ground truth를 Cypher query로 정의해 데이터 변경에도 유효하게 유지한다.
- RAGAS의 context recall, faithfulness, answer correctness 세 metric으로 시스템 성능을 정량 평가한다.

---

## 1. Designing the benchmark dataset
> RAG 파이프라인의 각 단계를 시험하는 다양한 질의를 포함해야 한다.

- 평가 대상 단계(figure 8.1): tool selection 정확도 → retrieved context 관련성 → 생성 답변의 일관성·정확성 → 전체 파이프라인.
- 핵심: "올바른 context를 줬을 때 LLM이 올바른 답을 내는가"를 retrieval 성능과 분리해 측정.
- benchmark에 포함할 차원: tool selection 평가, entity/value mapping, multistep(chaining) retrieval, edge case·기능 커버리지, conversational usability(인사·모호 질의·역량 안내).

### 1.1 Test examples
- 각 example = 질문 + ground truth.
- **정적 string 대신 Cypher query로 ground truth 정의** → 데이터 변경돼도 benchmark 유효.
- 유형별 예시:
  - 인사·무관 질의: `RETURN "greeting..."` 등 static RETURN.
  - tool usage·value mapping: "Who acted in Top Gun?" (대소문자 변형 포함 → mapping 검증).
  - aggregation·filtering: "Who acted in the most movies?" → text2cypher 필요.
  - missing data edge case: "Which movie has the most Oscars?" → "This information is missing".
- 잘 알려진 entity는 pretraining으로 정답 생성되나, private·long-tail 데이터는 전용 mapping 시스템 필요.
- 예제 총 17개.

---

## 2. Evaluation (RAGAS)
> RAGAS framework로 context recall, faithfulness, answer correctness 세 metric을 측정한다.

### 2.1 Context recall
> 답변의 각 문장이 retrieved context로 뒷받침되는지(Yes=1/No=0) 분류해 retrieval이 필요 정보를 얼마나 포착했는지 측정.

### 2.2 Faithfulness
> 생성 답변이 retrieved context와 사실적으로 일치하는지(hallucination 여부) 평가. 2단계.

- 1단계: 답변을 pronoun 없는 atomic statement로 분해.
- 2단계: 각 statement가 context에서 직접 추론 가능(1)/불가(0) 판정.

### 2.3 Answer correctness
> 생성 답변과 ground truth를 비교해 정확성·완전성 평가.

- statement를 TP(답변·GT 모두 지지), FP(답변에만 있고 GT 미지지), FN(GT에만 있고 답변에 없음)으로 분류.

### 2.4 Loading & running
```python
test_data = pd.read_csv("benchmark_data.csv", delimiter=";")
# 각 행: Cypher 실행 → ground_truth, agent 실행 → answer/context, latency 기록
dataset = Dataset.from_pandas(test_data.fillna("I don't know"))
result = evaluate(dataset, metrics=[answer_correctness, context_recall, faithfulness])
```
- RAGAS는 non-null 필요 → 누락은 "I don't know"로 채움.

### 2.5 Observations
> faithfulness는 우수(0.97)하나 answer correctness·context recall(~0.78)은 retrieval 개선 여지를 시사한다.

- 결과: answer_correctness 0.7774, context_recall 0.7941, faithfulness 0.9657.
- 해석: hallucination은 거의 없으나(높은 faithfulness), retrieval coverage·답변 formulation 개선 필요.
- 패턴: text2cypher 불필요 질의는 LLM 호출이 줄어 latency↓. LLM-as-judge라 일부 score 불일치(예: Hello).
- 한계 예: "가장 긴 이름의 배우?" → 적절 Cypher 미생성 → few-shot 추가나 전용 tool로 보완. benchmark는 지속 확장.

---

## 3. Next steps
> tool 품질·설계·통합이 LLM 효과를 좌우하며, embedding 검색과 structured 추출을 결합해 agentic GraphRAG를 구축한다.

- LLM은 빠르게 발전하나 효과는 제공하는 tool의 품질·설계·통합에 의존.
- unstructured 데이터를 직접 embed(빠른 유사도 검색)하거나 entity·relationship·event를 추출해 knowledge graph 구성(정밀·semantic·multihop 질의) → 결합으로 진정한 context-aware 시스템.

---

## Summary (핵심 정리)
- RAG 파이프라인 평가는 정확·일관된 답변 보장에 필수이며, benchmark가 성능 측정과 역량 정의를 돕는다.
- 평가는 tool selection, context retrieval 관련성, answer 생성 품질, 전체 효과 등 여러 단계를 다룬다.
- 좋은 benchmark는 retrieval 정확도·entity mapping·인사·무관 질의·다양한 Cypher 조회를 포함한다.
- 정적 답 대신 Cypher query를 ground truth로 쓰면 데이터 변경에도 benchmark가 유효하다.
- context recall은 관련 정보 retrieval 정도를 측정한다.
- faithfulness는 생성 답변이 retrieved context와 사실적으로 일치하는지 평가한다.
- answer correctness는 답변이 질의를 완전·정확히 다루는지 평가한다.
