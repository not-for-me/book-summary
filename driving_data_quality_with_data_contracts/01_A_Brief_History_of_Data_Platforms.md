# 01. A Brief History of Data Platforms

## 챕터 개요 (3줄 요약)

- 지난 20여 년간 데이터 아키텍처는 EDW → 빅데이터 플랫폼 → 모던 데이터 스택(MDS)으로 발전했지만, 동일한 구조적 한계를 계속 이어왔다.
- 세 세대 모두 "소비를 위해 만들어지지 않은(not built for consumption)" 원천 데이터와 중앙 데이터팀이라는 병목(bottleneck)을 공유한다.
- 데이터가 비즈니스 핵심 애플리케이션에 점점 더 많이 쓰이는 만큼, 신뢰성·기대치·자율성을 근본적으로 개선하는 새로운 접근(데이터 컨트랙트)이 필요하다.

---

## 1. The Enterprise Data Warehouse (EDW)

> 1990년대 후반~2000년대 초의 EDW는 대형 중앙 DB를 중심으로 한 고비용 분석 솔루션으로, 오늘날까지 이어지는 병목의 원형을 보여준다.

- EDW는 보고/분석을 위한 통합 솔루션으로, 1~2개 대형 벤더에 의존했고 비용이 높아 대기업만 도입 가능했다.
- 중심에는 온프레미스(on-premises) Oracle 또는 MS SQL Server 같은 대형 DB가 있었다.
- 원천 시스템의 DB에 직접 ETL(Extract, Transform, Load)을 수행해 보고/분석을 구동했다.
- 원천 DB에 부하를 주어 업스트림 서비스 성능에 악영향을 주었고, 신뢰성이 낮았다.
- 스키마 변경 시 ETL을 수동 갱신해야 했고, 데이터 생성자의 사전 통보에 의존했다.
- 고정 크기의 비싼 온프레미스 DB라 확장이 어려워 저장·분석 가능한 데이터량이 제한되었다.
- ETL 개발자가 어떤 데이터를 넣을지 결정하는 유일한 통제자가 되어 병목(bottleneck)이 발생했다.
- 보통 회사 KPI를 구동하는 데이터만 우선 적재되었고, 그 외 요청은 백로그에 묶였다.

```
[Source DB] --ETL--> [EDW (Oracle/MSSQL)] --> [Reporting / Analytics]
     ^                       ^
   load impact         expensive, fixed-size
```

---

## 2. The Big Data Platform

> 2006년 Apache Hadoop 등장으로 저비용 대용량 저장·처리가 가능해졌지만, 거버넌스 부재로 '데이터 스왐프(data swamp)'와 '다크 데이터(dark data)' 문제가 생겼다.

- Google의 GFS(2003)·MapReduce(2004) 논문이 Yahoo!에서 구현되어 Apache Hadoop(2006)으로 오픈소스화되었다.
- HDFS(Hadoop Distributed File System)는 상용 하드웨어에 거의 무제한 데이터를 저장하고, MapReduce는 대규모 처리 모델을 제공했다.
- 비용이 낮아져 모든 규모의 조직이 데이터 레이크(data lake)를 구축할 수 있었다.
- 구조 제약이 없어 일단 저장하고 나중에 스키마/변환을 적용하는 ELT(Extract, Load, Transform) 개념이 등장했다.
- 그러나 문서·기대치·거버넌스가 없어 무엇이 들어있는지 알기 어려운 '데이터 스왐프'가 되었다.
- Gartner는 수집되었으나 쓰이지 않는 데이터를 '다크 데이터(dark data)'로 명명했다(2015년 IDC: 비정형 데이터의 90%가 다크).
- 중앙 데이터 엔지니어링팀이 도입되어 생성자와 최종 소비자 사이 거리가 더 멀어졌다.

```
[Sources] --ELT--> [Data Lake / HDFS] --MapReduce--> [EDW] --> [BI Tools]
                         |
                    dark data (unused)
```

---

## 3. The Modern Data Stack (MDS)

> 2012년 Amazon Redshift로 시작된 클라우드 네이티브 데이터 웨어하우스는 SQL 호환성과 MPP로 접근성을 높였지만, 원천 모델과의 강결합(tight coupling) 문제는 그대로 남았다.

