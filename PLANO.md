# Plano de execução — MVP Engenharia de Dados

Mapa completo: `docs/mapa-mvp.html`. Este arquivo é o checklist operacional.
**Dataset decidido em 11/09/2026: Beach Volleyball (Kaggle `jessemostipak/beach-volleyball`).**

## Decisões travadas (independem do dataset)
| Item | Escolha |
|---|---|
| Plataforma | Databricks Free Edition (serverless) |
| Arquitetura | Medalhão: `bronze` / `silver` / `gold` |
| Formato | Delta Lake |
| Linguagem | PySpark na ingestão/limpeza, SQL nos agregados e análise |
| Versionamento | Git folder do Databricks → repositório público no GitHub |
| Dataset | **Beach Volleyball** — `kagglehub.dataset_download("jessemostipak/beach-volleyball")` → `vb_matches.csv` (76.756 partidas, 2000–2019, FIVB + AVP, 65 colunas) |

## Matriz de decisão do dataset (preenchida em 11/09)
Pontuar 0–3 e multiplicar pelo peso. Máximo 54; viável a partir de 36.
Licença não declarada ou volume irredutível **eliminam** o candidato.

| Critério | Peso | Beach Volleyball | F1 (rohanrao) | Olist |
|---|---|---|---|---|
| Tabelas relacionadas com PK/FK identificáveis | 3 | 2 | 3 | 3 |
| Métrica numérica que sustente um fato | 3 | 3 | 3 | 3 |
| Coluna temporal utilizável (12+ meses) | 2 | 3 | 3 | 2 |
| Sujeira real para tratar | 2 | 3 | 3 | 2 |
| Documentação das colunas na fonte | 2 | 3 | 2 | 3 |
| Interesse pessoal / valor de portfólio | 2 | 3 | 1 | 1 |
| Dimensão geográfica ou categórica | 1 | 3 | 3 | 3 |
| Licença explícita (eliminatório) | 2 | 2 | 3 | 3 |
| Volume compatível com Free Edition (eliminatório) | 1 | 3 | 3 | 3 |
| **Total** | 18 | **49** | 48 | 46 |

Outros candidatos de vôlei avaliados: PlusLiga 2008–2023 (CC0, mas 160 KB e nomes já
unificados), combo VNL 2021–2024 (3 datasets, <500 KB, licença do principal não declarada),
NCAA boxscores 2012–2019 (licença não declarada → eliminado).

### Ficha do dataset escolhido
- **Fonte de coleta:** Kaggle, `jessemostipak/beach-volleyball`, via `kagglehub` (script `00_coleta_kaggle.py`).
- **Licença a registrar no README:** página do Kaggle marca "Unknown"; o arquivo é o `vb_matches.csv`
  do repositório TidyTuesday (`rfordatascience/tidytuesday`, semana 2020-05-19), distribuído sob
  **CC0-1.0**, compilado por Adam Vagnar a partir de resultados públicos da FIVB Beach Volleyball
  World Tour e da AVP. Citar as duas URLs e a data do download.
- **Dicionário de colunas:** readme do TidyTuesday 2020-05-19.
- **Sujeira conhecida na origem:** estatísticas detalhadas ausentes em ~81% das partidas; datas de
  nascimento antes de 1970 com erro de 100 anos; colunas duplicadas; formato wide
  (`w_p1_*`, `w_p2_*`, `l_p1_*`, `l_p2_*`).
- **Esboço de modelo:** `dim_atleta`, `dim_torneio`, `dim_data`, `fato_partida` (grão partida) e
  `fato_desempenho_atleta` (grão atleta × partida, via unpivot das 4 posições).

Onde procurar: Kaggle · Dados Abertos gov.br / Portal da Transparência ·
Databricks Public Datasets · IMDb · Yelp · Google Cloud Public Datasets ·
API ou scraping (caso avançado) · dados anonimizados da empresa.

## Checklist

### Ambiente — 09–10/09 (faça agora, não depende do dataset)
- [ ] Conta Databricks Free Edition criada
- [ ] Repositório **público** no GitHub criado
- [ ] Git folder do Databricks conectado ao repositório (PAT configurado)
- [ ] `CREATE SCHEMA bronze / silver / gold`
- [ ] `CREATE VOLUME bronze.raw_files`
- [ ] `.gitignore` com `*.csv`, `*.zip`, `data/`

