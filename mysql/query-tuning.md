# MySQL 쿼리 튜닝 실전 가이드

## 1. 튜닝의 기본 원칙과 프로세스

### 1-1. 튜닝의 목표

**❌ 잘못된 접근법**
- 느릴 것이라는 '감'으로 추측
- 무작정 인덱스 추가
- 전체 테이블을 대상으로 튜닝

**✅ 올바른 접근법**
- **데이터 기반 식별**: APM, Slow Query Log로 실제 느린 쿼리를 식별
- **옵티마이저와의 협업**: 옵티마이저가 최적의 실행 계획을 세울 수 있도록 정확한 정보(인덱스, 통계) 제공
- **ROI 분석**: 인덱스 유지 비용(Write 성능 저하, 스토리지) 대비 성능 개선 효과 고려

### 1-2. 4단계 튜닝 프로세스

#### 1단계: 문제 정의 (Analyze)
```sql
-- Slow Query Log 확인
mysql> SET GLOBAL slow_query_log = 'ON';
mysql> SET GLOBAL long_query_time = 1;

-- 실행 계획 분석
mysql> EXPLAIN SELECT * FROM orders WHERE status = 'PENDING';
```

#### 2단계: 우선순위 결정 (Prioritize)
- **비즈니스 영향도**: 핵심 비즈니스 로직에 영향을 주는 쿼리 우선
- **호출 빈도**: 초당 실행 횟수가 많은 쿼리 우선
- **시스템 부하**: CPU, I/O 사용량이 높은 쿼리 우선

#### 3단계: 튜닝 실행 (Execute)
1. 인덱스 추가/수정
2. 쿼리 재작성
3. 고급 기법 적용 (파티셔닝, 캐싱 등)

#### 4단계: 결과 검증 (Verify)
- EXPLAIN 결과 비교
- API 응답 시간 측정
- 시스템 리소스 사용량 모니터링

---

## 2. 실행 계획 (EXPLAIN) 완벽 분석

### 2-1. 1순위 확인 항목 (심각한 문제 신호)

| 컬럼 | 값 | 의미 & 해결 방향 |
|------|----|--------------------|
| `type` | `ALL` | 🚨 **Full Table Scan** - 테이블 전체 스캔<br/>→ WHERE, JOIN 절에 인덱스 생성 시급 |
| `type` | `index` | ⚠️ **Full Index Scan** - 인덱스 전체 스캔<br/>→ 더 효율적인 복합 인덱스 필요 |
| `Extra` | `Using filesort` | 🚨 **별도 정렬 수행** - 인덱스 미사용<br/>→ ORDER BY 절 컬럼을 인덱스에 추가 |
| `Extra` | `Using temporary` | 🚨 **임시 테이블 생성** - 메모리/디스크 I/O<br/>→ GROUP BY 최적화 또는 인덱스 추가 |

### 2-2. 2순위 확인 항목 (튜닝의 방향성)

| 컬럼 | 의미 & 확인 사항 |
|------|-------------------|
| `type` | **이상적인 순서**: `system` > `const` > `eq_ref` > `ref` > `range`<br/>최소 `range` 이상을 목표 |
| `rows` | 예상 스캔 행 수 - 비정상적으로 크면 인덱스 비효율 또는 통계 오류 |
| `key` | 실제 사용된 인덱스 - `NULL`이면 옵티마이저가 Full Scan 선택 |
| `Extra` | `Using index` (커버링 인덱스), `Using where` 등 추가 정보 |

### 2-3. EXPLAIN 실전 예제

```sql
-- 나쁜 예: Full Table Scan
mysql> EXPLAIN SELECT * FROM orders WHERE DATE(created_at) = '2023-11-20';
+----+------+-------+------+------+---------+------+--------+-------------+
| id | type | table | key  | rows | filtered| Extra|
+----+------+-------+------+------+---------+------+--------+-------------+
|  1 | ALL  | orders| NULL |50000 |   100.00| Using where |
+----+------+-------+------+------+---------+------+--------+-------------+

-- 좋은 예: Range Scan
mysql> EXPLAIN SELECT * FROM orders 
       WHERE created_at >= '2023-11-20' AND created_at < '2023-11-21';
+----+-------+--------+---------------+------+---------+------+--------+-------+
| id | type  | table  | key           | rows | filtered| Extra |
+----+-------+--------+---------------+------+---------+------+--------+-------+
|  1 | range | orders | idx_created_at|  125 |   100.00| NULL  |
+----+-------+--------+---------------+------+---------+------+--------+-------+
```

