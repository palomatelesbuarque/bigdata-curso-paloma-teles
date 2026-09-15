# Relatório de Achados e Ideia do BI — Etapa 3

## Análises realizadas

### 1. Risco por canal

O canal `app` concentra a maior taxa de fraude (3,79%), quase 2,7 vezes maior que o canal mais seguro, `pos` (1,42%). Um ponto que chama atenção: o `risk_score` médio é praticamente igual entre os quatro canais (em torno de 49-50), ou seja, o score de risco isolado não capturaria essa diferença — é preciso olhar a taxa de fraude real, não só o score, para enxergar o padrão.

- **Finding:** `app` tem taxa de fraude de 3,79%, contra 1,42% do `pos`.
- **Insight:** o canal `app` provavelmente tem menos camadas de autenticação/verificação que canais presenciais como o `pos`, o que facilita fraude.
- **Ação:** reforçar autenticação (biometria, segunda confirmação) especificamente para transações via app, em vez de aplicar a mesma régua de verificação a todos os canais.

### 2. Risco por categoria de estabelecimento

A categoria `viagem` lidera isoladamente a taxa de fraude (5,32%), quase o dobro da segunda colocada (`saude`, 2,47%).

- **Finding:** `viagem` tem taxa de fraude de 5,32%, mais que o dobro de qualquer outra categoria.
- **Insight:** compras de viagem costumam ter tickets mais altos e ocorrer de forma menos previsível (parcelamento, moeda estrangeira, datas específicas), características que tornam mais fácil mascarar uma transação fraudulenta como legítima.
- **Ação:** aplicar verificação adicional para compras de viagem acima de um valor de corte, antes da aprovação automática.

### 3. Pergunta de negócio própria: risco na combinação app + viagem, por período do dia

Cruzando os dois achados anteriores, testei se a combinação `app` + `viagem` — já a mais arriscada isoladamente em cada dimensão — piora ainda mais dependendo do horário. O resultado confirma que sim: dentro dessa combinação, a madrugada concentra 9,24% de fraude, contra 5,71% à tarde.

- **Finding:** transações via app, em compras de viagem, feitas de madrugada, têm taxa de fraude de 9,24% — mais de 3,7 vezes a média geral da base (2,50%).
- **Insight:** a combinação de menor vigilância humana (madrugada), canal com menos camadas de autenticação (app) e categoria de ticket alto e baixa previsibilidade (viagem) concentra o risco de forma multiplicativa, não apenas aditiva.
- **Ação:** criar uma regra específica de verificação reforçada (segunda autenticação obrigatória) para compras de viagem via app entre 0h e 5h — uma regra estreita, que não penaliza a experiência da maioria dos usuários, mas cobre o segmento de maior risco identificado.

## Descrição da ideia do BI

**Para quem é este dashboard:** o time de análise de risco/fraude da TechPay — analistas que monitoram taxas de fraude diariamente e decidem quando escalar uma regra de bloqueio ou verificação adicional.

**Que decisão ele ajuda a tomar:** se e onde aplicar camadas extras de verificação (autenticação adicional, bloqueio automático, revisão manual), com base em qual canal, categoria ou combinação de fatores está concentrando risco no momento. O dashboard não decide sozinho, mas dá ao analista os elementos para priorizar onde investigar primeiro.

**Por que esses KPIs e não outros:** os 4 blocos escolhidos (taxa de fraude geral, tendência por dia da semana, composição de transações por canal, detalhe por categoria) cobrem as três perguntas centrais de um analista de risco: "como estamos no geral" (KPI), "isso está mudando com o tempo" (tendência) e "onde exatamente está concentrado o problema" (composição + detalhe). Foi descartado um KPI de volume financeiro total (soma de `amount`), porque a área de risco se importa mais com proporção de fraude do que com volume bruto transacionado — volume financeiro é uma métrica mais relevante para o time comercial do que para o time de risco.

**Quem seria o dono do painel:** o analista ou coordenador de risco/antifraude da TechPay seria quem olharia esse painel semanalmente (ou diariamente, em cenário de produção) e agiria sobre ele — por exemplo, acionando o time de produto para criar uma nova regra de verificação quando um padrão como o de `app` + `viagem` + madrugada aparecer de forma consistente ao longo do tempo, não só como um pico isolado.
