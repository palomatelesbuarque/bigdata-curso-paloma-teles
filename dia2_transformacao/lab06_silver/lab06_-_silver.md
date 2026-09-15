# 🧪 Lab 6 — Silver: enriquecer com contexto

**Tempo:** 40 min · **Dia:** 2 · **Pré-requisito:** Lab 5 concluído

## 🎯 Objetivo

Construir a camada Silver: **juntar** transações com clientes (JOIN) e **derivar** colunas de análise (data, dia da semana, faixa de valor). É aqui que o dado ganha contexto de negócio — cada transação passa a saber de que segmento ela é.

O resultado, `silver_transactions`, é exatamente o que a Gold (Lab 7) vai agregar.

---

## 🖥️ ROTA A — Cluster real (Hive)

### Passo 1 — Juntar transações com clientes (a Silver)

> ⚠️ Se for reexecutar este passo (ex: depois de corrigir algo na Bronze), o `CREATE TABLE` falha com `Table already exists: default.silver_transactions`. Rode `DROP TABLE IF EXISTS silver_transactions;` antes.

O JOIN traz o `segment` e o `credit_score` do cliente para cada transação. Como `bronze_customers` é pequena, usamos a dica **MAPJOIN** (o broadcast do slide 15) para evitar shuffle.

```sql
CREATE TABLE silver_transactions
STORED AS PARQUET AS
SELECT /*+ MAPJOIN(c) */
  t.transaction_id,
  t.customer_id,
  t.amount,
  t.transaction_type,
  t.status,
  t.risk_score,
  t.is_fraud,
  t.ts,
  c.segment,
  c.credit_score,
  YEAR(t.ts)  AS year,
  MONTH(t.ts) AS month,
  DAY(t.ts)   AS day,
  DAYOFWEEK(t.ts) AS day_of_week,
  CASE
    WHEN t.amount < 100  THEN 'baixo'
    WHEN t.amount < 1000 THEN 'medio'
    ELSE 'alto'
  END AS amount_band
FROM bronze_transactions t
JOIN bronze_customers c ON t.customer_id = c.customer_id;
```

### Passo 2 — Conferir se o JOIN não perdeu nem duplicou linhas

```sql
SELECT COUNT(*) FROM silver_transactions;
SELECT COUNT(*) FROM bronze_transactions;
```
**Esperado:** os dois números são iguais **ou** a Silver tem um pouco menos que a Bronze — nesse caso, é esperado: alguns `customer_id` em `bronze_transactions` não têm registro correspondente em `bronze_customers` (clientes que nunca existiram na base, não é um filtro nosso). O `INNER JOIN` descarta essas transações "órfãs" corretamente. No dataset de referência, isso costuma ser ~46 linhas (4 clientes inexistentes). Se a diferença for muito maior que isso, investigue com o `LEFT JOIN` abaixo. Se a Silver ficou **maior** que a Bronze, há `customer_id` duplicado na dimensão de clientes — isso sim é bug.

Para identificar as transações órfãs:
```sql
SELECT DISTINCT t.customer_id
FROM bronze_transactions t
LEFT JOIN bronze_customers c ON t.customer_id = c.customer_id
WHERE c.customer_id IS NULL;
```
Confirme se esses `customer_id` existem na fonte crua (`raw_customers`) — se não existirem lá também, é dado realmente ausente, não um problema do filtro de `credit_score` do Lab 5:
```sql
SELECT customer_id FROM raw_customers WHERE customer_id IN (/* IDs encontrados acima */);
```

### Passo 3 — Garantir que todo mundo tem segmento

```sql
SELECT COUNT(*) FROM silver_transactions WHERE segment IS NULL;
```
**Esperado:** `0` — todas as transações foram enriquecidas.

### Passo 4 — Espiar as colunas derivadas

```sql
SELECT amount, amount_band, year, month, day, day_of_week
FROM silver_transactions
LIMIT 10;
```
**Esperado:** `amount_band` coerente com `amount`, e `year/month/day` batendo com o `ts`.

### Passo 5 — Um primeiro cruzamento (prévia da Gold)

```sql
SELECT segment,
       COUNT(*) AS transacoes,
       ROUND(100.0 * SUM(CASE WHEN is_fraud THEN 1 ELSE 0 END) / COUNT(*), 2) AS taxa_fraude_pct
FROM silver_transactions
GROUP BY segment
ORDER BY taxa_fraude_pct DESC;
```
**Esperado:** 3 segmentos, com **High-Risk** liderando a taxa de fraude — o insight que a Gold vai formalizar amanhã.