---

## 3. 인덱스 설계 및 활용 전략

### 3-1. 인덱스 기본 원칙

#### 선택도(Selectivity)가 핵심
```sql
-- 선택도 계산: 특정 값으로 조회 시 반환되는 행의 비율
-- 선택도 = 조건을 만족하는 행 수 / 전체 행 수

-- 좋은 선택도 (0.01 = 1%)
SELECT COUNT(*) FROM users WHERE email = 'user@example.com'; -- 1건
SELECT COUNT(*) FROM users; -- 100건
-- 선택도 = 1/100 = 0.01

-- 나쁜 선택도 (0.8 = 80%)  
SELECT COUNT(*) FROM users WHERE gender = 'M'; -- 80건
SELECT COUNT(*) FROM users; -- 100건
-- 선택도 = 80/100 = 0.8
```

#### 카디널리티(Cardinality) 판단 기준
```sql
-- 카디널리티 확인
SELECT 
    COLUMN_NAME,
    CARDINALITY,
    TABLE_ROWS,
    ROUND(CARDINALITY/TABLE_ROWS, 2) as uniqueness_ratio
FROM INFORMATION_SCHEMA.STATISTICS s
JOIN INFORMATION_SCHEMA.TABLES t ON s.TABLE_NAME = t.TABLE_NAME
WHERE s.TABLE_NAME = 'users';
```

### 3-2. SARGable: 인덱스를 '탈 수 있는' 쿼리

| ❌ Bad (Non-SARGable) | ✅ Good (SARGable) | 이유 |
|------------------------|---------------------|------|
| `WHERE SUBSTRING(name, 1, 3) = '김'` | `WHERE name LIKE '김%'` | 컬럼에 함수 사용 |
| `WHERE price / 10 = 1000` | `WHERE price = 10000` | 컬럼에 연산 수행 |
| `WHERE DATE(created_at) = '2023-11-20'` | `WHERE created_at >= '2023-11-20' AND created_at < '2023-11-21'` | 컬럼에 함수 사용 |
| `WHERE column_varchar = 123` | `WHERE column_varchar = '123'` | 암시적 형변환 발생 |
| `WHERE age + 1 = 31` | `WHERE age = 30` | 컬럼에 연산 수행 |

#### SARGable 쿼리 변환 실습
```sql
-- ❌ Non-SARGable: 함수 사용
SELECT * FROM orders WHERE YEAR(created_at) = 2023;

-- ✅ SARGable: 범위 조건으로 변경
SELECT * FROM orders 
WHERE created_at >= '2023-01-01' AND created_at < '2024-01-01';

-- ❌ Non-SARGable: 컬럼 연산
SELECT * FROM products WHERE price * 0.9 = 900;

-- ✅ SARGable: 연산을 우변으로 이동
SELECT * FROM products WHERE price = 900 / 0.9;
```

### 3-3. 복합 인덱스: 컬럼 순서가 전부다

#### 순서 결정 3단계 원칙

1. **등치/범위 분리**: 등치(`=`, `IN`) 조건을 범위(`>`, `<`, `BETWEEN`, `LIKE 'text%'`) 조건보다 앞에
2. **카디널리티 정렬**: 등치 조건 그룹 내에서 카디널리티 높은 컬럼을 앞에
3. **정렬 컬럼 추가**: `ORDER BY` 컬럼을 맨 뒤에 추가하여 filesort 방지

#### 실전 예제
```sql
-- 쿼리 분석
SELECT * FROM orders 
WHERE status = 'ACTIVE' 
  AND user_id = 123 
  AND amount > 1000
ORDER BY created_at DESC;

-- 조건 분류
-- 등치 조건: status, user_id
-- 범위 조건: amount  
-- 정렬 조건: created_at

-- 카디널리티 확인 (예시)
-- user_id: 10,000개 (높음)
-- status: 5개 (낮음)  
-- amount: 1,000개 (중간)

-- 최적 인덱스 순서
CREATE INDEX idx_optimal ON orders (user_id, status, amount, created_at);
```

#### 복합 인덱스 활용 패턴
```sql
-- 인덱스: (A, B, C)가 있을 때 사용 가능한 패턴
-- ✅ 사용 가능
WHERE A = 1                    -- A만 사용
WHERE A = 1 AND B = 2          -- A, B 사용  
WHERE A = 1 AND B = 2 AND C = 3 -- A, B, C 모두 사용
WHERE A = 1 AND C = 3          -- A만 사용 (B 생략되어 C는 스캔)

-- ❌ 사용 불가
WHERE B = 2                    -- 선행 컬럼 A 없음
WHERE B = 2 AND C = 3          -- 선행 컬럼 A 없음
WHERE A > 1 AND B = 2          -- A가 범위 조건이므로 B는 비효율적
```

