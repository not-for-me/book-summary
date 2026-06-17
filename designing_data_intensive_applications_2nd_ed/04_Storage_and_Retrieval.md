# 04. Storage and Retrieval

> Keywords: log-structured storage, hash index, SSTable/LSM-tree, memtable/compaction(size-tiered/leveled), Bloom filter, B-tree/WAL, write amplification, sequential vs random write, clustered/covering index, in-memory DB, column-oriented storage, bitmap encoding, vectorization/JIT, materialized view/data cube, multidimensional index(R-tree), full-text/inverted index, vector embedding(IVF/HNSW)

## 챕터 개요 (3줄 요약)
- DB의 본질은 '저장하고 다시 찾아주기' — index는 read를 빠르게 하지만 write를 느리게 하는 핵심 trade-off다.
- OLTP 엔진은 두 학파: append-only log-structured(LSM-tree, 높은 write throughput) vs update-in-place(B-tree, 빠른 read).
- analytics 엔진은 column-oriented storage + compression + vectorization/JIT으로 대량 scan을 최적화하며, 다차원/full-text/vector index로 고급 query를 지원한다.

---

## 1. Storage and Indexing for OLTP
> append는 가장 빠른 write지만 read는 O(n) — index(원본에서 파생된 추가 구조)로 read를 가속한다.

### Log & Hash Index
- `log`: append-only record 시퀀스(application log와 다른 일반적 의미). append는 매우 효율적.
- in-memory `hash index`: key→byte offset 매핑(Bitcask 방식). 빠르지만 한계: 디스크 공간 미회수, restart 시 재구축, hash가 메모리에 들어가야 함, range query 비효율.

### SSTable & LSM-tree
- `SSTable`(Sorted String Table): key로 정렬, key 유일. block 단위 + `sparse index`(일부 key만)로 메모리 절약, block compression.
- `LSM-tree` 흐름: write는 in-memory `memtable`(red-black tree/skip list)에 정렬 삽입 → 임계 초과 시 SSTable로 flush → background `compaction`(mergesort식 병합, 최신 값만 유지). crash 대비 별도 WAL.
- 삭제는 `tombstone` record append. RocksDB, Cassandra, ScyllaDB, HBase가 이 계열(Bigtable 논문 기원).
- segment는 immutable이라 crash recovery 단순, object storage에도 적합(SlateDB, Delta Lake).

```
write -> memtable (in-mem, sorted) --flush--> SSTable seg (immutable, on disk)
read  -> memtable -> newest seg -> ... -> oldest seg   (+Bloom filter, +WAL)
background: compaction merges segments, drops overwritten/tombstoned keys
```

### Bloom filter & Compaction strategy
- `Bloom filter`: key가 SSTable에 있는지 빠른 확률적 검사. false positive는 있어도 false negative는 없음 → 없으면 확실히 skip. 키당 10 bit면 1% 오탐.
- `size-tiered compaction`: 작은 SSTable을 큰 것으로 병합. write-heavy에 유리(쓰기 적게 재기록).
- `leveled compaction`: 고정 크기 SSTable을 L0,L1...로 계층화, key-range 분할. 디스크 절약·read-heavy에 유리.
- embedded storage engine(RocksDB, SQLite, LMDB, DuckDB): 네트워크 API 없이 app process 내 라이브러리. multitenant에 tenant별 인스턴스 가능.

### B-Tree
- 1970년 등장, 거의 모든 relational DB의 표준 index. 고정 크기 `page`(4~16 KiB)로 분할, in-place 덮어쓰기.
- page는 page number로 참조(디스크 포인터), tree 구성. root→중간→leaf로 key 탐색. `branching factor`(보통 수백)로 depth O(log n)(3~4 level로 250TB).
- 삽입 시 공간 부족하면 page split, 부모로 전파(루트까지). balanced 유지.
- 신뢰성: `WAL`(write-ahead log) — 모든 수정을 tree 적용 전 append. crash 후 복구. `torn page` 대비. fsync로 durability.
- 변형: copy-on-write(LMDB), key 축약, leaf 간 sibling 포인터.

### B-Tree vs LSM-Tree
- 일반: LSM은 write-heavy, B-tree는 read에 빠르고 예측가능(워크로드별 벤치마크 필수).
- `sequential vs random write`: B-tree는 random write(흩어진 page 덮어쓰기), LSM은 sequential(큰 segment). 디스크는 sequential이 빠름(SSD GC/wear도 random이 불리).
- `write amplification`: 1 write가 디스크에 여러 I/O로. LSM(log+memtable flush+compaction), B-tree(WAL+page, 부분 변경도 전체 page). LSM이 보통 낮음(write-heavy 유리). SSD wear에도 영향.
- 디스크 공간: B-tree는 fragmentation(vacuum 필요), LSM은 compaction으로 재기록 + compression으로 작음. LSM은 삭제가 compaction 전까지 잔존(규제 삭제 주의), immutable segment는 snapshot에 유리.

