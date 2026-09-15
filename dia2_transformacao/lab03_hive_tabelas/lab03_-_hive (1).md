# 🧪 Lab 3 — Hive: tabelas Managed vs External

**Tempo:** 40 min · **Dia:** 2 · **Pré-requisito:** Lab 1 e 2 concluídos

## 🎯 Objetivo

Criar tabelas Hive sobre os dados do HDFS e comprovar na prática a diferença entre Managed e External — inclusive o que acontece com `DROP TABLE`.

---

## 🖥️ ROTA A — Cluster real (Hive)

### Passo 1 — Abrir o Hive

```bash
hive
```

### Passo 2 — Criar tabela EXTERNAL sobre o raw

```sql
CREATE EXTERNAL TABLE raw_customers (
  customer_id INT, name STRING, cpf STRING, email STRING,
  segment STRING, credit_score INT, created_at STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/bigdata/raw/customers'
TBLPROPERTIES ('skip.header.line.count'='1');
```

### Passo 3 — Conferir os dados

```sql
SELECT COUNT(*) FROM raw_customers;
SELECT * FROM raw_customers LIMIT 5;
```
**Esperado:** 9993 e uma amostra com nomes/CPFs/segmento.

### Passo 3.5 (opcional) — Corrigir HADOOP_CLASSPATH antes do INSERT

> Pule se seus `INSERT` no Hive já funcionam normalmente. O `INSERT INTO ... VALUES` do próximo passo dispara um job MapReduce de verdade — diferente do `CREATE TABLE` e do `SELECT`, que não precisam disso. Se aparecer:
> ```
> FAILED: Execution Error, return code -101 from org.apache.hadoop.hive.ql.exec.mr.MapRedTask. HADOOP_CLASSPATH
> ```
> é porque a variável `HADOOP_CLASSPATH` não está exportada para o processo do Hive.

Saia do Hive (`quit;`) e, no bash, teste e exporte:

```bash
hadoop classpath
export HADOOP_CLASSPATH=$(hadoop classpath)
```

Para não precisar repetir a cada sessão nova, torne permanente:

```bash
echo 'export HADOOP_CLASSPATH=$(hadoop classpath)' >> ~/.bashrc
source ~/.bashrc
```

Reabra o Hive (`hive`) e siga para o Passo 4.

### Passo 4 — Criar uma tabela MANAGED de teste

```sql
CREATE TABLE teste_managed (id INT, valor STRING);
INSERT INTO teste_managed VALUES (1, 'linha de teste');
SELECT * FROM teste_managed;
```
**Esperado:** o `INSERT` completa o job MapReduce (mostra `Launching Job 1 out of 3` e termina sem erro) e o `SELECT` retorna a linha `1  linha de teste`. Se o `INSERT` falhar com `return code -101 ... HADOOP_CLASSPATH`, veja o Passo 3.5.

### Passo 5 — O experimento do DROP (o coração do lab)

```sql
-- confira que o arquivo existe no HDFS ANTES de dropar:
!hadoop fs -ls /user/hive/warehouse/teste_managed;

DROP TABLE teste_managed;

-- confira DE NOVO — o que aconteceu?
!hadoop fs -ls /user/hive/warehouse/;
```
**Esperado:** a pasta `teste_managed` **sumiu** do HDFS. Managed table = Hive é dono, `DROP` apaga tudo.

### Passo 6 — Repita o experimento com EXTERNAL

```sql
DROP TABLE raw_customers;
!hadoop fs -ls /user/bigdata/raw/customers/;
```
**Esperado:** o arquivo `customers_synthetic.csv` **continua lá** no HDFS. Só o metadado do Hive sumiu.

### Passo 7 — Recriar a tabela raw_customers (você vai precisar dela)

Rode novamente o comando do Passo 2.

---

## 🔓 ROTA B — Sem admin (DuckDB)

> DuckDB não tem o conceito de "tabela dona do arquivo" — ele sempre lê o arquivo diretamente, então o comportamento é sempre como uma External table. O experimento do DROP vira uma demonstração *conceitual* em Python.

### Passo 1 — Abrir o Python com DuckDB

```bash
python3
```
```python
import duckdb
con = duckdb.connect()
```

### Passo 2 — "Criar tabela" apontando para o CSV (equivalente a External)

```python
con.sql("""
CREATE VIEW raw_customers AS
SELECT * FROM read_csv_auto('bigdata/raw/customers/customers_synthetic.csv')
""")
con.sql("SELECT COUNT(*) FROM raw_customers").show()
```
**Esperado:** 9993.

### Passo 3 — Demonstrar Managed vs External com um teste

```python
import os

# "Managed": copiamos o dado para dentro do "warehouse" do DuckDB
con.sql("CREATE TABLE teste_managed AS SELECT * FROM raw_customers LIMIT 5")
con.sql("DROP TABLE teste_managed")
# o arquivo original nunca foi tocado:
print("Arquivo original ainda existe?", os.path.exists('bigdata/raw/customers/customers_synthetic.csv'))
```
**Esperado:** `True` — mesmo dropando a tabela, o CSV original nunca foi apagado.

### Passo 4 — O que isso ensina

```python
print("""
No DuckDB/pandas, você SEMPRE trabalha como se fosse External:
o arquivo fonte nunca é apagado por um DROP.
No Hive de verdade, Managed table apaga o arquivo — cuidado redobrado lá.
""")
```

---

## ✅ Checkpoint

- [ ] Tabela sobre `customers` criada e com 9.993 linhas
- [ ] (Rota A) Você viu o HDFS perder o arquivo com Managed + DROP, e manter com External + DROP
- [ ] (Rota B) Você entende por que essa distinção quase não existe fora do Hive/Spark real
- [ ] `raw_customers` está recriada e disponível para o próximo lab

---

## 🔧 Troubleshooting

| Problema | Rota | Solução |
|----------|------|---------|
| `FAILED: SemanticException` ao criar tabela | A | Confira vírgulas e tipos no `CREATE TABLE` |
| `FAILED: Execution Error, return code -101 ... HADOOP_CLASSPATH` no `INSERT` | A | `HADOOP_CLASSPATH` não exportado para o Hive. Rode `export HADOOP_CLASSPATH=$(hadoop classpath)` no bash antes de abrir o `hive` (veja Passo 3.5) |
| Comando `!hadoop fs` não funciona no Hive | A | O `!` no início roda shell dentro do Hive CLI — confirme que está no prompt `hive>` |
| `read_csv_auto` não reconhece cabeçalho | B | Use `read_csv_auto('arquivo.csv', header=true)` explicitamente |
| `duckdb` não instalado | B | `pip install duckdb --user` |

**Próximo lab:** `DIA2_LAB04_PARTICOES.md` — particionar por ano/mês.