---

## 🔓 ROTA B — Sem admin (DuckDB)

Usa as tabelas Bronze salvas em Parquet no Lab 5.

### Passo 1 — Carregar a Bronze e montar a Silver

```python
import duckdb
con = duckdb.connect()

con.sql("CREATE TABLE bronze_customers    AS SELECT * FROM read_parquet('bigdata/bronze/customers.parquet')")
con.sql("CREATE TABLE bronze_transactions AS SELECT * FROM read_parquet('bigdata/bronze/transactions.parquet')")

con.sql("""
CREATE TABLE silver_transactions AS
SELECT
  t.transaction_id, t.customer_id, t.amount, t.transaction_type,
  t.status, t.risk_score, t.is_fraud, t.ts,
  c.segment, c.credit_score,
  year(t.ts)  AS year,
  month(t.ts) AS month,
  day(t.ts)   AS day,
  dayofweek(t.ts) AS day_of_week,
  CASE
    WHEN t.amount < 100  THEN 'baixo'
    WHEN t.amount < 1000 THEN 'medio'
    ELSE 'alto'
  END AS amount_band
FROM bronze_transactions t
JOIN bronze_customers c ON t.customer_id = c.customer_id
""")
con.sql("SELECT COUNT(*) FROM silver_transactions").show()
```

### Passo 2 — Validar o JOIN

```python
con.sql("SELECT COUNT(*) AS silver, (SELECT COUNT(*) FROM bronze_transactions) AS bronze FROM silver_transactions").show()
con.sql("SELECT COUNT(*) AS sem_segmento FROM silver_transactions WHERE segment IS NULL").show()
```
**Esperado:** `silver` = `bronze` e `sem_segmento` = 0.

### Passo 3 — Prévia do cruzamento por segmento

```python
con.sql("""
SELECT segment,
       COUNT(*) AS transacoes,
       ROUND(100.0 * SUM(CASE WHEN is_fraud THEN 1 ELSE 0 END) / COUNT(*), 2) AS taxa_fraude_pct
FROM silver_transactions
GROUP BY segment
ORDER BY taxa_fraude_pct DESC
""").show()
```

### Passo 4 — Persistir a Silver (o Lab 7 e o Lab 10 leem daqui)

```python
con.sql("COPY silver_transactions TO 'bigdata/silver/transactions_enriched.parquet' (FORMAT PARQUET)")
print("Silver layer salva em bigdata/silver/transactions_enriched.parquet")
```

---

## ✅ Checkpoint

- [ ] `silver_transactions` criada a partir do JOIN transações × clientes
- [ ] Contagem da Silver bate com a `bronze_transactions` — ou a diferença é pequena e explicada por clientes ausentes na origem (não um bug)
- [ ] Nenhuma linha com `segment IS NULL`
- [ ] Colunas derivadas `year/month/day`, `day_of_week` e `amount_band` presentes
- [ ] (Rota B) `bigdata/silver/transactions_enriched.parquet` salvo

---

## 🔧 Troubleshooting

| Problema | Rota | Solução |
|----------|------|---------|
| Silver com **mais** linhas que a Bronze | A/B | Há `customer_id` duplicado em `bronze_customers` — volte ao Lab 5 e confira o `SELECT DISTINCT` da dimensão |
| Silver com **menos** linhas (diferença pequena, ex: ~46) | A/B | Esperado — alguns `customer_id` em transações não existem na base de clientes (dado ausente na origem, não um bug). Confirme com `LEFT JOIN` (Passo 2) que os IDs órfãos também não aparecem em `raw_customers` |
| Silver com **muito menos** linhas (queda grande, não só alguns órfãos) | A/B | Aí sim investigue: tipos de `customer_id` diferentes entre as tabelas, ou filtro do Lab 5 descartando clientes em excesso |
| `DAYOFWEEK`/`YEAR` não existe | A | Em Hive antigo use `from_unixtime(unix_timestamp(ts), 'yyyy')` etc., ou `date_format(ts,'u')` para dia da semana |
| `segment` vem tudo nulo | A/B | A chave do JOIN pode estar com tipos diferentes — garanta `customer_id` como `INT` nas duas Bronze |
| MAPJOIN não ajuda / estoura memória | A | Remova a dica `/*+ MAPJOIN(c) */` — o Hive faz JOIN normal (só mais lento) |

**Próximo lab:** `DIA2_LAB07_GOLD.md` — agregações e KPIs por segmento.
