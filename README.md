# DT·AI Module — San Marco Brescia | Prologis

## Visão geral

Módulo de inteligência artificial responsável por classificar itens por padrão de demanda, mapear políticas de gestão de estoque e gerar os parâmetros de simulação para o Digital Twin (DT) do projeto San Marco Brescia / Prologis.

O output final é a tabela `simulation_table.xlsx` — uma linha por item com cluster, política de inventário e parâmetros (SS, ROP, EOQ, s, S) — que alimenta diretamente o módulo de simulação do DT.

## Estrutura do projeto

```
dt_ai_module/
├── data/                          # Dados de entrada (não versionados)
│   ├── Industrial_case_20251203.xlsx
│   ├── Variability.xlsx
│   └── Alloy_Brass_price.xlsx
├── notebooks/
│   ├── 01_feature_extraction.ipynb  # Fase 1 e 2 — features de demanda
│   ├── 02_clustering.ipynb          # Fase 3 — clustering de itens
│   ├── 03_policy_parameters.ipynb   # Fase 4 — parâmetros SS, ROP, EOQ
│   ├── 04_cascade.ipynb             # Fase 5 — lógica de cascata
│   └── 05_export.ipynb              # Fase 6 — exportação para o DT
├── outputs/
│   ├── feature_table.csv
│   ├── clustering_table.csv
│   ├── params_table.csv
│   ├── cascade_table.csv
│   └── simulation_table.xlsx        # Entrega final para o DT
└── README.md
```

## Pipeline

### Fase 1 — Feature extraction (`01_feature_extraction.ipynb`)
Carrega o histórico de pedidos (52.289 linhas, 1.388 itens, jan/2023–dez/2024) e calcula as features de demanda por item: média mensal, desvio padrão, ADI, CV², volume anual, número de clientes e pedidos, e tendência.

### Fase 2 — Clustering (`02_clustering.ipynb`)
Classifica os itens em dois eixos independentes:

- **Eixo 1 — Regularidade:** ADI e CV² (limiares: ADI=1.32, CV²=0.49) → stable, erratic, intermittent, lumpy
- **Eixo 2 — Perfil comercial:** volume anual e número de clientes (threshold: mediana) → high/low volume, many/few clients

A combinação dos dois eixos gera 5 clusters de política: MTS, Comakership, Batch-to-Order, Dynamic MTS/MTO e MTO.

### Fase 3 — Parâmetros (`03_policy_parameters.ipynb`)
Calcula SS, ROP, EOQ, s e S por item usando parâmetros reais extraídos dos arquivos de dados:

- **SS:** distribuição Normal (ADI < 1.32) ou Gamma (ADI ≥ 1.32)
- **LT:** 7.66 dias (calculado via OEE das máquinas do Variability.xlsx)
- **H:** €0.70/kg/ano para CW614N, €0.90/kg/ano para CW724R
- **S_setup:** €100/ordem (média drawing e extrusion)

### Fase 4 — Cascata (`04_cascade.ipynb`)
Atribui o nível de produção a cada item.

> **Nota (v1):** A cascata completa requer uma BoM com quantidades de conversão entre níveis, que não está disponível nos dados atuais. Na v1, o nível é determinado diretamente pelo cluster: MTS/Comakership/Batch-to-Order/Dynamic → `finished_products`, MTO → `raw_material`.

### Fase 5 — Exportação (`05_export.ipynb`)
Monta e exporta a tabela final de simulação em CSV e Excel, com abas separadas por nível de produção e uma aba de resumo por cluster.

---

## Resultados

| Cluster | N Itens | Modelo | Nível |
|---|---|---|---|
| Dynamic | 721 | (s,S) | finished_products |
| MTO | 422 | — | raw_material |
| MTS | 241 | EOQ-ROP | finished_products |
| Comakership | 4 | EOQ-ROP | finished_products |

---

## Ambiente

```bash
conda create -n dt_ai python=3.11
conda activate dt_ai
pip install pandas numpy scipy scikit-learn openpyxl jupyter
```

---

## Limitações e próximos passos (v2)

- Implementar cascata completa com BoM quantificada
- Refinar thresholds de volume e número de clientes com validação operacional
- Adicionar modelagem Gamma completa para itens intermitentes
- Incorporar setup time real por item do OrdiniProduzione
