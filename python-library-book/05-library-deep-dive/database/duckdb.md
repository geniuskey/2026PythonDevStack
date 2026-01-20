# DuckDB 완벽 가이드

> **초고속 인메모리 분석 데이터베이스**

⭐ **2026 추천** | 🦆 SQL | ⚡ 빠름 | 📊 분석

---

## 개요

| 항목 | 내용 |
|------|------|
| **공식 사이트** | https://duckdb.org |
| **GitHub** | https://github.com/duckdb/duckdb |

### 한 줄 요약

**DataFrame에 SQL을 직접 실행할 수 있는 초고속 분석 데이터베이스**

---

## 설치

```bash
$ uv add duckdb
```

---

## 기본 사용법

### DataFrame 직접 쿼리

```python
import duckdb
import pandas as pd

df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie'],
    'age': [25, 30, 35],
    'city': ['NYC', 'LA', 'SF']
})

# DataFrame에 직접 SQL!
result = duckdb.query("""
    SELECT name, age
    FROM df
    WHERE age > 25
    ORDER BY age DESC
""").df()  # pandas DataFrame 반환

print(result)
```

### CSV 직접 쿼리

```python
# CSV 파일을 메모리에 로드하지 않고 쿼리
result = duckdb.query("""
    SELECT category, AVG(price) as avg_price
    FROM read_csv_auto('products.csv')
    WHERE price > 10
    GROUP BY category
""").df()
```

---

## 실전 패턴

### 패턴 1: Polars 통합

```python
import polars as pl
import duckdb

df = pl.DataFrame({
    'product': ['A', 'B', 'A', 'C'],
    'sales': [100, 200, 150, 300]
})

# Polars DataFrame에 SQL
result = duckdb.query("""
    SELECT product, SUM(sales) as total
    FROM df
    GROUP BY product
""").pl()  # Polars DataFrame 반환
```

### 패턴 2: Parquet 최적화

```python
# Parquet 읽기/쓰기 (매우 빠름)
duckdb.query("""
    COPY (SELECT * FROM df WHERE sales > 100)
    TO 'output.parquet' (FORMAT PARQUET)
""")

# 읽기
result = duckdb.query("SELECT * FROM 'output.parquet'").df()
```

### 패턴 3: 영구 데이터베이스

```python
# 파일 기반 DB
con = duckdb.connect('my_database.db')

con.execute("CREATE TABLE users (id INTEGER, name VARCHAR)")
con.execute("INSERT INTO users VALUES (1, 'Alice'), (2, 'Bob')")

result = con.execute("SELECT * FROM users").df()
con.close()
```

---

## pandas보다 빠른 이유

```python
# pandas - 그룹화 집계
df.groupby('category')['sales'].sum()  # 느림

# DuckDB - 동일 작업
duckdb.query("""
    SELECT category, SUM(sales)
    FROM df
    GROUP BY category
""").df()  # 10배+ 빠름!
```

---

**[← 데이터베이스](../../04-library-catalog/database/README.md)**
