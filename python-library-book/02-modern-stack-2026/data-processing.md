# 현대적 데이터 처리

## 2026 표준 스택

| 도구 | 등급 | 역할 |
|------|------|------|
| `polars` | Yes | 고성능 데이터프레임 |
| `duckdb` | Yes | 인메모리 분석 DB |
| `pyarrow` | Yes | 컬럼형 데이터 포맷 |

---

## 1. polars: 차세대 데이터프레임 ⭐

### 핵심 특징

- **10배 빠름**: pandas보다 월등한 성능
- **병렬 처리**: 멀티코어 자동 활용
- **메모리 효율**: Rust 기반

### 기본 사용

```python
import polars as pl

# DataFrame 생성
df = pl.DataFrame({
    "name": ["Alice", "Bob", "Charlie"],
    "age": [25, 30, 35],
    "city": ["NYC", "LA", "SF"]
})

# 데이터 조회
df.head()
df.filter(pl.col("age") > 25)
df.select(["name", "city"])
df.sort("age", descending=True)
```

### pandas와 비교

```python
# pandas
df_pandas = df.to_pandas()

# polars (더 빠름)
df_polars = pl.from_pandas(df_pandas)

# 체인 메서드 (동일)
result = (
    df
    .filter(pl.col("age") > 25)
    .group_by("city")
    .agg(pl.col("age").mean())
)
```

### 대용량 데이터

```python
# Lazy evaluation (메모리 효율적)
lazy_df = pl.scan_csv("large_file.csv")

result = (
    lazy_df
    .filter(pl.col("revenue") > 1000)
    .group_by("category")
    .agg([
        pl.col("revenue").sum().alias("total_revenue"),
        pl.col("id").count().alias("count")
    ])
    .collect()  # 여기서 실제 실행
)
```

---

## 2. duckdb: 인메모리 분석 DB ⭐

### 핵심 특징

- **SQL 쿼리**: 익숙한 SQL 문법
- **초고속**: 컬럼형 스토리지
- **pandas/polars 통합**

### 기본 사용

```python
import duckdb

# DataFrame 직접 쿼리
import polars as pl

df = pl.DataFrame({
    "product": ["A", "B", "A", "C"],
    "sales": [100, 200, 150, 300]
})

# SQL 쿼리
result = duckdb.query("""
    SELECT product, SUM(sales) as total
    FROM df
    GROUP BY product
    ORDER BY total DESC
""").pl()  # polars DataFrame 반환
```

### 파일 직접 쿼리

```python
# CSV 직접 쿼리 (메모리 효율적)
result = duckdb.query("""
    SELECT category, AVG(price) as avg_price
    FROM read_csv_auto('products.csv')
    WHERE price > 10
    GROUP BY category
""").df()  # pandas DataFrame 반환
```

### Parquet 최적화

```python
# Parquet 읽기/쓰기 (매우 빠름)
duckdb.query("""
    COPY (SELECT * FROM df WHERE sales > 100)
    TO 'output.parquet' (FORMAT PARQUET)
""")

# 읽기
result = duckdb.query("SELECT * FROM 'output.parquet'").pl()
```

---

## 3. 실전 예시: ETL 파이프라인

```python
import polars as pl
import duckdb

# 1. 추출 (Extract)
df_users = pl.read_csv("users.csv")
df_orders = pl.read_csv("orders.csv")

# 2. 변환 (Transform) - Polars
df_clean = (
    df_users
    .filter(pl.col("age") > 18)
    .with_columns([
        pl.col("email").str.to_lowercase().alias("email_clean")
    ])
)

# 3. 분석 (Analyze) - DuckDB
result = duckdb.query("""
    SELECT
        u.country,
        COUNT(DISTINCT o.order_id) as order_count,
        SUM(o.amount) as total_revenue
    FROM df_users u
    LEFT JOIN df_orders o ON u.user_id = o.user_id
    GROUP BY u.country
    ORDER BY total_revenue DESC
""").pl()

# 4. 적재 (Load)
result.write_parquet("report.parquet")
```

---

## 대체 도구들

### pandas ⚠️

- **이유**: polars가 10배 빠름
- **언제 OK**: 생태계 의존성 (sklearn, scipy 등), 작은 데이터

### spark ⚠️

- **이유**: 클러스터 불필요하면 polars+duckdb가 더 간단
- **언제 OK**: 진짜 빅데이터 (수십 TB+), 분산 처리 필요

---

**[← 관측성](observability.md)** | **[LLM 앱 →](llm-apps.md)**
