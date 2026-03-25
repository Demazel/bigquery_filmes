# 🎬 Desafio Técnico - Pipeline de Dados de Filmes

## 📐 Arquitetura do Pipeline
```
CSVs (GCS - Bronze)
        ↓
External Tables (BigQuery - raw)
        ↓
Tabelas Analíticas (BigQuery - analytics)
        ↓
Views Analíticas (BigQuery - analytics)
        ↓
Dashboard (Metabase)
```

## 🛠️ Tecnologias Utilizadas

- **Google Cloud Storage (GCS)** — armazenamento dos arquivos CSV
- **BigQuery** — data warehouse, transformações e views
- **Metabase** — visualização e dashboard
- **Docker** — execução do Metabase localmente

## 📁 Estrutura dos Dados

### Dataset `raw` — dados brutos
| Tabela | Descrição |
|---|---|
| `movies` | Catálogo de filmes |
| `user_rating_history` | Histórico de ratings |
| `ratings_for_additional_users` | Ratings adicionais |

### Dataset `analytics` — dados transformados
| Tabela/View | Descrição |
|---|---|
| `dim_movies` | Dimensão de filmes com ano de lançamento |
| `fact_ratings` | Fato de ratings unificado e limpo |
| `vw_movie_kpis` | KPIs por filme |
| `vw_top_movies` | Top 10 filmes por rating médio |
| `vw_ratings_heatmap` | Atividade por dia e hora |
| `vw_scatter_popularity_vs_quality` | Popularidade vs qualidade |
| `vw_user_activity` | Atividade por usuário |
| `vw_genre_performance` | Performance por gênero |

## 🔧 Passos de Execução

### 1. Criar bucket no GCS
```bash
gcloud storage buckets create gs://desafio123 \
  --location=us-east1
```

### 2. Criar External Tables (raw)
```sql
CREATE OR REPLACE EXTERNAL TABLE `datafriend-491216.raw.movies` (
  movieId STRING, title STRING, genres STRING
)
OPTIONS (
  format = 'CSV',
  uris = ['gs://desafio123/bronze/data_release/movies.csv'],
  skip_leading_rows = 1
);
```

### 3. Criar fact_ratings
```sql
CREATE OR REPLACE TABLE `datafriend-491216.analytics.fact_ratings` AS
WITH all_ratings AS (
  SELECT userId, movieId, rating, tstamp
  FROM `datafriend-491216.raw.user_rating_history`
  UNION ALL
  SELECT userId, movieId, rating, tstamp
  FROM `datafriend-491216.raw.ratings_for_additional_users`
)
SELECT
  SAFE_CAST(r.userId  AS INT64)          AS user_id,
  SAFE_CAST(r.movieId AS INT64)          AS movie_id,
  m.title                                 AS movie_title,
  m.genres                                AS genres,
  SAFE_CAST(r.rating  AS FLOAT64)        AS rating,
  SAFE_CAST(r.tstamp  AS DATETIME)       AS rated_at,
  DATE(SAFE_CAST(r.tstamp AS DATETIME))  AS rated_date
FROM all_ratings r
LEFT JOIN `datafriend-491216.raw.movies` m
  ON SAFE_CAST(r.movieId AS INT64) = SAFE_CAST(m.movieId AS INT64);
```

### 4. Subir o Metabase
```bash
docker run -d -p 3000:3000 --name metabase metabase/metabase
```
Acessar em: http://localhost:3000

## 📊 Dashboard

O dashboard inclui as seguintes visualizações:

- **Evolução de Ratings ao Longo do Tempo** — linha temporal de avg_rating e volume
- **Top 10 Filmes por Rating Médio** — ranking dos filmes mais bem avaliados
- **Análise por Gênero** — média de rating por gênero
- **Popularidade vs Qualidade** — scatter plot de total_ratings x avg_rating
- **Heatmap de Atividade** — volume de ratings por dia da semana e hora

## 💡 Principais Insights

- **Film-Noir** é o gênero com maior rating médio (3.37)
- **The Shawshank Redemption (1994)** lidera o ranking de filmes
- O volume de ratings cresceu exponencialmente a partir de 2015
- Domingos têm o maior pico de atividade de avaliações
