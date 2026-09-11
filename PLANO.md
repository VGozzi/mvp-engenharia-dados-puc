# Plano de execução — MVP Engenharia de Dados

Mapa completo: `docs/mapa-mvp.html`. Este arquivo é o checklist operacional.
Repositório: `github.com/VGozzi/mvp-engenharia-dados-puc`. Entrega: **27/09/2026**.

## Decisões travadas
| Item | Escolha |
|---|---|
| Plataforma | Databricks Free Edition (serverless) |
| Arquitetura | Medalhão: `bronze` / `silver` / `gold` |
| Formato | Delta Lake |
| Linguagem | PySpark na coleta/ingestão/limpeza, SQL nos agregados e análise |
| Versionamento | Git folder do Databricks → este repositório |
| Dataset | **Beach Volleyball** — Kaggle `jessemostipak/beach-volleyball` (decidido em 11/09) |

## Dataset: Beach Volleyball
- **Arquivo:** `vb_matches.csv` — 76.756 partidas, 65 colunas, 2000–2019, FIVB World Tour + AVP,
  masculino e feminino. Uma linha por partida, em formato wide: 2 vencedores (`w_p1_*`, `w_p2_*`)
  e 2 perdedores (`l_p1_*`, `l_p2_*`).
- **Coleta:** `kagglehub.dataset_download("jessemostipak/beach-volleyball")` no notebook
  `00_coleta_kaggle.py`, copiando para `/Volumes/<catalogo>/bronze/raw_files/`. Credenciais
  Kaggle via Databricks Secrets, nunca no código.
- **Licença (texto para o README):** distribuído sob CC0-1.0 pelo projeto TidyTuesday
  (`rfordatascience/tidytuesday`, semana 2020-05-19), compilado por Adam Vagnar a partir de
  resultados públicos da FIVB e da AVP; espelhado no Kaggle (página marca "Unknown").
  Registrar as duas URLs e a data do download.
- **Dicionário de colunas:** readme do TidyTuesday 2020-05-19.
- **Sujeira conhecida na origem:** estatísticas detalhadas (ataques, kills, aces, bloqueios,
  defesas) ausentes em ~81% das partidas; datas de nascimento antes de 1970 com erro de 100
  anos; colunas duplicadas; nomes de atletas sem chave — dedup por nome + nascimento.
- **Modelo alvo (gold):**
  - `dim_atleta` — nome, nascimento, altura, país, gênero
  - `dim_torneio` — circuito (FIVB/AVP), nome, país, ano, gênero
  - `dim_data` — gerada por sequência
  - `fato_partida` — grão: **uma partida**; FKs para torneio, data, 4 atletas; placar, duração, ranking
  - `fato_desempenho_atleta` — grão: **um atleta em uma partida** (unpivot das 4 posições);
    ataques, kills, erros, aces, bloqueios, defesas, flag vencedor

## Checklist

### Ambiente — até 12/09
- [ ] Conta Databricks Free Edition criada
- [x] Repositório **público** no GitHub criado (11/09)
- [x] `.gitignore` com `*.csv`, `*.zip`, `*.parquet`, `data/` (11/09)
- [ ] Git folder do Databricks conectado ao repositório (PAT configurado)
- [ ] `CREATE SCHEMA bronze / silver / gold`
- [ ] `CREATE VOLUME bronze.raw_files`
- [ ] Secret scope com `KAGGLE_USERNAME` / `KAGGLE_KEY`

### Objetivo — até 12/09
- [x] Dataset decidido (Beach Volleyball, 11/09)
- [ ] Problema escrito: quem tem, o que não se sabe, qual decisão depende, custo de não saber
- [ ] 6 a 8 perguntas escritas **antes** de abrir os dados, variando o tipo
      (volume, evolução temporal, comparativa, relacional, concentração, segmentação, operacional)
- [ ] Licença registrada no README: fonte, data do download, o que a licença obriga

### Coleta e Bronze — 13–14/09
- [ ] `notebooks/00_coleta_kaggle.py`: kagglehub → Volume `bronze.raw_files`
- [ ] `notebooks/01_ingestao_bronze.py`: `vb_matches.csv` → `bronze.vb_matches` (Delta), tudo como string
- [ ] Metadados `_ingestao_ts`, `_arquivo_origem`, `_camada`
- [ ] Escrita idempotente (`overwrite` + `overwriteSchema`)
- [ ] Contagem de linhas conferida com a origem (76.756)
- [ ] Inventário do arquivo bruto: linhas, colunas, grão, chaves candidatas
- [ ] Screenshots: arquivo no Volume + tabela no schema `bronze`
- [ ] Nenhum `dropna`/`cast`/rename nesta camada

### Qualidade — 15–16/09
- [ ] `notebooks/02_perfilamento_qualidade.py` → tabela `gold.qualidade_perfil`
- [ ] Completude: % de nulos por coluna; separar nulo estrutural (estatísticas não coletadas
      no torneio) de nulo por falha
- [ ] Unicidade: partida (torneio + data + 4 atletas) e atleta (nome + nascimento)
- [ ] Consistência: formatos de data, caixa/acentuação de nomes e países, colunas duplicadas
- [ ] Acurácia: nascimento com erro de 100 anos, idade/altura implausíveis, placar × vencedor,
      duração fora de faixa
- [ ] Outliers: mínimo, máximo, percentis das métricas
- [ ] Inventário problema → tratamento → qual pergunta é afetada

### Silver — 17–19/09
- [ ] `notebooks/03_silver_limpeza.py`: `silver.partida`, `silver.atleta`, `silver.desempenho_atleta`
- [ ] Renomeação padronizada, cast de datas e numéricos, correção dos nascimentos, dedup de atletas
- [ ] Unpivot das 4 posições (`w_p1`, `w_p2`, `l_p1`, `l_p2`) para o grão atleta × partida
- [ ] Colunas derivadas que as perguntas exigem (idade na partida, ano, década, sets)
- [ ] Flags de exceção em vez de exclusão de linhas (rastreabilidade)

### Gold — 20–22/09
- [ ] Grão de cada fato declarado em uma frase, por escrito, antes de codar
- [ ] `04_gold_dimensoes.py`: `dim_atleta`, `dim_torneio`, `dim_data`
- [ ] `05_gold_fatos.py`: `fato_partida` e `fato_desempenho_atleta`, com FKs e métricas
- [ ] `06_gold_agregados.sql`: 2–3 tabelas `agg_*` nos cortes das perguntas
- [ ] Screenshots dos schemas `silver` e `gold` persistidos

### Catálogo — 23/09
- [ ] `07_catalogo_comentarios.sql`: `COMMENT ON TABLE` + comentário em toda coluna
- [ ] Domínio de valores no comentário (faixa para numérico, lista para categórico)
- [ ] Linhagem no comentário (coluna de origem ou fórmula)
- [ ] `PRIMARY KEY` / `FOREIGN KEY ... NOT ENFORCED` nas tabelas gold
- [ ] `docs/catalogo_de_dados.md` transcrito
- [ ] Screenshots: Overview com comentários, Lineage, diagrama de entidades

### Análise — 24–25/09
- [ ] `08_analise_perguntas.py`, uma seção por pergunta
- [ ] Padrão de resposta: pergunta → consulta → resultado → 2–3 frases com o número citado
- [ ] Resultados negativos também respondidos ("não há relação entre X e Y")
- [ ] Discussão geral amarrando tudo ao problema original
- [ ] Screenshot de cada resultado

### Fechamento — 26–27/09
- [ ] Job encadeando `00 → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 09`
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