### 3-4. 연산자별 인덱스 효율 우선순위

| 우선순위 | 연산자 | 인덱스 활용 방식 | type |
|----------|--------|------------------|------|
| 1 (최상) | `=`, `IN` | 등치 조건 - 특정 지점 탐색 (Seek) | `const`, `eq_ref`, `ref` |
| 2 (차상) | `BETWEEN`, `>`, `<`, `LIKE 'text%'` | 범위 조건 - 시작/끝 지점 스캔 | `range` |
| 3 (나쁨) | `!=`, `<>`, `NOT IN` | 부정 조건 - 범위가 너무 넓음 | `ALL` or `index` |
| 4 (불가) | `LIKE '%text'`, `함수(컬럼)` | 인덱스 무력화 | `ALL` |

---

## 4. 고급 튜닝 전략 및 시나리오

### 4-1. InnoDB 내부 구조와 커버링 인덱스

#### InnoDB 스토리지 엔진 구조
```
클러스터링 인덱스 (Primary Key)
├── 리프 노드: 모든 컬럼 데이터 저장
└── 넌리프 노드: PK 값으로 구성

넌클러스터링 인덱스 (Secondary Index)  
├── 리프 노드: PK 값 저장
└── 넌리프 노드: 인덱스 키 값으로 구성
```

#### 일반적인 SELECT 동작 과정
```sql
-- 테이블 구조
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(100),
    age INT,
    INDEX idx_email (email)
);

-- 쿼리 실행
SELECT * FROM users WHERE email = 'user@example.com';

-- 내부 동작
-- 1단계: 2차 인덱스(idx_email)에서 PK 값 찾기 (1차 I/O)
-- 2단계: 찾은 PK로 클러스터링 인덱스에서 전체 데이터 조회 (2차 I/O)
```

#### 커버링 인덱스의 위력
```sql
-- ❌ 2차 I/O 발생 (일반 인덱스)
SELECT id, name, age FROM users WHERE email = 'user@example.com';
-- 실행계획: key: idx_email, Extra: NULL

-- ✅ 1차 I/O만 발생 (커버링 인덱스)
CREATE INDEX idx_email_covering ON users (email, name, age);
SELECT id, name, age FROM users WHERE email = 'user@example.com';
-- 실행계획: key: idx_email_covering, Extra: Using index

-- 성능 차이: 디스크 접근 50% 감소!
```

### 4-2. 실행 계획 변동성 대응

#### 원인 분석
- 대량 데이터 변경 후 통계 정보와 실제 데이터 분포 불일치
- 옵티마이저가 잘못된 통계로 비효율적인 실행 계획 선택

#### 단기 해결책
```sql
-- 1. 통계 정보 강제 갱신
ANALYZE TABLE orders;

-- 2. 옵티마이저 힌트 사용
-- USE INDEX: 권장
SELECT * FROM orders USE INDEX (idx_status_date) 
WHERE status = 'PENDING' AND created_at > '2023-01-01';

-- FORCE INDEX: 강제
SELECT * FROM orders FORCE INDEX (idx_status_date)
WHERE status = 'PENDING' AND created_at > '2023-01-01';

-- IGNORE INDEX: 특정 인덱스 제외
SELECT * FROM orders IGNORE INDEX (idx_wrong) 
WHERE status = 'PENDING';
```

#### 장기 해결책
```sql
-- 1. 주기적 통계 갱신 자동화 (cron 등)
-- 매일 새벽 2시 통계 갱신
0 2 * * * mysql -u root -p -e "ANALYZE TABLE mydb.orders;"

-- 2. MySQL 8+ 옵티마이저 힌트
SELECT /*+ INDEX(orders idx_status_date) */ 
FROM orders 
WHERE status = 'PENDING' AND created_at > '2023-01-01';

-- 3. 통계 정보 고정 (MySQL 8+)
SET PERSIST innodb_stats_persistent = 1;
SET PERSIST innodb_stats_auto_recalc = 0;
```

### 4-3. 다중 조인 튜닝 전략

#### 1단계: 조인 순서 최적화

