# 데이터 파이프라인 구축

## 목표

ETL (Extract, Transform, Load) 파이프라인을 구축하여 대용량 데이터를 효율적으로 처리하기

---

## 기술 스택

| 분야 | 도구 | 이유 |
|------|------|------|
| 데이터프레임 | Polars | pandas보다 10배 빠름 |
| SQL 분석 | DuckDB | 컬럼형 스토리지, 고성능 |
| 워크플로우 | Dagster | 현대적 오케스트레이션 |
| 파일 포맷 | Parquet | 압축률, 속도 |

---

## 간단한 ETL 예시

```python
import polars as pl
import duckdb

# Extract: 여러 소스에서 데이터 로드
df_users = pl.read_csv("users.csv")
df_orders = pl.read_parquet("orders.parquet")
df_products = pl.scan_csv("products.csv")  # Lazy

# Transform: 데이터 정제 및 변환
df_clean = (
    df_users
    .filter(pl.col("age") >= 18)
    .with_columns([
        pl.col("email").str.to_lowercase().alias("email_normalized")
    ])
)

# Join: DuckDB로 복잡한 쿼리
result = duckdb.query("""
    SELECT
        u.user_id,
        u.name,
        COUNT(o.order_id) as order_count,
        SUM(o.total) as revenue,
        AVG(o.total) as avg_order_value
    FROM df_users u
    LEFT JOIN df_orders o ON u.user_id = o.user_id
    GROUP BY u.user_id, u.name
    HAVING order_count > 0
    ORDER BY revenue DESC
""").pl()

# Load: 결과 저장
result.write_parquet("output/user_stats.parquet")
```

---

## Dagster를 활용한 파이프라인

```python
from dagster import asset, AssetExecutionContext
import polars as pl

@asset
def raw_users(context: AssetExecutionContext):
    """원시 사용자 데이터 추출"""
    context.log.info("Loading users...")
    return pl.read_csv("data/users.csv")

@asset
def clean_users(raw_users: pl.DataFrame):
    """사용자 데이터 정제"""
    return (
        raw_users
        .filter(pl.col("age") >= 18)
        .drop_nulls()
    )

@asset
def user_stats(clean_users: pl.DataFrame):
    """사용자 통계 생성"""
    return (
        clean_users
        .group_by("country")
        .agg([
            pl.count().alias("user_count"),
            pl.col("age").mean().alias("avg_age")
        ])
    )
```

---

**[← LLM 서비스](llm-service.md)** | **[자동화 →](automation.md)**
