# 🏎️ F1 Lakehouse — Pipeline de Dados da Fórmula 1

Pipeline de dados completo construído no **Databricks** com dados reais da Fórmula 1, cobrindo todo o ciclo: ingestão de arquivos brutos, transformação, modelagem dimensional e um dashboard de classificação de pilotos.

O projeto segue a **arquitetura Medallion** (Bronze → Silver → Gold), organizada em Unity Catalog (`formula1.bronze`, `formula1.silver`, `formula1.gold`).

---

## 🏗️ Arquitetura

```
                ┌──────────┐     ┌──────────┐     ┌──────────┐
   Arquivos  →  │  BRONZE  │  →  │  SILVER  │  →  │   GOLD   │  →  Dashboard
   (CSV)        │  raw     │     │  limpo   │     │  modelado│
                └──────────┘     └──────────┘     └──────────┘
```

- **Bronze:** ingestão dos arquivos brutos (`circuits`, `races`, `constructors`, `drivers`, `results`, `sprints`), com metadados de ingestão (`ingestion_timestamp`, `source_file`) adicionados automaticamente.
- **Silver:** limpeza, padronização de nomes de colunas, remoção de duplicados e normalização de texto (`initcap`, tipos corretos).
- **Gold:** modelagem dimensional — dimensões (`dim_races`, `dim_drivers`, `dim_constructors`) e uma tabela fato unificada (`fact_session_results`), combinando corridas e sprints com flags de negócio (`is_win`, `is_podium`, `has_points`).
- **Analytics:** views SQL com window functions para classificação por temporada e análises de desempenho histórico dos pilotos.

## 📂 Estrutura do repositório

```
f1-lakehouse/
├── 00-common/
│   ├── 01.enviroment-config.ipynb      # Nomes de catálogo/schemas e paths centralizados
│   └── 02.bronze-helpers.ipynb         # Funções auxiliares (metadados de ingestão)
├── 01-setup/
│   └── 01-setup.ipynb                  # Setup inicial do ambiente (catalog, volumes)
├── 02-bronze/                          # Ingestão dos arquivos brutos (circuits, races, constructors, drivers, results, sprints)
├── 03-silver/                          # Limpeza e padronização de cada entidade
├── 04-gold/
│   ├── 01.Build Races Dimension.ipynb
│   ├── 02.Build Constuctors Dimension.ipynb
│   ├── 03.Build Drivers Dimension.ipynb
│   ├── 04.Build Result Fact.ipynb
│   └── 91.Build Nationality Region Reference.ipynb  # Mapeamento manual nacionalidade → região
├── 05-analytics/
│   ├── 01.Build Drivers Standing View.ipynb    # View de classificação por temporada
│   ├── 02.Build Constructor Standing View.ipynb
│   └── 03.Analyze Dominant Drivers.ipynb       # Ranking histórico de "grandeza" dos pilotos
├── jobs/
│   └── f1_pipeline.yml                 # Definição do Job/Workflow (DAG de dependências entre tasks)
└── Formula1 Analytics Dashboard.lvdash.json   # Dashboard Lakeview exportado
```

## 🔄 Pipeline (Job)

O arquivo [`jobs/f1_pipeline.yml`](jobs/f1_pipeline.yml) define o Workflow `job_formula1_lakehouse_full_refresh`, orquestrando as tasks de ingestão e transformação com dependências explícitas — por exemplo, a dimensão de corridas só é construída depois que `circuits` e `races` já passaram pela camada Silver, e a dimensão de pilotos depende tanto da transformação de drivers quanto da referência de nacionalidade/região.

## 📊 Camada Analytics

A view `formula1.gold.v_driver_standing` calcula, por temporada, a posição de cada piloto no campeonato usando `RANK() OVER (PARTITION BY season ORDER BY total_points DESC, number_of_wins DESC)`, com total de pontos, vitórias e pódios agregados a partir da tabela fato.

A partir dela, `03.Analyze Dominant Drivers.ipynb` calcula um "índice de grandeza" (`greatness_score`) combinando campeonatos, vitórias e pódios, para identificar os pilotos historicamente mais dominantes.

## 📈 Dashboard

O dashboard **Driver Championship Standings** ([`Formula1 Analytics Dashboard.lvdash.json`](<Formula1 Analytics Dashboard.lvdash.json>)) traz:

- Tabela de classificação por temporada (posição, piloto, nacionalidade, pontos, vitórias)
- Gráfico de distribuição de vitórias por piloto
- Filtro interativo por temporada

## 🛠️ Tecnologias

- **Databricks** (notebooks, Jobs/Workflows, Unity Catalog, Lakeview Dashboards)
- **PySpark** (ingestão e transformações Bronze/Silver)
- **SQL** (modelagem Gold e views analíticas com window functions)
- **Delta Lake** (formato de armazenamento das tabelas)

## ▶️ Como executar

1. Configure o Unity Catalog com o catálogo `formula1` e os schemas `bronze`, `silver` e `gold`.
2. Suba os arquivos CSV de origem (circuits, races, constructors, drivers, results, sprints) no volume configurado em `00-common/01.enviroment-config.ipynb` (`landing_folder_path`).
3. Execute o Job `job_formula1_lakehouse_full_refresh` (definido em `jobs/f1_pipeline.yml`), ou rode os notebooks manualmente na ordem: `02-bronze` → `03-silver` → `04-gold` → `05-analytics`.
4. Importe `Formula1 Analytics Dashboard.lvdash.json` como um dashboard Lakeview para visualizar os resultados.

## 💡 Aprendizados

- Modelar antes de codar economiza retrabalho.
- A camada semântica (views analíticas) é subestimada e faz toda a diferença na hora de visualizar os dados.
- Dado mal estruturado na entrada vira dor de cabeça em todo o resto do pipeline.