```sql
-- ❌ 비효율적인 조인 순서
SELECT o.*, u.name, p.title
FROM orders o
JOIN users u ON o.user_id = u.id        -- users: 100만건
JOIN products p ON o.product_id = p.id  -- products: 10만건  
WHERE o.status = 'COMPLETED'             -- orders: 1000건으로 필터링
  AND o.created_at >= '2023-01-01';

-- ✅ 효율적인 조인 순서 (STRAIGHT_JOIN 사용)
SELECT STRAIGHT_JOIN o.*, u.name, p.title
FROM orders o                            -- 1. 가장 작은 결과셋부터
JOIN users u ON o.user_id = u.id         -- 2. 필터링된 orders와 조인
JOIN products p ON o.product_id = p.id   -- 3. 마지막에 products 조인
WHERE o.status = 'COMPLETED'
  AND o.created_at >= '2023-01-01';

-- 인덱스 전략
CREATE INDEX idx_orders_filter ON orders (status, created_at, user_id, product_id);
CREATE INDEX idx_users_id ON users (id);
CREATE INDEX idx_products_id ON products (id);
```

#### 2단계: 서브쿼리 → JOIN 변환

```sql
-- ❌ 상관 서브쿼리 (DEPENDENT SUBQUERY)
SELECT u.name, u.email
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.user_id = u.id 
      AND o.status = 'COMPLETED'
);

-- ✅ JOIN으로 변환
SELECT DISTINCT u.name, u.email  
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.status = 'COMPLETED';

-- ❌ 스칼라 서브쿼리
SELECT 
    u.name,
    (SELECT COUNT(*) FROM orders WHERE user_id = u.id) as order_count
FROM users u;

-- ✅ LEFT JOIN + GROUP BY로 변환
SELECT 
    u.name,
    COALESCE(COUNT(o.id), 0) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
```

#### 3단계: 궁극의 해결책 - 역정규화

```sql
-- 상황: 복잡한 실시간 조인 쿼리의 물리적 한계 도달
-- 원본 쿼리 (5초 소요)
SELECT 
    u.name,
    COUNT(o.id) as order_count,
    SUM(oi.quantity * oi.price) as total_amount,
    AVG(r.rating) as avg_rating
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
LEFT JOIN order_items oi ON o.id = oi.order_id  
LEFT JOIN reviews r ON u.id = r.user_id
WHERE u.created_at >= '2023-01-01'
GROUP BY u.id;

-- ✅ 해결: 집계 테이블 생성
CREATE TABLE user_statistics (
    user_id INT PRIMARY KEY,
    user_name VARCHAR(100),
    order_count INT DEFAULT 0,
    total_amount DECIMAL(15,2) DEFAULT 0,
    avg_rating DECIMAL(3,2) DEFAULT 0,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_last_updated (last_updated)
);

-- 배치로 집계 테이블 갱신 (매시간)
INSERT INTO user_statistics (user_id, user_name, order_count, total_amount, avg_rating)
SELECT 
    u.id,
    u.name,
    COALESCE(stats.order_count, 0),
    COALESCE(stats.total_amount, 0),
    COALESCE(stats.avg_rating, 0)
FROM users u
LEFT JOIN (
    SELECT 
        user_id,
        COUNT(DISTINCT o.id) as order_count,
        SUM(oi.quantity * oi.price) as total_amount,
        AVG(r.rating) as avg_rating
    FROM orders o
    JOIN order_items oi ON o.id = oi.order_id
    LEFT JOIN reviews r ON o.user_id = r.user_id  
    GROUP BY user_id
) stats ON u.id = stats.user_id
ON DUPLICATE KEY UPDATE
    order_count = VALUES(order_count),
    total_amount = VALUES(total_amount),
    avg_rating = VALUES(avg_rating);

-- 실시간 조회 (0.01초)
SELECT * FROM user_statistics WHERE user_id = 123;
```

---

## 5. 성능 모니터링 및 도구 활용

### 5-1. Slow Query Log 설정 및 분석

```sql
-- Slow Query Log 활성화
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
SET GLOBAL long_query_time = 1;  -- 1초 이상 걸리는 쿼리
SET GLOBAL log_queries_not_using_indexes = 'ON';  -- 인덱스 미사용 쿼리도 로깅

-- 설정 확인
SHOW VARIABLES LIKE 'slow%';
SHOW VARIABLES LIKE 'long_query_time';
```

#### mysqldumpslow를 이용한 분석
```bash
# 가장 느린 쿼리 10개
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# 가장 많이 실행된 쿼리 10개  
mysqldumpslow -s c -t 10 /var/log/mysql/slow.log

# 평균 실행 시간이 긴 쿼리 10개
mysqldumpslow -s at -t 10 /var/log/mysql/slow.log
```

