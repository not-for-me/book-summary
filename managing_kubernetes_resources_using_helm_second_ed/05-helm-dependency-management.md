# 05. Helm Dependency Management

## 챕터 개요 (3줄 요약)
- parent chart가 다른 chart(예: Redis, MariaDB)를 dependency로 선언해 백엔드를 손쉽게 배포하는 방법을 다룬다.
- Chart.yaml의 dependencies map 필드(name/repository/version/condition/tags/alias/import-values)와 helm dependency 서브커맨드를 설명한다.
- guestbook chart에 Redis dependency를 추가해 백엔드를 구성한다.

---

## 1. Declaring dependencies & the dependencies map
> dependency는 parent chart가 의존하는 다른 chart의 리소스를 함께 설치하며, Chart.yaml의 dependencies map에 선언한다.

필드:
- **name / repository / version** (필수).
- **condition** (boolean값으로 포함 토글), **tags** (label 기반 다수 토글).
- **alias** (동일 dependency 중복 시 고유 이름), **import-values** (값 propagation 단순화).

예(WordPress): mariadb(condition), memcached(condition=false 기본), common(library chart, tag).

---

## 2. Downloading dependencies
> helm dependency 서브커맨드로 dependency를 조회·다운로드하며, charts/에 .tgz가 받아지고 Chart.lock이 생성된다.

```
helm dependency list <chart>    # 선언된 dep + 상태(missing/ok)
helm dependency update <chart>  # Chart.yaml 기준 다운로드 + Chart.lock 생성
helm dependency build <chart>   # Chart.lock 기준 다운로드(버전 고정 재현)
```

- version에 wildcard(`9.x.x`) 사용 → 최신 minor/patch 다운로드. **Chart.lock**이 실제 받은 버전(예 9.8.1)을 고정.
- update는 최신을 재조회(버전 변동 위험), build는 lock 버전을 그대로 재현.

---

## 3. Conditional dependencies (condition & tags)
> condition은 `chartname.enabled` 단일 토글, tags는 여러 dependency를 공통 label로 묶어 토글한다.

```yaml
# condition
condition: mariadb.enabled         # value true/false로 포함 제어
# tags
tags: [backend, database]          # backend: true면 해당 dep 포함
```

- 값 없으면 기본 포함. condition은 comma-list 가능(첫 존재값 우선).
- tag 하나라도 true면 포함. condition과 tags 병용 시 **condition이 tags를 override**.

---

## 4. Altering names & values
> dependency 값은 dependency 이름을 root로 한 map으로 override하며, alias로 중복 dependency를 구분하고 import-values로 값 전파를 단순화한다.

```yaml
mariadb:                    # dependency 이름 root로 값 override
  image: { tag: latest }
```
- **alias**: 동일 chart 여러 번 선언 시 db1/db2처럼 고유 이름 부여 → 각각 값 override.
- **import-values**: exports format / child-parent format으로 깊은 nested 값을 parent로 평탄하게 가져옴. 단 import한 값은 override 불가.

---

## 5. Updating the guestbook chart (Redis dependency)
> Artifact Hub에서 Bitnami Redis chart를 찾아 condition·wildcard version으로 dependency 선언 후 helm dependency update로 받아 설치한다.

```yaml
dependencies:
- name: redis
  repository: https://raw.githubusercontent.com/bitnami/charts/archive-full-index/bitnami
  version: 15.5.x
  condition: redis.enabled
```
```
helm dependency update guestbook
helm install guestbook guestbook -n chapter5
kubectl get statefulsets -n chapter5   # guestbook-redis-master/replicas
```

---

## Summary (핵심 정리)
- dependency는 복잡한 백엔드(Redis/MariaDB)를 5줄 YAML로 재사용 설치하게 해 개발 노력을 크게 줄인다.
- 선언은 name/version/repository, 토글은 condition·tags, 중복은 alias, 복잡 값은 import-values로 처리한다.
- helm dependency update(최신 다운로드+lock 생성) vs build(lock 버전 재현)로 버전을 관리한다.
- guestbook은 redis.enabled condition + wildcard version으로 Redis dependency를 추가해 백엔드를 구성했다.
