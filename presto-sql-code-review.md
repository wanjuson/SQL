---
name: presto-sql-code-review
description: >
  Presto(Trino) SQL 쿼리에 대한 종합 코드 리뷰 가이드. 특히 AI가 생성한
  쿼리를 머지하기 전에 정확성, 보안, 성능, 스키마 변경 리스크를 분산
  쿼리 엔진 관점에서 점검합니다.
---

# Presto SQL 코드 리뷰 체크리스트

Presto(Trino)는 분산 쿼리 엔진이라는 특성상 일반 RDBMS와는 다른 관점의
리뷰가 필요합니다. 특히 AI가 빠르게 작성한 쿼리는 문법적으로는 맞아
보여도 조인 키, NULL 처리, 파티션 프루닝 같은 부분에서 조용히 틀린
결과를 낼 수 있으므로, 머지 전 아래 항목을 기준으로 반드시 점검하세요.

---

## ✅ 정확성 검증

### 조인 키 & 카디널리티

```sql
-- ❌ 위험: 조인 조건 누락 또는 의도치 않은 CROSS JOIN
SELECT a.*, b.*
FROM orders a, order_items b;

-- ✅ 명시적 조인 조건
SELECT a.*, b.*
FROM orders a
JOIN order_items b ON a.order_id = b.order_id;
```

- 조인 키가 실제로 1:1 / 1:N 관계에 맞는지 확인 (의도치 않은 row 증폭 없는지)
- `CROSS JOIN`이 명시적으로 의도된 것인지, 실수로 조인 조건이 빠진 것은 아닌지 확인

### NULL 처리

```sql
-- ❌ 위험: Presto에서 `= NULL`은 항상 NULL(= 거짓)로 평가되어 조건이 무의미해짐
SELECT * FROM users WHERE deleted_at = NULL;

-- ✅ 올바른 NULL 비교
SELECT * FROM users WHERE deleted_at IS NULL;
SELECT COALESCE(discount, 0) AS discount FROM orders;
```

- `= NULL` / `!= NULL` 대신 `IS NULL` / `IS NOT NULL` 사용 여부 확인
- `COALESCE`, `IFNULL` 등으로 NULL을 의도적으로 처리했는지, 아니면 실수로 방치했는지 확인

### 집계 & GROUP BY

- `SELECT` 절의 비집계 컬럼이 `GROUP BY`에 모두 포함되어 있는지 확인 (Presto는 표준 SQL을 엄격히 따르므로 누락 시 에러가 나지만, `GROUP BY`에 엉뚱한 컬럼을 넣어 의도와 다른 그룹핑이 되는 경우는 에러 없이 통과되니 주의)
- `HAVING`과 `WHERE`의 역할이 혼동되지 않았는지 (그룹핑 전/후 필터링 구분)

### 정렬 & 페이지네이션 안정성

```sql
-- ❌ 위험: 분산 실행 특성상 ORDER BY 없이는 결과 순서가 매 실행마다 달라질 수 있음
SELECT * FROM events LIMIT 100 OFFSET 100;

-- ✅ 고유 컬럼 기준 정렬로 순서 고정
SELECT * FROM events
ORDER BY event_id
LIMIT 100 OFFSET 100;
```

- Presto는 여러 워커에서 병렬로 데이터를 처리하므로, `ORDER BY` 없이는 동일 쿼리를 재실행해도 결과 순서가 보장되지 않음 (단일 노드 DB와의 큰 차이점)
- 페이지네이션에 사용하는 정렬 기준 컬럼이 **고유값**인지 확인 (동일 값이 여러 행에 걸치면 페이지 경계에서 행 누락/중복 발생)

### 중첩 타입(UNNEST) 처리

```sql
-- ❌ 위험: CROSS JOIN UNNEST는 배열이 비어있거나 NULL이면 해당 행 자체가 사라짐
SELECT u.id, t.tag
FROM users u
CROSS JOIN UNNEST(u.tags) AS t(tag);

-- ✅ 빈 배열/NULL이어도 원본 행을 보존하려면 LEFT JOIN UNNEST 사용
SELECT u.id, t.tag
FROM users u
LEFT JOIN UNNEST(u.tags) AS t(tag) ON TRUE;
```

- `ARRAY`, `MAP` 컬럼을 `UNNEST`할 때 `CROSS JOIN`과 `LEFT JOIN`의 결과 행 수 차이를 의도했는지 확인 (빈 배열 행이 통째로 사라지는 것을 놓치기 쉬움)

---

## 🔒 보안 분석

### SQL 인젝션 방지

```sql
-- ❌ 위험: 문자열 결합으로 쿼리를 생성하는 패턴
query = "SELECT * FROM users WHERE id = " + userInput;
query = f"SELECT * FROM orders WHERE user_id = {user_id}";

-- ✅ 안전: 파라미터 바인딩 사용 (JDBC PreparedStatement 등)
SELECT * FROM users WHERE id = ?;
```