### 5-2. Performance Schema 활용

```sql
-- Performance Schema 활성화 확인
SELECT * FROM performance_schema.setup_consumers 
WHERE NAME LIKE '%statement%';

-- 가장 느린 쿼리 TOP 10
SELECT 
    ROUND(SUM_TIMER_WAIT/1000000000000,6) as exec_time_s,
    COUNT_STAR as exec_count,
    ROUND(AVG_TIMER_WAIT/1000000000000,6) as avg_time_s,
    ROUND(SUM_ROWS_EXAMINED/COUNT_STAR,0) as avg_rows_examined,
    DIGEST_TEXT
FROM performance_schema.events_statements_summary_by_digest  
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- 테이블별 I/O 통계
SELECT 
    OBJECT_NAME as table_name,
    COUNT_READ as select_count,
    COUNT_WRITE as insert_update_delete_count,
    COUNT_READ + COUNT_WRITE as total_io
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA = 'your_database'
ORDER BY total_io DESC;
```

### 5-3. INFORMATION_SCHEMA 활용

```sql
-- 인덱스 사용률 분석
SELECT 
    t.TABLE_NAME,
    t.TABLE_ROWS,
    s.INDEX_NAME,
    s.COLUMN_NAME,
    s.CARDINALITY,
    ROUND(s.CARDINALITY/t.TABLE_ROWS*100, 2) as selectivity_percent
FROM INFORMATION_SCHEMA.STATISTICS s
JOIN INFORMATION_SCHEMA.TABLES t ON s.TABLE_NAME = t.TABLE_NAME
WHERE s.TABLE_SCHEMA = 'your_database'
  AND t.TABLE_SCHEMA = 'your_database'
ORDER BY selectivity_percent DESC;

-- 테이블 크기 및 인덱스 크기
SELECT 
    TABLE_NAME,
    ROUND(DATA_LENGTH/1024/1024, 2) as data_size_mb,
    ROUND(INDEX_LENGTH/1024/1024, 2) as index_size_mb,
    ROUND((DATA_LENGTH + INDEX_LENGTH)/1024/1024, 2) as total_size_mb,
    TABLE_ROWS
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'your_database'
ORDER BY total_size_mb DESC;
```

---

## 6. 실무 시나리오별 튜닝 전략

### 6-1. 페이징 쿼리 최적화

#### ❌ 비효율적인 OFFSET 페이징
```sql
-- 100만번째 페이지 조회 시 매우 느림
SELECT * FROM orders 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 2000000;  -- 200만건을 스캔 후 20건만 반환
```

#### ✅ 커서 기반 페이징 (Cursor Pagination)
```sql
-- 첫 페이지
SELECT * FROM orders 
WHERE created_at <= NOW()
ORDER BY created_at DESC 
LIMIT 21;  -- 20건 + 다음 페이지 존재 여부 확인용 1건

-- 다음 페이지 (마지막 created_at 값 사용)
SELECT * FROM orders 
WHERE created_at < '2023-11-20 15:30:00'  -- 이전 페이지의 마지막 값
ORDER BY created_at DESC 
LIMIT 21;

-- 필요 인덱스
CREATE INDEX idx_created_at ON orders (created_at);
```

#### ✅ 복합 키를 이용한 안정적인 페이징
```sql
-- ID와 created_at를 함께 사용 (동일 시간 데이터 처리)
SELECT * FROM orders 
WHERE (created_at < '2023-11-20 15:30:00') 
   OR (created_at = '2023-11-20 15:30:00' AND id < 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- 복합 인덱스
CREATE INDEX idx_created_at_id ON orders (created_at DESC, id DESC);
```

### 6-2. 집계 쿼리 최적화

#### COUNT(*) 최적화
```sql
-- ❌ 느린 COUNT
SELECT COUNT(*) FROM orders WHERE status = 'PENDING';

-- ✅ 커버링 인덱스 활용
CREATE INDEX idx_status ON orders (status);
-- 이제 COUNT(*)가 인덱스만으로 처리됨 (Using index)

-- 대용량 테이블의 근사치 COUNT
SELECT TABLE_ROWS 
FROM INFORMATION_SCHEMA.TABLES 
WHERE TABLE_NAME = 'orders';
```

