# Polars 완벽 가이드

> **차세대 데이터프레임 라이브러리**

⭐ **2026 추천** | 📊 데이터 처리 | ⚡ 10배 빠름 | 🦀 Rust 기반

---

## 개요

| 항목 | 내용 |
|------|------|
| **공식 사이트** | https://pola.rs |
| **GitHub** | https://github.com/pola-rs/polars |
| **PyPI** | https://pypi.org/project/polars/ |
| **첫 릴리즈** | 2020년 |
| **라이선스** | MIT |

### 한 줄 요약

**pandas보다 10배 빠른, 병렬 처리 기본 지원, 메모리 효율적인 데이터프레임 라이브러리**

---

## 왜 Polars인가

### 속도 비교

```python
# 1GB CSV 파일 읽기
pandas: 8.2초
polars: 0.6초  ← 13배 빠름!

# 그룹화 집계 (1억 행)
pandas: 45초
polars: 3초   ← 15배 빠름!
```

### 주요 특징

- ✅ **10배 빠른 속도** (Rust + SIMD + 병렬)
- ✅ **자동 멀티코어 활용**
- ✅ **Lazy evaluation** (메모리 효율)
- ✅ **Arrow 포맷** (제로카피 상호운용)
- ✅ **표현력 높은 API**

---

## 설치

```bash
$ uv add polars

# 모든 기능 포함
$ uv add 'polars[all]'
```

---

## 기본 사용법

### DataFrame 생성

```python
import polars as pl

# Dict에서
df = pl.DataFrame({
    "name": ["Alice", "Bob", "Charlie"],
    "age": [25, 30, 35],
    "city": ["NYC", "LA", "SF"]
})

# CSV에서
df = pl.read_csv("data.csv")

# Parquet에서 (가장 빠름)
df = pl.read_parquet("data.parquet")
```

### 기본 조작

```python
# 데이터 확인
df.head()
df.describe()
df.schema

# 필터링
df.filter(pl.col("age") > 25)

# 선택
df.select(["name", "age"])
df.select([pl.col("name"), pl.col("age") * 2])

# 정렬
df.sort("age", descending=True)
```

---

## 실전 패턴

### 패턴 1: 메서드 체이닝

```python
result = (
    df
    .filter(pl.col("age") > 25)
    .select(["name", "city"])
    .sort("name")
    .limit(10)
)
```

### 패턴 2: 표현식 (Expressions)

```python
df.select([
    pl.col("name").str.to_uppercase().alias("NAME"),
    (pl.col("age") * 12).alias("age_months"),
    pl.col("city").str.len().alias("city_len")
])
```

### 패턴 3: 그룹화 & 집계

```python
df.group_by("city").agg([
    pl.col("age").mean().alias("avg_age"),
    pl.col("age").max().alias("max_age"),
    pl.count().alias("count")
])
```

### 패턴 4: Lazy Evaluation

```python
# Lazy 모드 (메모리 효율적)
lazy_df = pl.scan_csv("huge_file.csv")  # 즉시 실행 안 됨

result = (
    lazy_df
    .filter(pl.col("value") > 1000)
    .group_by("category")
    .agg(pl.col("value").sum())
    .collect()  # 여기서 실제 실행 (최적화됨)
)
```

### 패턴 5: Window Functions

```python
df.select([
    pl.col("name"),
    pl.col("sales"),
    pl.col("sales").sum().over("department").alias("dept_total"),
    pl.col("sales").rank().over("department").alias("rank")
])
```

### 패턴 6: Join

```python
df1.join(
    df2,
    on="user_id",
    how="left"  # inner, left, outer, cross
)
```

### 패턴 7: 날짜/시간

```python
df.select([
    pl.col("date").dt.year().alias("year"),
    pl.col("date").dt.month().alias("month"),
    pl.col("date").dt.weekday().alias("weekday")
])
```

---

## pandas vs Polars

### 문법 비교

```python
# pandas
df[df['age'] > 25][['name', 'city']]

# polars (더 명시적)
df.filter(pl.col("age") > 25).select(["name", "city"])
```

### 마이그레이션

```python
# pandas DataFrame을 polars로
df_polars = pl.from_pandas(df_pandas)

# polars를 pandas로
df_pandas = df_polars.to_pandas()
```

### 언제 pandas, 언제 polars?

**Polars 선택:**
- ✅ 대용량 데이터 (100MB+)
- ✅ 성능이 중요
- ✅ 새 프로젝트
- ✅ 메모리 제약

**pandas 유지:**
- ✅ 기존 코드베이스
- ✅ sklearn, scipy 등과 통합
- ✅ 작은 데이터
- ✅ 복잡한 데이터 변환 (pandas가 더 유연)

---

## 성능 최적화 팁

### 1. Lazy Evaluation 사용

```python
# Eager (즉시 실행)
df = pl.read_csv("data.csv")
result = df.filter(...).group_by(...)

# Lazy (최적화된 실행)
result = (
    pl.scan_csv("data.csv")
    .filter(...)
    .group_by(...)
    .collect()  # 한 번에 최적화해서 실행
)
```

### 2. Parquet 사용

```python
# CSV (느림)
df = pl.read_csv("data.csv")  # 10초

# Parquet (빠름)
df = pl.read_parquet("data.parquet")  # 0.5초

# 변환
df.write_parquet("data.parquet")
```

### 3. 메모리 매핑

```python
df = pl.read_csv(
    "huge.csv",
    memory_map=True  # 디스크에서 직접 읽기
)
```

---

## 실전 예제

### ETL 파이프라인

```python
# Extract
df = pl.scan_csv("sales.csv")

# Transform
clean_df = (
    df
    .filter(pl.col("amount") > 0)
    .with_columns([
        pl.col("date").str.to_date().alias("date"),
        (pl.col("amount") * 1.1).alias("amount_with_tax")
    ])
    .group_by("product")
    .agg([
        pl.col("amount").sum().alias("total"),
        pl.col("amount").mean().alias("avg")
    ])
)

# Load
clean_df.collect().write_parquet("output.parquet")
```

---

**[← 데이터 처리](../../04-library-catalog/data-processing/README.md)** | **[다음: DuckDB →](duckdb.md)**