- Presto JDBC/ODBC 드라이버 사용 시 반드시 **PreparedStatement**로 파라미터를 바인딩할 것
- 애플리케이션 레벨에서 쿼리를 동적으로 조립하는 경우, 사용자 입력값이 컬럼명·테이블명·정렬 방향 등 **식별자(identifier)** 자리에 그대로 들어가지 않는지 확인 (파라미터 바인딩으로 막을 수 없는 영역)
- 화이트리스트 방식으로 허용된 컬럼/테이블명만 매핑해서 사용

### 접근 제어 & 권한

- **커넥터별 권한 정책 확인**: Hive 커넥터는 HDFS/S3 ACL, Iceberg는 카탈로그 레벨 권한 등 데이터 소스에 따라 접근 제어 방식이 다름
- **카탈로그/스키마 단위 최소 권한**: 특정 카탈로그·스키마에만 접근 가능하도록 제한
- **뷰(View)를 통한 민감 컬럼 마스킹**: 원본 테이블 대신 마스킹 뷰를 경유하도록 설계했는지 확인
- **행 단위 필터링**: 멀티테넌트 환경이라면 `tenant_id`, `user_id` 등 필터가 누락되지 않았는지 점검

### 데이터 보호

- **민감 정보 노출**: `SELECT *` 사용 시 민감 컬럼(주민번호, 카드번호 등)이 결과에 그대로 포함되는지 확인
- **감사 로그**: 민감 테이블에 대한 쿼리 실행 이력이 로깅되는지 확인
- **외부 테이블 권한**: S3/GCS 등 외부 스토리지를 직접 참조하는 external table의 경우 스토리지 자체 접근 권한도 함께 점검

---

## ⚡ 성능 최적화 (Presto 특화)

### 파티션 프루닝 (Partition Pruning)

```sql
-- ❌ 나쁨: 파티션 컬럼에 함수를 씌워 파티션 프루닝이 동작하지 않음
SELECT * FROM events
WHERE date_format(event_date, '%Y-%m-%d') = '2026-09-01';

-- ✅ 좋음: 파티션 컬럼을 원본 그대로 조건에 사용
SELECT * FROM events
WHERE event_date = DATE '2026-09-01';
```

- 파티션 컬럼에 함수·형변환을 적용하면 파티션 프루닝이 깨져 전체 스캔이 발생함
- `WHERE` 절에 파티션 키가 반드시 포함되어 있는지 확인 (특히 대용량 테이블)

### 조인 전략

```sql
-- ❌ 나쁨: 큰 테이블끼리 브로드캐스트 조인을 유도하는 힌트 없는 조인
SELECT a.*, b.*
FROM huge_table a
JOIN another_huge_table b ON a.id = b.id;

-- ✅ 좋음: 작은 테이블을 broadcast, 큰 테이블끼리는 distributed join 유도
SELECT /*+ BROADCAST(small_table) */ a.*, b.*
FROM huge_table a
JOIN small_table b ON a.id = b.id;
```

- 작은 테이블(디멘전 테이블 등)과 큰 테이블(팩트 테이블) 조인 시 broadcast join이 되는지 확인
- 큰 테이블끼리 조인할 때는 distributed(hash) join으로 유도했는지, 조인 키의 데이터 분포가 치우쳐 있지 않은지(data skew) 확인
- `EXPLAIN` / `EXPLAIN ANALYZE`로 실제 조인 전략이 의도한 대로 동작하는지 확인 권장

### 컬럼 및 스캔 범위 최소화

- **`SELECT *` 지양**: 컬럼 기반 스토리지(ORC/Parquet)의 장점을 살리려면 필요한 컬럼만 명시
- **불필요한 서브쿼리/CTE 중복 스캔**: 동일 테이블을 여러 번 스캔하는 CTE 구조는 `WITH` 절 재사용이 최적화기에서 실제로 병합되는지 확인
- **`LIMIT` 없는 대량 결과 반환**: 탐색성 쿼리에서 `LIMIT`이 빠지지 않았는지 확인

### 데이터 타입 및 캐스팅

- **암묵적 캐스팅으로 인한 인덱스/파티션 무효화**: `VARCHAR`와 `BIGINT` 비교처럼 타입이 맞지 않아 내부적으로 캐스팅이 발생하는 조건은 없는지 확인
- **`ARRAY`, `MAP`, `ROW` 등 중첩 타입 처리**: `UNNEST` 사용 시 불필요하게 큰 중첩 구조를 통째로 펼치고 있지 않은지 확인

### 집계 및 윈도우 함수

- **`APPROX_DISTINCT` 활용**: 정확한 `COUNT(DISTINCT ...)`가 꼭 필요한 게 아니라면 대용량 데이터에서는 근사 함수 사용을 검토
- **윈도우 함수의 `PARTITION BY` 범위**: 파티션 범위가 지나치게 넓어 메모리 부담을 주지 않는지 확인