- Amazon Redshift는 최초의 클라우드 네이티브 웨어하우스로, 저비용·MPP(Massively Parallel Processing) 기반의 SQL 호환 처리를 제공했다.
- BigQuery, Snowflake 등이 뒤따랐고 Fivetran/Stitch(ELT), dbt(변환), Hightouch(reverse ETL) 같은 도구 생태계가 폭발적으로 늘었다.
- 웨어하우스와 레이크의 장점을 합친 데이터 레이크하우스(data lakehouse)로 진화했다.
- CDC(Change Data Capture) 도구(Debezium, Striim 등)로 원천 트랜잭션 DB의 변경을 그대로 복제할 수 있게 되었다.
- 그러나 CDC는 업스트림 내부 모델과 소비자를 강결합시켜, 예고 없는 breaking change가 다운스트림을 망가뜨린다.
- 데이터가 분석용이 아닌 트랜잭션용으로 설계되어 변환·조인 비용이 크고, 문서가 거의 없다.
- Seagate(2022) 보고서에 따르면 조직 데이터의 68%가 여전히 미사용 상태로, 다크 데이터 문제가 지속된다.
- 변환 복잡성과 도메인 지식 요구 때문에 결국 중앙 데이터팀이 수천 개 모델을 관리하는 병목이 재현된다.

```
[Sources] --CDC/ELT--> [Data Lakehouse (ODS)] --transform--> [EDW area] --> [Consumers]
                              ^
                      tight coupling, not built for consumption
```

---

## 4. The State of Today's Data Platforms

> 오늘날 데이터 플랫폼은 기대치·신뢰성·자율성의 결핍이라는 보편적 문제를 안고 있다.

- 기대치 결핍(lack of expectations): 소비자는 데이터의 출처·용도·신뢰성을 알 수 없고, 낙관적으로 가정해 변경 시 충격이 커진다.
- 생성자는 자신이 데이터 생성자임을 모르는 경우가 많아 변경 책임을 지우기 어렵다.
- 생성자와 소비자 간 협업·소통이 거의 없어 책임 소재가 불명확하다.
- 신뢰성 결핍(lack of reliability): 소비용으로 만들어지지 않은 데이터를 기반으로 해 파이프라인이 자주 깨진다.
- 스키마 변경은 시끄러운 실패(loud failure), 데이터/로직 변경은 조용한 실패(silent failure)를 일으킨다.
- 임시방편 CASE/IFNULL 패치가 쌓여 파이프라인이 점점 복잡·취약해지고, 사용자는 신뢰를 잃는다.
- 자율성 결핍(lack of autonomy): 중앙팀이 병목이 되어 KPI 외 요청은 백로그에 방치되고, 가치 있는 데이터가 미사용으로 남는다.

```
gen 1 (EDW) | gen 2 (Big Data) | gen 3 (MDS)
   [bottleneck]   [bottleneck]     [bottleneck]  <- same central-team bottleneck
```

---

## 5. The Ever-Increasing Use of Data in Business-Critical Applications

> 데이터가 내부 KPI를 넘어 외부 고객 대상 핵심 제품에까지 쓰이면서, 플랫폼 신뢰성 향상이 필수가 되었다.

- 데이터를 효과적으로 활용하는 조직은 산업 전반에서 실질적 경쟁우위를 얻는다.
- McKinsey: 상위 25개 리테일러(디지털 리더)는 83% 더 높은 수익성을 보였다.
- 더 많은 데이터가 조직 전반에 접근 가능해야 데이터 프로젝트가 성공한다.
- Anmut(2021): 리더 91%가 데이터를 사업 성공에 필수로 보지만, 34%만이 다른 자산처럼 규율 있게 관리한다.
- 규율 없는 관리(lack of discipline)가 파이프라인 전반의 기대치 부재로 이어진다.
- 이제 내부 보고뿐 아니라 매출 창출 서비스와 외부 고객 신뢰까지 영향을 받아 브랜드 손상 위험이 커진다.

---

## Summary (핵심 정리)

- 도구는 발전했지만 데이터 플랫폼 아키텍처는 진화하지 못해, 접근성을 제한하는 병목과 낮은 신뢰성이 세대를 거쳐 지속되었다.
- 근본 원인은 '소비를 위해 만들어지지 않은' 원천 데이터와, 기대치·책임을 명시하지 않는 데이터 문화에 있다.
- 다음 장에서는 이를 해결하기 위한 새로운 아키텍처 패턴인 데이터 컨트랙트(data contracts)를 소개한다.