#### GROUP BY 최적화
```sql
-- ❌ Using temporary, Using filesort 발생
SELECT user_id, COUNT(*) as order_count
FROM orders 
WHERE created_at >= '2023-01-01'
GROUP BY user_id
ORDER BY order_count DESC;

-- ✅ 최적화된 인덱스
CREATE INDEX idx_date_user ON orders (created_at, user_id);

-- 더 나은 방법: 커버링 인덱스
CREATE INDEX idx_covering_group ON orders (created_at, user_id, id);
SELECT user_id, COUNT(*) as order_count
FROM orders 
WHERE created_at >= '2023-01-01'
GROUP BY user_id
ORDER BY user_id;  -- 인덱스 순서를 따라 정렬
```

### 6-3. 실시간 랭킹/TOP N 쿼리 최적화

#### ❌ 매번 정렬하는 비효율적인 방식
```sql
-- 실시간 인기 상품 TOP 10 (매우 느림)
SELECT p.name, COUNT(*) as view_count
FROM product_views pv
JOIN products p ON pv.product_id = p.id
WHERE pv.created_at >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY pv.product_id, p.name
ORDER BY view_count DESC
LIMIT 10;
```

#### ✅ 집계 테이블을 활용한 최적화
```sql
-- 시간별 집계 테이블 생성
CREATE TABLE product_hourly_stats (
    product_id INT,
    hour_key VARCHAR(13),  -- '2023-11-20-15' 형태
    view_count INT DEFAULT 0,
    PRIMARY KEY (product_id, hour_key),
    INDEX idx_hour_views (hour_key, view_count DESC)
);

-- 배치로 집계 (매 10분마다)
INSERT INTO product_hourly_stats (product_id, hour_key, view_count)
SELECT 
    product_id,
    DATE_FORMAT(created_at, '%Y-%m-%d-%H') as hour_key,
    COUNT(*) as view_count
FROM product_views
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY product_id, hour_key
ON DUPLICATE KEY UPDATE view_count = VALUES(view_count);

-- 실시간 TOP 10 조회 (0.01초)
SELECT p.name, SUM(phs.view_count) as total_views
FROM product_hourly_stats phs
JOIN products p ON phs.product_id = p.id
WHERE phs.hour_key >= DATE_FORMAT(DATE_SUB(NOW(), INTERVAL 1 HOUR), '%Y-%m-%d-%H')
GROUP BY phs.product_id, p.name
ORDER BY total_views DESC
LIMIT 10;
```

### 6-4. 전문 검색 최적화

#### FULLTEXT 인덱스 활용
```sql
-- FULLTEXT 인덱스 생성
ALTER TABLE articles ADD FULLTEXT(title, content);

-- ❌ LIKE를 이용한 검색 (매우 느림)
SELECT * FROM articles 
WHERE title LIKE '%MySQL%' OR content LIKE '%MySQL%';

-- ✅ FULLTEXT 검색 (빠름)
SELECT *, MATCH(title, content) AGAINST('MySQL' IN BOOLEAN MODE) as relevance
FROM articles 
WHERE MATCH(title, content) AGAINST('MySQL' IN BOOLEAN MODE)
ORDER BY relevance DESC;

-- 고급 FULLTEXT 검색
SELECT * FROM articles 
WHERE MATCH(title, content) AGAINST('+MySQL -Oracle' IN BOOLEAN MODE);  -- MySQL 포함, Oracle 제외
```

---

## 7. 고급 MySQL 기능 활용

### 7-1. 파티셔닝을 통한 대용량 데이터 처리

#### 날짜 기반 파티셔닝
```sql
-- 월별 파티셔닝
CREATE TABLE orders (
    id INT AUTO_INCREMENT,
    user_id INT,
    created_at DATETIME,
    status VARCHAR(20),
    amount DECIMAL(10,2),
    PRIMARY KEY (id, created_at),
    INDEX idx_user_status (user_id, status)
) PARTITION BY RANGE (YEAR(created_at) * 100 + MONTH(created_at)) (
    PARTITION p202301 VALUES LESS THAN (202302),
    PARTITION p202302 VALUES LESS THAN (202303),
    PARTITION p202303 VALUES LESS THAN (202304),
    -- ... 추가 파티션들
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- 파티션 프루닝 확인
EXPLAIN PARTITIONS 
SELECT * FROM orders 
WHERE created_at BETWEEN '2023-01-01' AND '2023-01-31';
-- partitions: p202301 (하나의 파티션만 스캔)
```

