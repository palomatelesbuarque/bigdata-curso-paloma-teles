# Relatório do Pipeline — Etapa 2 (Big Data)

## 1. Ingestão

O arquivo `avaliacao_transactions.csv`, com 30.000 transações, foi carregado diretamente no DuckDB a partir do CSV fornecido pelo professor. Essa etapa apenas trouxe os dados brutos para dentro do ambiente de processamento, sem nenhuma transformação — o objetivo aqui foi só confirmar que a origem estava acessível e legível corretamente. Nenhum problema apareceu: as 30.000 linhas foram lidas sem erro.

## 2. Criação da tabela raw

A leitura automática do DuckDB (`read_csv_auto`) já inferiu os tipos corretos de cada coluna sem necessidade de conversão manual: `transaction_id` e `customer_id` como inteiros, `amount` e `risk_score` como ponto flutuante, `timestamp` como data/hora, e `is_fraud` como booleano. A única decisão tomada aqui foi validar esse schema antes de seguir adiante (via `DESCRIBE`), em vez de assumir que os tipos estavam certos — o que evitou arrastar um problema de tipagem para as etapas seguintes.

## 3. Particionamento

Os dados foram reorganizados em arquivos Parquet particionados pela coluna `ano_mes`, derivada do campo `timestamp` de cada transação. Essa decisão segue diretamente a estratégia definida na Etapa 1 (arquitetura): como o padrão de consulta mais provável da área de risco é recortado por período (relatórios mensais, comparação entre meses), particionar por `ano_mes` permite que consultas futuras leiam só os arquivos do mês relevante, em vez de varrer a base inteira. Nenhum problema técnico apareceu nesta etapa além de garantir que a pasta de destino existisse antes da escrita.

## 4. Bronze — limpeza

Na camada Bronze, foram aplicados dois filtros de limpeza: remoção de duplicatas por `transaction_id` e descarte de valores impossíveis (`amount` menor ou igual a zero, `risk_score` fora do intervalo 0-100, `credit_score` fora do intervalo 300-900). O resultado foi que as 30.000 linhas permaneceram intactas — nenhuma foi removida. Isso não indica falha na limpeza, e sim que a base fornecida pelo professor já foi gerada sinteticamente dentro desses limites (não havia sujeira real para remover nesses critérios específicos). A limpeza foi aplicada e verificada de qualquer forma, o que confirma a qualidade da base de origem em vez de simplesmente presumi-la.

## 5. Silver — enriquecimento

A camada Silver derivou três colunas novas a partir dos dados já limpos: `faixa_valor` (baixo/médio/alto, com base no valor da transação), `periodo_dia` (madrugada/manhã/tarde/noite, com base na hora do timestamp) e `dia_semana`. As colunas `channel` e `merchant_category`, que já existiam na base original, foram mantidas propositalmente na Silver — essa foi uma decisão direta, já que a Etapa 3 desta avaliação pede explicitamente análise de risco por canal e por categoria de estabelecimento, e essas duas colunas acabaram se mostrando as mais relevantes de toda a análise (a combinação `app` + `viagem` concentrou a maior taxa de fraude de toda a base).

## 6. Gold — agregações

Foram criadas duas tabelas Gold, cada uma pensada para alimentar diretamente as análises da Etapa 3:

- **`gold_risco_canal_categoria`**: agrega total de transações, risco médio, taxa de fraude e valor total por combinação de `channel` e `merchant_category`. Essa tabela responde diretamente à pergunta de risco por canal e por categoria pedida na Etapa 3, e foi ela que revelou o achado mais forte do projeto — a combinação `app` + `viagem` com taxa de fraude muito acima de qualquer outra.
- **`gold_segmento_tempo`**: agrega volume e valor transacionado por `segment`, faixa de `credit_score` e dia. Essa tabela dá suporte a uma visão de volume/risco ao longo do tempo, cruzando o segmento do cliente com sua faixa de score.

---

## Respostas às perguntas do enunciado

**1. Quantas linhas sobreviveram da Bronze em diante? Alguma foi descartada, por quê?**
Todas as 30.000 linhas sobreviveram à limpeza da Bronze — nenhuma foi descartada. Os filtros de deduplicação e de valores impossíveis foram aplicados e verificados, mas a base já vinha limpa nesses critérios, por ser gerada sinteticamente pelo script do professor.

**2. Sua Silver layer usa as colunas `channel` e `merchant_category`? Justifique.**
Sim, ambas foram mantidas. A justificativa é que a Etapa 3 exige análise de risco por canal e por categoria de estabelecimento — sem manter essas colunas na Silver, seria impossível responder a essas perguntas na etapa seguinte. Na prática, essas duas colunas se mostraram as mais informativas de todo o pipeline.

**3. Que agregações você colocou na Gold, e por quê — o que elas respondem que a Etapa 3 vai usar?**
A `gold_risco_canal_categoria` responde diretamente às análises de risco por canal e por categoria da Etapa 3, além de ter sido a base para identificar o achado central do projeto (risco concentrado em app + viagem). A `gold_segmento_tempo` dá suporte a uma visão complementar, cruzando segmento do cliente e faixa de score ao longo do tempo — uma dimensão de negócio que não estava coberta pela primeira tabela.
