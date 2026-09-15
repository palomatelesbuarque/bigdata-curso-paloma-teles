# Justificativa da Arquitetura — TechPay

## 1. Por que a rota sem admin (DuckDB + PySpark local + Plotly)?

Optei pela rota sem administrador — DuckDB para ingestão e processamento (Bronze, Silver, Gold), PySpark local para exploração e Plotly para o dashboard — em vez do cluster real com HDFS, Hive e Spark distribuído.

A alternativa avaliada foi montar o cluster completo (HDFS + Hive + Spark). A razão de não seguir por esse caminho foi o risco de infraestrutura: configurar um cluster do zero envolve dependências de rede, memória e serviços que frequentemente falham de forma independente da qualidade do pipeline em si — e, para uma base de 30 mil transações, o ganho de usar um cluster distribuído de verdade é irrelevante, já que o volume cabe confortavelmente em memória local. A escolha prioriza garantir a execução completa e correta do pipeline (ingestão → raw → particionamento → Bronze → Silver → Gold → BI) em vez de arriscar o entendimento do fluxo por um problema de configuração de cluster que não tem relação com a lógica de dados em si. O DuckDB, em particular, executa SQL padrão sobre arquivos Parquet locais com desempenho colunar comparável a um motor distribuído em pequena escala, o que preserva o mesmo raciocínio de camadas (Bronze/Silver/Gold) que seria usado no Hive.

## 2. Estratégia de particionamento

Os dados foram particionados por `ano_mes`, uma coluna derivada do campo `timestamp` de cada transação (formato `YYYY-MM`).

A justificativa é o padrão de consulta mais provável da área de risco da TechPay: análises de fraude e de volume transacional tendem a ser recortadas por período — "quantas transações fraudulentas tivemos em agosto", "como o risco por canal evoluiu no último trimestre". Com o particionamento por mês, uma consulta desse tipo lê apenas os arquivos Parquet da(s) partição(ões) relevante(s), em vez de varrer a base inteira. Num cenário concreto: se a diretoria pede o relatório mensal de risco, a query filtra `ano_mes = '2025-08'` e o motor acessa diretamente a pasta `ano_mes=2025-08/`, ignorando os outros 11 meses — o ganho de leitura é proporcional ao número de partições descartadas.

Uma alternativa considerada foi particionar por `channel`, já que a análise de risco por canal também é central neste projeto. Optei por `ano_mes` porque consultas temporais tendem a ser mais frequentes no dia a dia operacional (relatórios recorrentes) do que consultas fixas por canal, e porque o número de canais (4) geraria partições muito desbalanceadas em volume, enquanto os meses distribuem os dados de forma mais uniforme.

## 3. Pontos de falha se o volume multiplicar por 100

Com a base crescendo de 30 mil para 3 milhões de transações, o primeiro ponto a quebrar seria a **camada de ingestão single-node**: hoje a leitura do CSV e a escrita em Parquet acontecem num único processo, em memória local — em um volume 100x maior, a leitura inteira do arquivo de origem antes de processar deixaria de ser viável, tanto pelo tempo quanto pelo uso de memória.

O segundo ponto de risco é a **granularidade do particionamento**: com `ano_mes`, cada partição ficaria 100x maior, e uma única partição poderia voltar a ser lenta de consultar, reproduzindo o mesmo problema que o particionamento resolveu na escala atual — nesse caso, seria necessário adicionar um segundo nível de particionamento (por exemplo, `ano_mes` + `channel`, ou `ano_mes` + dia).

**Mitigação:** migrar a ingestão para um formato de leitura incremental/streaming (não reler o arquivo inteiro a cada execução) e trocar o processamento single-node por um motor distribuído de verdade (Spark em cluster), mantendo a mesma lógica lógica de camadas Bronze/Silver/Gold — a arquitetura conceitual não muda, apenas a capacidade de execução por trás dela.

## 4. O que mudaria para detecção de fraude em tempo real

O desenho atual é inteiramente batch: os dados são processados em lotes (a cada execução do pipeline), o que introduz um atraso natural entre a transação acontecer e ela aparecer nas camadas Silver/Gold e no dashboard.

Para detecção em tempo real, a mudança central seria na camada de ingestão: em vez de ler um arquivo CSV estático, a origem passaria a publicar cada transação em um sistema de streaming (por exemplo, Kafka), e o processamento Bronze/Silver deixaria de rodar em lotes agendados para rodar continuamente sobre esse fluxo (Spark Structured Streaming, ou equivalente). A camada Gold também mudaria de natureza: em vez de tabelas agregadas recalculadas periodicamente, passaria a manter métricas atualizadas em uma janela deslizante (ex.: taxa de fraude nos últimos 5 minutos por canal). O dashboard, por sua vez, deixaria de ser um HTML estático gerado sob demanda e passaria a se conectar a essa camada de métricas em tempo real, atualizando automaticamente.