---

## 🔁 스키마 변경 & 대용량 테이블 작업 (Migrations)

Presto는 전통적인 RDBMS의 migration 개념은 없지만, `CTAS`/`INSERT OVERWRITE`로
테이블을 통째로 재작성하거나 Iceberg/Delta 테이블의 스키마를 변경하는
작업은 그에 준하는 리스크를 가지므로 동일한 기준으로 점검이 필요합니다.

- **되돌릴 수 있는가**: Iceberg 테이블이라면 스냅샷/타임트래블로 롤백이 가능한지, Hive 테이블이라면 원본 백업 없이 `INSERT OVERWRITE`를 실행하는 것은 아닌지 확인
- **스키마 변경의 하위 호환성**: 컬럼 추가/삭제/타입 변경이 기존에 이 테이블을 읽는 다른 쿼리·잡을 깨뜨리지 않는지 확인 (특히 컬럼 순서에 의존하는 레거시 쿼리 여부)
- **파티션 스킴 변경**: 파티션 컬럼이나 파티셔닝 방식을 바꾸는 경우, 기존 파티션 데이터까지 소급 적용되는지 아니면 신규 데이터부터만 적용되는지 명시했는지 확인 (Iceberg의 partition evolution은 안전하지만 Hive 스타일은 수동 재작성 필요)
- **락/리소스 경합**: 대용량 `CTAS`/`INSERT OVERWRITE`가 실행되는 동안 같은 테이블을 읽는 다른 쿼리에 영향을 주지 않는지, 클러스터 리소스를 과도하게 점유해 다른 쿼리를 지연시키지 않는지 확인
- **백필 방식**: 대량 백필 시 한 번에 전체 테이블을 재작성하기보다 파티션 단위로 나눠 점진적으로 처리했는지 확인

---

## 🛠 유지보수성 & 코드 스타일

- **CTE(`WITH` 절) 활용**: 복잡한 서브쿼리 중첩보다 `WITH` 절로 단계를 분리해 가독성 확보
- **일관된 네이밍 컨벤션**: 별칭(alias), 컬럼명 스타일이 팀 컨벤션과 일치하는지 확인
- **매직 넘버/하드코딩된 날짜**: 날짜, 임계값 등이 하드코딩되어 있다면 파라미터화 가능한지 검토
- **주석**: 복잡한 비즈니스 로직이 담긴 CASE WHEN, 조인 조건에는 의도를 설명하는 주석 권장

---

## 📋 요약 체크리스트

```txt
[ ] 조인 키가 올바르며 의도치 않은 CROSS JOIN이 없음
[ ] NULL 비교에 IS NULL/IS NOT NULL 사용 (= NULL 아님)
[ ] GROUP BY 컬럼이 SELECT 절과 정확히 대응됨
[ ] 페이지네이션이 고유 컬럼 기준 ORDER BY로 안정적임
[ ] UNNEST 시 CROSS JOIN/LEFT JOIN 선택이 빈 배열 처리 의도와 일치함
[ ] 파라미터 바인딩 사용 (문자열 결합 쿼리 없음)
[ ] 사용자 입력이 식별자(컬럼/테이블명)에 직접 들어가지 않음
[ ] 카탈로그/스키마 단위 최소 권한 원칙 준수
[ ] 멀티테넌트 환경에서 tenant_id/user_id 필터 누락 없음
[ ] SELECT * 대신 필요한 컬럼만 명시
[ ] 파티션 컬럼에 함수/캐스팅 없이 조건절 사용 (파티션 프루닝 확인)
[ ] 큰 테이블 간 조인 시 data skew 및 조인 전략(broadcast/distributed) 확인
[ ] 대량 결과 반환 시 LIMIT 적용
[ ] COUNT(DISTINCT) 대신 APPROX_DISTINCT 검토 (대용량 데이터)
[ ] EXPLAIN / EXPLAIN ANALYZE로 실행 계획 확인
[ ] 스키마 변경 시 하위 호환성 및 롤백 가능 여부 확인
[ ] 대용량 CTAS/INSERT OVERWRITE의 락/리소스 경합 영향 확인
[ ] 민감 컬럼 마스킹 및 감사 로그 적용 여부 확인
```

---

## 🔧 리뷰용 프롬프트 예시

```txt title="Fix Prompt"
아래 Presto SQL 쿼리를 위 체크리스트 기준으로 검토해줘.
조인 키와 NULL 처리, GROUP BY가 올바른지 먼저 확인하고,
파티션 프루닝이 깨지는 조건이 있다면 수정하고,
필요하다면 조인 힌트(BROADCAST 등)를 제안하고,
SELECT *를 필요한 컬럼으로 변경한 최종 쿼리만 반환해줘.
```