### Dataset e objetivo — 10–11/09
- [x] Candidatos pontuados na matriz e decisão tomada (Beach Volleyball, 11/09)
- [ ] Licença registrada no README: fonte, data do download, o que a licença obriga
- [ ] Problema escrito: quem tem, o que não se sabe, qual decisão depende, custo de não saber
- [ ] 6 a 8 perguntas escritas **antes** de abrir os dados, variando o tipo
      (volume, evolução temporal, comparativa, relacional, concentração, segmentação, operacional)
- [ ] Inventário dos arquivos brutos: linhas, colunas, grão, PK, FK

### Bronze — 12–13/09
- [ ] Upload dos arquivos para `/Volumes/<catalogo>/bronze/raw_files/`
- [ ] `notebooks/01_ingestao_bronze.py`: arquivo → Delta, 1 tabela por arquivo
- [ ] Metadados `_ingestao_ts`, `_arquivo_origem`, `_camada` em todas as tabelas
- [ ] Escrita idempotente (`overwrite` + `overwriteSchema`)
- [ ] Loop parametrizado, não um bloco por arquivo
- [ ] Contagem de linhas conferida com a origem
- [ ] Screenshots: upload no Volume + schema `bronze`
- [ ] Nenhum `dropna`/`cast`/rename nesta camada

### Qualidade — 14–15/09
- [ ] `notebooks/02_perfilamento_qualidade.py` → tabela `gold.qualidade_perfil`
- [ ] Completude: % de nulos e vazios por coluna; separar nulo estrutural de nulo por falha
- [ ] Unicidade: `count` vs. `count distinct` em cada chave candidata
- [ ] Consistência: formatos, caixa, acentuação, encoding, unidades, domínio de categóricos
- [ ] Acurácia: datas fora de ordem, negativos indevidos, valores implausíveis no contexto
- [ ] Outliers: mínimo, máximo, percentis
- [ ] Inventário problema → tratamento → qual pergunta é afetada

### Silver — 16–18/09
- [ ] `notebooks/03_silver_limpeza.py`: uma tabela limpa por entidade
- [ ] Renomeação padronizada, cast de datas e decimais, deduplicação, normalização de texto
- [ ] Colunas derivadas que as perguntas exigem
- [ ] Flags de exceção em vez de exclusão de linhas (rastreabilidade)

### Gold — 19–21/09
- [ ] Grão do fato declarado em uma frase, por escrito, antes de codar
- [ ] `04_gold_dimensoes.py`: dimensões conformadas + `dim_data` gerada por sequência
- [ ] `05_gold_fatos.py`: fato(s) no grão declarado, com FKs e métricas
- [ ] Grãos diferentes → fatos diferentes (evitar fan-out)
- [ ] `06_gold_agregados.sql`: 2–3 tabelas `agg_*` nos cortes mais usados
- [ ] Screenshots dos schemas `silver` e `gold` persistidos

### Catálogo — 22/09
- [ ] `07_catalogo_comentarios.sql`: `COMMENT ON TABLE` + comentário em toda coluna
- [ ] Domínio de valores no comentário (faixa para numérico, lista para categórico)
- [ ] Linhagem no comentário (coluna de origem ou fórmula)
- [ ] `PRIMARY KEY` / `FOREIGN KEY ... NOT ENFORCED` nas tabelas gold
- [ ] `docs/catalogo_de_dados.md` transcrito
- [ ] Screenshots: Overview com comentários, Lineage, diagrama de entidades

### Análise — 23–24/09
- [ ] `08_analise_perguntas.py`, uma seção por pergunta
- [ ] Padrão de resposta: pergunta → consulta → resultado → 2–3 frases com o número citado
- [ ] Resultados negativos também respondidos ("não há relação entre X e Y")
- [ ] Discussão geral amarrando tudo ao problema original
- [ ] Screenshot de cada resultado

### Fechamento — 25–27/09
- [ ] Job encadeando `01 → 02 → 03 → 04 → 05 → 06 → 07 → 09`
- [ ] `09_validacao_final.py`: contagens, unicidade de PK, integridade referencial, faixas
- [ ] Screenshot do grafo do Job e da execução bem-sucedida
- [ ] README com as 7 seções obrigatórias e títulos exatos
- [ ] Autoavaliação: atingido, não atingido e por quê, dificuldades, trabalhos futuros
- [ ] Repositório público, notebooks commitados, links verificados

## Seções obrigatórias do README (títulos exatos)
1. Contexto de Negócios e Perguntas (Etapa 2 e 4.1)
2. Carga dos Dados (Etapa 4.2)
3. Modelagem e Catálogo de Dados (Etapa 4.3)
4. Pipeline de Dados (Etapa 4.4)
5. Qualidade de Dados (Etapa 4.5)
6. Análise de Dados (Etapa 4.5)
7. Autoavaliação