### 인덱스 종류 & In-Memory
- `secondary index`: key가 유일하지 않음 — postings list 또는 row id 추가로 해결.
- `clustered index`(데이터를 index에 직접 저장, InnoDB PK) vs reference(heap file) vs `covering index`(일부 column 포함, index만으로 query 응답).
- in-memory DB(VoltDB, SingleStore, Redis): 빠른 이유는 디스크를 안 읽어서가 아니라 디스크용 인코딩 overhead를 없애서. durability는 log/snapshot/replication으로.

---

## 2. Data Storage for Analytics
> 대량 row를 scan·aggregate하는 read에 최적화. column-oriented + compression + vectorization.

### Cloud Data Warehouse & 컴포넌트 분해
- cloud DW(BigQuery, Redshift, Snowflake)는 storage/compute 분리로 elastic. object storage 기반.
- 과거 통합(Hive)이 컴포넌트로 분해: `query engine`(Trino, Presto, DataFusion), `storage format`(Parquet, ORC, Lance, Nimble), `table format`(Iceberg, Delta — insert/delete·time travel·transaction), `data catalog`(Polaris, Unity Catalog — 테이블 목록·governance).

### Column-Oriented Storage
- 행 전체를 모으는 row-oriented와 달리, 각 column 값을 함께 저장 → query가 필요한 column만 read.
- 모든 column이 같은 row 순서 유지 → k번째 항목을 모아 row 재구성. block(수천~수백만 row) 단위 저장.
- `bitmap encoding`: distinct 값이 적을 때 값별 bitmap + `run-length encoding`(roaring bitmap). WHERE IN은 bitwise OR, AND 조건은 bitwise AND로 매우 효율.
- 주의: column-oriented ≠ wide-column(column-family, Bigtable/HBase는 row-oriented).
- `sort order`: 행 단위로 정렬(첫 sort key가 compression 최강). date_key 우선 정렬로 date range query 가속.
- write: bulk import(ETL). log-structured로 in-memory row store에 모았다가 column 파일과 병합.

### Query Execution & 사전 집계
- 대량 scan은 디스크 I/O뿐 아니라 CPU도 병목 → 두 접근: `query compilation`(SQL→machine code, JIT/LLVM) vs `vectorized processing`(column을 batch로 연산, SIMD). 둘 다 sequential access·tight loop·compressed data 직접 연산으로 CPU 활용.
- `materialized view`(query 결과를 디스크에 복제) vs virtual view. `data cube`(OLAP cube): dimension별 집계 grid 사전계산 → 특정 query 초고속, 단 raw 유연성↓(price 같은 비차원은 불가).

---

## 3. Multidimensional and Full-Text Indexes
> 단일 attribute range를 넘어 여러 column 동시 query, 키워드 검색, 의미 검색.

### 다차원 & Full-text
- `concatenated index`(여러 필드 결합)는 (lat AND long) 동시 query 불가. `multidimensional index`(R-tree, Bkd-tree, space-filling curve)가 geospatial 등에 필요(PostGIS는 R-tree).
- `full-text search`: term이 곧 dimension. `inverted index`(term→문서 ID postings list, sparse bitmap). 두 term은 bitwise AND. Lucene(Elasticsearch/Solr)은 SSTable식 정렬 파일 병합.
- n-gram(trigram) index로 부분 문자열·정규식 검색. Lucene은 Levenshtein automaton으로 edit distance(오타) 검색.

### Vector Embeddings (semantic search)
- `semantic search`: 동의어·의도까지 이해(RAG의 핵심). embedding model(주로 LLM)이 문서를 `vector embedding`(부동소수 배열, 다차원 공간의 한 점)으로 변환. 의미 유사 문서는 공간상 가까움.
- 주의: vectorized processing의 vector(bit batch)와 embedding의 vector(좌표)는 다른 의미.
- 거리: cosine similarity, Euclidean distance. Word2Vec/BERT/GPT → 최근 multimodal.
- vector index: `flat`(정확하나 전수 비교, 느림), `IVF`(centroid 분할, probe 수로 정확도/속도 trade-off, 근사), `HNSW`(다층 graph traversal, 근사). Faiss, pgvector가 IVF/HNSW 지원.

```
query text -> embedding model -> query vector -> vector index (IVF/HNSW)
           -> nearest neighbors by cosine/Euclidean distance
```

---

## Summary (핵심 정리)
- OLTP 엔진은 log-structured(append-only, LSM-tree, 높은 write throughput)와 update-in-place(B-tree, 빠른 read) 두 학파로 나뉜다. index는 read↑ write↓의 trade-off.
- analytics 엔진은 column-oriented storage + compression(bitmap/run-length) + vectorization/JIT으로 대량 scan을 최소 I/O·CPU로 처리한다. data cube/materialized view는 반복 집계를 사전계산.
- 다차원 index(R-tree)는 geospatial, inverted index는 full-text, vector index(IVF/HNSW)는 의미 기반 semantic search/RAG를 지원한다.
- 다음 연결: Ch5에서 데이터를 바이트로 encoding하고 schema를 evolution하는 방법을 다룸.