#### 해시 파티셔닝 (균등 분산)
```sql
-- user_id 기반 해시 파티셔닝
CREATE TABLE user_activities (
    id BIGINT AUTO_INCREMENT,
    user_id INT,
    activity_type VARCHAR(50),
    created_at TIMESTAMP,
    PRIMARY KEY (id, user_id),
    INDEX idx_user_created (user_id, created_at)
) PARTITION BY HASH(user_id) PARTITIONS 8;
```

### 7-2. JSON 데이터 최적화

#### JSON 컬럼 인덱싱
```sql
-- JSON 컬럼이 있는 테이블
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    attributes JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- JSON 경로에 가상 컬럼 생성 후 인덱싱
ALTER TABLE products 
ADD COLUMN brand VARCHAR(100) GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.brand'))) STORED;

CREATE INDEX idx_brand ON products (brand);

-- 최적화된 쿼리
SELECT * FROM products WHERE brand = 'Apple';  -- 인덱스 사용

-- 함수형 인덱스 (MySQL 8.0+)
CREATE INDEX idx_json_category ON products ((CAST(JSON_EXTRACT(attributes, '$.category') AS CHAR(50))));

SELECT * FROM products 
WHERE CAST(JSON_EXTRACT(attributes, '$.category') AS CHAR(50)) = 'Electronics';
```

### 7-3. Common Table Expressions (CTE) 활용

#### 복잡한 계층 쿼리 최적화
```sql
-- 재귀 CTE로 조직 구조 조회
WITH RECURSIVE org_hierarchy AS (
    -- 앵커: 최상위 부서
    SELECT id, name, parent_id, 1 as level, CAST(name AS CHAR(1000)) as path
    FROM departments 
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- 재귀: 하위 부서들
    SELECT d.id, d.name, d.parent_id, oh.level + 1, 
           CONCAT(oh.path, ' > ', d.name)
    FROM departments d
    JOIN org_hierarchy oh ON d.parent_id = oh.id
    WHERE oh.level < 10  -- 무한 재귀 방지
)
SELECT * FROM org_hierarchy ORDER BY path;

-- 인덱스 최적화
CREATE INDEX idx_parent_id ON departments (parent_id, id, name);
```

### 7-4. 윈도우 함수를 활용한 분석 쿼리

```sql
-- 사용자별 주문 순서와 누적 금액
SELECT 
    user_id,
    order_id,
    amount,
    created_at,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at) as order_sequence,
    SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) as cumulative_amount,
    LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) as prev_amount
FROM orders
WHERE created_at >= '2023-01-01';

-- 성능 최적화를 위한 인덱스
CREATE INDEX idx_user_created_amount ON orders (user_id, created_at, amount, order_id);
```

---

## 8. 핵심 질문 정리

### 8-1. 자주 나오는 질문과 답변

#### Q1: "인덱스란 무엇이고, 어떻게 동작하나요?"

**A:** 인덱스는 테이블 데이터의 물리적 위치를 가리키는 포인터들을 정렬된 구조로 저장한 객체입니다.

**핵심 포인트:**
- **B+Tree 구조**: 균형잡힌 트리로 O(log N) 시간복잡도 보장
- **클러스터링 vs 넌클러스터링**: InnoDB에서 PK는 클러스터링, 나머지는 넌클러스터링
- **Trade-off**: 조회 성능 향상 vs INSERT/UPDATE/DELETE 성능 저하

```sql
-- 예시로 설명
CREATE TABLE users (id INT PRIMARY KEY, email VARCHAR(255), name VARCHAR(100));
CREATE INDEX idx_email ON users (email);

-- 쿼리 실행 시
SELECT * FROM users WHERE email = 'user@example.com';
-- 1. idx_email에서 'user@example.com' 검색 (B+Tree 탐색)
-- 2. 찾은 레코드에서 PK 값 추출
-- 3. PK로 클러스터링 인덱스에서 전체 데이터 조회
```

#### Q2: "복합 인덱스의 컬럼 순서는 어떻게 정하나요?"

**A:** 다음 3단계 원칙을 따릅니다:

1. **등치 조건을 범위 조건보다 앞에**
2. **등치 조건 내에서 카디널리티가 높은 순서로**
3. **ORDER BY 컬럼을 맨 뒤에**

```sql
-- 쿼리: WHERE status='ACTIVE' AND user_id=123 AND date > '2023-01-01' ORDER BY created_at
-- 분석:
-- 등치: status, user_id  
-- 범위: date
-- 정렬: created_at
-- 카디널리티: user_id(높음) > status(낮음)

-- 최적 인덱스
CREATE INDEX idx_optimal ON orders (user_id, status, date, created_at);
```

