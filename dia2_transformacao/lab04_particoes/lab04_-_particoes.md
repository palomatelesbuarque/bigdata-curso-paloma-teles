# 🧪 Lab 4 — Partições: organizar por ano/mês

**Tempo:** 35 min · **Dia:** 2 · **Pré-requisito:** Lab 3 concluído

## 🎯 Objetivo

Criar uma tabela particionada por `year`/`month` e medir a diferença de tempo entre uma busca com e sem partição.

---

## 🖥️ ROTA A — Cluster real (Hive)

### Passo 1 — Criar tabela raw de transactions (se ainda não existe)

```sql
CREATE EXTERNAL TABLE raw_transactions (
  transaction_id INT, customer_id INT, amount FLOAT,
  transaction_type STRING, ts STRING, status STRING,
  risk_score FLOAT, is_fraud STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/bigdata/raw/transactions'
TBLPROPERTIES ('skip.header.line.count'='1');
```
> ⚠️ **`is_fraud` como STRING, não BOOLEAN:** declarar a coluna direto como `BOOLEAN` faz o parser do Hive (`LazySimpleSerDe`) converter qualquer valor que não reconheça exatamente para `false`, silenciosamente. Mantenha como `STRING` aqui e converta explicitamente na Bronze (Lab 5) com `CASE WHEN`, onde há controle sobre o mapeamento.

### Passo 2 — Criar a tabela particionada

> ⚠️ Mesma regra do Passo 1: `is_fraud` como `STRING`, não `BOOLEAN` — o `INSERT` do Passo 3 lê direto de `raw_transactions` (já `STRING`), e uma coluna de destino `BOOLEAN` reintroduziria a conversão silenciosa para `false`.

```sql
CREATE TABLE transactions_particionada (
  transaction_id INT, customer_id INT, amount FLOAT,
  transaction_type STRING, status STRING, risk_score FLOAT, is_fraud STRING
)
PARTITIONED BY (year INT, month INT)
STORED AS TEXTFILE;
```

### Passo 3 — Popular as partições a partir da raw

```sql
SET hive.exec.dynamic.partition.mode=nonstrict;

INSERT INTO TABLE transactions_particionada PARTITION(year, month)
SELECT transaction_id, customer_id, amount, transaction_type, status,
       risk_score, is_fraud,
       YEAR(ts) AS year, MONTH(ts) AS month
FROM raw_transactions;
```
**Esperado:** job MapReduce roda e cria várias partições.

### Passo 4 — Confirmar as partições

```sql
SHOW PARTITIONS transactions_particionada;
```
**Esperado:** ~24 linhas (2 anos × 12 meses).

### Passo 5 — Medir a diferença de tempo

```sql
-- SEM aproveitar partição (varre tudo)
SELECT COUNT(*) FROM raw_transactions WHERE MONTH(ts) = 1;

-- COM partição (Hive só lê a pasta de janeiro)
SELECT COUNT(*) FROM transactions_particionada WHERE month = 1;
```
**Esperado:** os dois retornam o mesmo número, mas o segundo aparece mais rápido no "Time taken" — compare os dois.

### Passo 6 — MSCK REPAIR (quando as partições já existem como pasta, mas não estão registradas)

```sql
MSCK REPAIR TABLE transactions_particionada;
```
**Esperado (se já populado):** `no partitions added` — usado quando alguém sobe arquivos direto no HDFS sem passar pelo INSERT.

---

## 🔓 ROTA B — Sem admin (DuckDB + Parquet particionado)

### Passo 1 — Ler o CSV e criar colunas year/month

```python
import duckdb
con = duckdb.connect()

con.sql("""
CREATE VIEW raw_transactions AS
SELECT *, strftime(timestamp::TIMESTAMP, '%Y')::INT AS year,
          strftime(timestamp::TIMESTAMP, '%m')::INT AS month
FROM read_csv_auto('bigdata/raw/transactions/transactions_synthetic.csv')
""")
con.sql("SELECT DISTINCT year, month FROM raw_transactions ORDER BY 1,2").show()
```
**Esperado:** ~24 combinações ano/mês listadas.

### Passo 2 — Gravar particionado de verdade em Parquet (DuckDB suporta isso nativamente)

```python
con.sql("""
COPY raw_transactions TO 'bigdata/silver/transactions_particionado'
(FORMAT PARQUET, PARTITION_BY (year, month))
""")
```

### Passo 3 — Conferir a estrutura de pastas gerada

```bash
find bigdata/silver/transactions_particionado -maxdepth 2 | head -10
```
**Esperado:** pastas `year=2023/month=1/`, `year=2023/month=2/`, etc — **isso é particionamento real**, o mesmo conceito do Hive.

### Passo 4 — Medir a diferença de tempo

```python
import time

t0 = time.time()
con.sql("SELECT COUNT(*) FROM raw_transactions WHERE month = 1").show()
print("sem partição:", time.time() - t0, "s")

t0 = time.time()
con.sql("""
SELECT COUNT(*) FROM read_parquet(
  'bigdata/silver/transactions_particionado/year=*/month=1/*.parquet'
)""").show()
print("com partição:", time.time() - t0, "s")
```
**Esperado:** a leitura particionada tende a ser mais rápida — mesmo num dataset pequeno, o padrão já aparece.

---

## ✅ Checkpoint

- [ ] Tabela/pasta particionada por year/month criada
- [ ] `SHOW PARTITIONS` (ou `find`) mostra ~24 partições
- [ ] Você comparou o tempo com e sem partição
- [ ] Entendeu quando usar `MSCK REPAIR TABLE`

---

## 🔧 Troubleshooting

| Problema | Rota | Solução |
|----------|------|---------|
| `dynamic partition mode strict` erro | A | Rode `SET hive.exec.dynamic.partition.mode=nonstrict;` antes do INSERT |
| `is_fraud` só mostra `false` | A | A coluna foi declarada `BOOLEAN` em vez de `STRING` no `CREATE EXTERNAL TABLE` (Passo 1) — o parser do Hive zera silenciosamente valores não reconhecidos. Recrie com `is_fraud STRING` |
| Poucas partições geradas | A | Confira se a coluna `ts`/`timestamp` está no formato esperado pelo `YEAR()`/`MONTH()` |
| `strftime` erro de tipo | B | Confirme que a coluna timestamp está sendo lida como string — use `::TIMESTAMP` no cast |
| Pasta particionada vazia | B | Confira se `bigdata/silver/` existe (`mkdir -p bigdata/silver`) |

**Próximo lab:** `DIA2_LAB05_BRONZE.md` — limpar os dados de verdade.