#### Q3: "EXPLAIN에서 가장 먼저 확인해야 할 것은?"

**A:** type 컬럼에서 `ALL`과 Extra 컬럼에서 `Using filesort`, `Using temporary`를 확인합니다.

**우선순위별 확인사항:**
1. **type = ALL**: Full Table Scan → 인덱스 추가 필요
2. **Using filesort**: 정렬을 위한 별도 처리 → ORDER BY 인덱스 필요
3. **Using temporary**: 임시 테이블 생성 → GROUP BY 최적화 필요
4. **rows 값**: 예상 스캔 행 수가 너무 큰지 확인

#### Q4: "쿼리가 느린 원인을 어떻게 찾나요?"

**A:** 체계적인 4단계 접근법을 사용합니다:

1. **문제 식별**: Slow Query Log, APM으로 실제 느린 쿼리 찾기
2. **실행계획 분석**: EXPLAIN으로 병목 지점 확인
3. **인덱스 검토**: 필요한 인덱스가 있는지, 올바르게 사용되는지 확인
4. **쿼리 재작성**: SARGable하지 않은 조건이 있는지 점검

```sql
-- 단계별 예시
-- 1. Slow Query에서 발견된 쿼리
SELECT * FROM orders WHERE DATE(created_at) = '2023-11-20';

-- 2. EXPLAIN 결과: type=ALL, rows=100000
-- 3. 문제: DATE() 함수로 인한 인덱스 미사용
-- 4. 해결: SARGable 쿼리로 변경
SELECT * FROM orders 
WHERE created_at >= '2023-11-20' AND created_at < '2023-11-21';
```

### 8-2. 주의사항과 안티패턴

#### 하지 말아야 할 것들

**❌ 무작정 인덱스 추가**
```sql
-- 잘못된 접근: 모든 컬럼에 인덱스
CREATE INDEX idx_col1 ON table1 (col1);
CREATE INDEX idx_col2 ON table1 (col2);
CREATE INDEX idx_col3 ON table1 (col3);
-- 문제: INSERT/UPDATE 성능 저하, 불필요한 스토리지 사용
```

**❌ 인덱스를 무력화하는 쿼리**
```sql
-- Function 사용
WHERE UPPER(name) = 'JOHN'        -- ❌
WHERE name = 'JOHN'               -- ✅

-- 암시적 형변환
WHERE varchar_column = 123        -- ❌
WHERE varchar_column = '123'      -- ✅

-- 선행 와일드카드
WHERE name LIKE '%john%'          -- ❌  
WHERE name LIKE 'john%'           -- ✅
```

**❌ 비효율적인 조인**
```sql
-- 카르테시안 곱 발생
SELECT * FROM orders o, users u 
WHERE o.amount > 1000 AND u.status = 'ACTIVE';  -- ❌ 조인 조건 없음

-- 올바른 조인
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.amount > 1000 AND u.status = 'ACTIVE';  -- ✅
```

---

## 9. 실전 체크리스트

### 9-1. 튜닝 전 체크리스트

- [ ] Slow Query Log 확인 완료
- [ ] 대상 쿼리의 비즈니스 중요도 파악
- [ ] 현재 실행계획 분석 완료
- [ ] 테이블 통계 정보 최신 여부 확인
- [ ] 기존 인덱스 사용률 분석 완료

### 9-2. 인덱스 설계 체크리스트

- [ ] WHERE 절의 모든 조건 컬럼 고려
- [ ] 복합 인덱스 컬럼 순서 최적화
- [ ] ORDER BY 절 고려한 인덱스 설계
- [ ] 커버링 인덱스 적용 가능성 검토
- [ ] 기존 인덱스와의 중복성 확인

### 9-3. 쿼리 작성 체크리스트

- [ ] 모든 조건이 SARGable한지 확인
- [ ] 불필요한 SELECT * 사용하지 않음
- [ ] 조인 조건 누락 없음
- [ ] 서브쿼리를 JOIN으로 변환할 수 있는지 검토
- [ ] LIMIT 절 활용하여 불필요한 데이터 조회 방지

### 9-4. 성능 검증 체크리스트

- [ ] 튜닝 전/후 EXPLAIN 결과 비교
- [ ] 실제 쿼리 실행 시간 측정
- [ ] 시스템 리소스 사용률 모니터링
- [ ] 동시성 테스트 수행
- [ ] 프로덕션 환경에서 A/B 테스트