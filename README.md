# LH Nautical — Desafio Lighthouse Indicium

> Solução completa do Desafio Técnico de Dados & IA da Indicium Academy.  
> **Analista:** Natalia Moraes | **Data:** Março de 2026  
> **Ferramentas:** Python 3 · DuckDB · Pandas · Scikit-learn

---

## Sobre o projeto

A LH Nautical é uma distribuidora de equipamentos náuticos que opera com uma assimetria estrutural relevante: receitas registradas em BRL e custos atrelados ao USD. Como o câmbio varia diariamente e as bases nunca foram integradas, parte do portfólio era vendida abaixo do custo sem que ninguém percebesse.

Este projeto integra dados de vendas, custos de importação e câmbio diário do Banco Central para revelar o impacto real: **R$ 39,9 milhões em prejuízo operacional** ao longo de 2023–2024, além de construir um diagnóstico completo com segmentação de clientes, análise temporal, previsão de demanda e sistema de recomendação.

O desafio é composto por **8 questões técnicas** (24 subquestões no total) e conta com um **relatório executivo** (`index.html`) disponível via GitHub Pages, voltado ao público não técnico.

---

## Estrutura do repositório

```
repositorio/
├── index.html                             ← Relatório executivo (GitHub Pages)
├── lh_nautical_analise_dados.html         ← Notebook exportado para visualização
├── lh_nautical_analise_dados.ipynb        ← Notebook principal com toda a análise
├── data/
│   ├── vendas_2023_2024.csv               ← Dataset de vendas (9.895 registros)
│   ├── produtos_raw.csv                   ← Catálogo de produtos bruto (157 registros)
│   ├── custos_importacao.json             ← Custos de importação (estrutura aninhada)
│   └── custos_importacao_normalizado.csv  ← JSON normalizado (1.260 registros)
└── outputs/
    ├── visual_prejuizo_produtos.png
    ├── visual_clientes_elite.png
    ├── visual_clientes_faturamento.png
    ├── visual_vendas_dia_semana.png
    ├── visual_recomendacao.png
    └── visual_categoria_elite.png
```

---

## Por que DuckDB no notebook?

O notebook utiliza uma conexão global `conn = duckdb.connect()`, instanciada uma única vez na **Seção 4** e reutilizada ao longo de todas as análises. Essa escolha não foi arbitrária — ela reflete um padrão deliberado de arquitetura analítica.

**Validação cruzada Python ↔ SQL.** Cada cálculo relevante foi primeiro implementado em Pandas/Python e depois reproduzido em SQL via DuckDB. Os resultados idênticos entre as duas abordagens funcionam como teste de consistência — se os números batem, a lógica está correta. Isso é especialmente crítico na Q4, onde a lógica de custo vigente por data (`start_date <= data_venda`) precisa ser reproduzível e auditável.

**SQL diretamente sobre DataFrames em memória.** O DuckDB permite executar queries SQL sobre DataFrames Pandas via `conn.register('nome', df)` sem precisar de banco de dados externo, arquivo intermediário ou infraestrutura distribuída. Isso reproduz o padrão híbrido de pipelines modernos de dados — transformação em Python, agregação e validação em SQL — dentro de um único ambiente de notebook.

**Expressividade para consultas analíticas complexas.** Construções como `generate_series` (Q6 — calendário completo), `LATERAL JOIN` com subconsulta ordenada (Q4 — custo vigente por data) e CTEs encadeadas (Q5 — clientes elite) são mais legíveis e próximas da linguagem de negócio em SQL do que em código Python imperativo. O DuckDB suporta essas construções nativamente, sem limitações do SQLite ou da necessidade de subir um PostgreSQL.

**Eficiência no cache de câmbio.** Na Q4, o cache de 726 taxas PTAX foi construído em Python via API do BCB e depois registrado no DuckDB como DataFrame. Isso permitiu fazer o JOIN entre vendas, câmbio e custos em uma única query SQL, sem loops Python sobre 9.895 registros.

Em resumo: **Python para ingestão, tratamento e modelagem; DuckDB para validação, agregação e consultas analíticas que se beneficiam da linguagem SQL.** As questões que o desafio pedia em SQL foram respondidas com SQL puro compatível com PostgreSQL/DuckDB — sem dependência de sintaxe proprietária do Python.

---

## Questões respondidas

> O notebook completo com todos os códigos, outputs e visualizações está disponível em [`lh_nautical_analise_dados.ipynb`](lh_nautical_analise_dados.ipynb) e também na versão exportada [`lh_nautical_analise_dados.html`](lh_nautical_analise_dados.html).

---

### Q1 — EDA (`vendas_2023_2024.csv`)
> **Notebook:** Seções 6.1 a 6.4

**Cenário:** O Sr. Almir quer saber se pode confiar nos dados antes de tomar qualquer decisão sobre análises, modelagens ou precificação.

#### Q1.1 — SQL

A query abaixo reproduz o que foi executado na **Seção 6.4** do notebook via DuckDB, convertida para SQL padrão:

```sql
SELECT
    COUNT(*)           AS total_linhas,
    MIN(sale_date)     AS data_minima,
    MAX(sale_date)     AS data_maxima,
    MIN(total)         AS total_minimo,
    MAX(total)         AS total_maximo,
    AVG(total)         AS total_medio
FROM vendas_2023_2024;
```

> **Resultados:** 9.895 linhas · 6 colunas (`id`, `id_client`, `id_product`, `qtd`, `total`, `sale_date`) · Período: 01/01/2023 a 31/12/2024 · Mínimo: R$ 294,50 · Máximo: R$ 2.222.973,00 · Média: R$ 263.797,83 · Nulos: zero em todas as colunas.

#### Q1.2 — Validação

> **Valor máximo registrado na coluna `total`:** R$ 2.222.973,00

#### Q1.3 — Interpretação

O dataset `vendas_2023_2024.csv` apresenta boa integridade estrutural — zero valores nulos em todas as colunas — e cobertura temporal completa de janeiro de 2023 a dezembro de 2024. Porém, três pontos exigem atenção antes de qualquer análise subsequente.

**Outliers em `total`:** o valor máximo (R$ 2.222.973,00) é aproximadamente 8,4 vezes maior que a média (R$ 263.797,83), indicando pedidos atípicos que podem distorcer métricas agregadas como ticket médio e faturamento total.

**Formatos de data inconsistentes:** a coluna `sale_date` apresenta formatos mistos — identificados no notebook na Seção 6.2 — exigindo padronização com `pd.to_datetime(..., format='mixed', dayfirst=True)` antes de qualquer operação temporal ou integração com outras bases.

**Ausência de integração:** o dataset registra apenas receitas em BRL. Análises de margem dependem do cruzamento com custos em USD e câmbio diário — sem essa integração, qualquer diagnóstico financeiro é estruturalmente incompleto.

**Conclusão:** a base está pronta para análises, desde que os três pontos acima sejam tratados. As inconsistências identificadas são sintomas de falta de validações na origem, risco que se amplifica com o crescimento do volume de dados. A consistência entre os cálculos Python e SQL (Seção 6.4) validou a integridade da camada de ingestão.

---

### Q2 — Normalização de Produtos (`produtos_raw.csv`)
> **Notebook:** Seções 7.1 a 7.4

**Cenário:** Gabriel percebeu que os dados de produtos estão desorganizados — categorias com múltiplas grafias, preços em formato textual, registros duplicados. A missão é normalizar `produtos_raw.csv` usando Python 3.

#### Q2.1 — Código Python

Extraído fielmente das **Seções 7.1 e 7.2** do notebook:

```python
import pandas as pd
import unicodedata

df_produtos = pd.read_csv('data/produtos_raw.csv')

# ── Parte 1: Padronização de categorias ──────────────────────────────────
def normalizar_texto(texto):
    texto = str(texto).strip().lower()
    texto = unicodedata.normalize('NFKD', texto)
    texto = ''.join(c for c in texto if not unicodedata.combining(c))
    texto = texto.replace(' ', '')
    return texto

def padronizar_categoria(cat):
    cat = normalizar_texto(cat)
    if any(x in cat for x in ['eletron', 'eletr', 'eletrun', 'eletro']):
        return 'eletrônicos'
    elif any(x in cat for x in ['propul', 'propus', 'propuc', 'prop']):
        return 'propulsão'
    elif any(x in cat for x in ['ancor', 'ankor', 'encor']):
        return 'ancoragem'
    else:
        return cat

df_produtos['actual_category'] = df_produtos['actual_category'].apply(padronizar_categoria)

# ── Parte 2: Conversão de price para numérico ────────────────────────────
df_produtos['price'] = (
    df_produtos['price']
    .str.replace('R$', '', regex=False)
    .str.strip()
    .astype(float)
)

# ── Parte 3: Remoção de duplicatas ───────────────────────────────────────
total_antes = len(df_produtos)
df_produtos = df_produtos.drop_duplicates()
total_depois = len(df_produtos)
duplicatas_removidas = total_antes - total_depois

print(f"Registros antes: {total_antes}")
print(f"Registros depois: {total_depois}")
print(f"Duplicatas removidas: {duplicatas_removidas}")
print(f"Categorias finais: {sorted(df_produtos['actual_category'].unique())}")
```

> **Decisão de design:** a padronização de categorias é aplicada **antes** da deduplicação. Sem essa ordem, registros como `Eletronicoz` e `ELETRONICOS` seriam tratados como produtos distintos e não seriam removidos no `drop_duplicates()`. A sequência correta — normalizar, depois deduplicar — foi a que revelou os 7 registros duplicados que a inspeção inicial não identificava.

#### Q2.2 — Validação

> **Produtos duplicados removidos:** 7 (de 157 para 150 registros)

---

### Q3 — Normalização de Custos de Importação (`custos_importacao.json`)
> **Notebook:** Seções 8.1 a 8.4

**Cenário:** o arquivo JSON armazena o histórico de preços de compra em estrutura aninhada dentro do campo `historic_data`. Esse formato não é compatível com operações relacionais — joins, agregações e filtros por data — sendo necessário expandir para uma tabela relacional.

#### Q3.1 — Código Python

Extraído fielmente das **Seções 8.1 e 8.2** do notebook:

```python
import json
import pandas as pd

with open('data/custos_importacao.json', 'r', encoding='utf-8') as f:
    raw = json.load(f)

# Estrutura raiz: lista de 150 produtos, cada um com campo 'historic_data'
registros = []
for produto in raw:
    product_id   = produto['product_id']
    product_name = produto['product_name']
    category     = produto['category']
    for entrada in produto['historic_data']:
        registros.append({
            'product_id':   product_id,
            'product_name': product_name,
            'category':     category,
            'start_date':   entrada['start_date'],
            'usd_price':    entrada['usd_price']
        })

df_custos = pd.DataFrame(registros)

# Ajuste de tipos
df_custos['product_id'] = df_custos['product_id'].astype(int)
df_custos['start_date'] = pd.to_datetime(
    df_custos['start_date'], format='mixed', dayfirst=True
)

# Ordenar por produto e data
# Requisito técnico para o filtro start_date <= data_venda da Q4
df_custos = df_custos.sort_values(['product_id', 'start_date'])

# Salvar CSV normalizado
df_custos.to_csv('data/custos_importacao_normalizado.csv', index=False)

print(f"Total de entradas no CSV: {len(df_custos)}")
```

> **Por que iteração explícita e não `pd.json_normalize()`?** Conforme documentado na Seção 8.4 do notebook: a abordagem iterativa oferece maior controle sobre a estrutura gerada e permite validações granulares por registro. O `format='mixed'` com `dayfirst=True` foi necessário devido à presença de múltiplos formatos de data na base original.

#### Q3.2 — Validação

> **Total de entradas no CSV após normalização:** 1.260

---

### Q4 — Análise de Margem com Dados Públicos (câmbio BCB/PTAX)
> **Notebook:** Seções 9.1 a 9.7

**Cenário:** o Sr. Almir suspeita que produtos foram vendidos abaixo do custo. O sistema de vendas registra valores em BRL; os custos dos fornecedores estão em USD; e o câmbio varia diariamente. Ninguém havia cruzado essas três fontes até este projeto.

**Premissas aplicadas:**
- Câmbio: cotação de venda PTAX do Banco Central do Brasil na data da venda (fallback retroativo de até 4 dias úteis para fins de semana e feriados)
- Custo vigente: último `usd_price` com `start_date <= data_venda`
- Custo total por transação: `custo_usd × taxa_cambio × qtd`
- Receita total inclui todas as transações, inclusive as lucrativas
- Impostos e frete desconsiderados

#### Q4.1 — SQL

A lógica abaixo reproduz a query executada na **Seção 9.5** do notebook, convertida para SQL padrão com `LATERAL JOIN` no lugar do join feito via `conn.register` no DuckDB:

```sql
WITH custo_vigente AS (
    SELECT
        v.id            AS venda_id,
        v.id_product,
        v.total,
        v.qtd,
        v.sale_date,
        c.taxa_cambio,
        ci.usd_price                                 AS custo_usd,
        ci.usd_price * c.taxa_cambio * v.qtd         AS custo_brl,
        (ci.usd_price * c.taxa_cambio * v.qtd)
            - v.total                                AS prejuizo,
        CASE
            WHEN (ci.usd_price * c.taxa_cambio * v.qtd) > v.total
            THEN TRUE ELSE FALSE
        END                                          AS tem_prejuizo
    FROM vendas_2023_2024 v
    -- dim_cambio: tabela com a PTAX BCB por data, pré-carregada via API (Seção 9.2)
    JOIN dim_cambio c
        ON c.data_cotacao = v.sale_date
    -- custo vigente: último preço registrado antes ou na data da venda
    JOIN LATERAL (
        SELECT usd_price
        FROM custos_importacao_normalizado ci
        WHERE ci.product_id = v.id_product
          AND ci.start_date <= v.sale_date
        ORDER BY ci.start_date DESC
        LIMIT 1
    ) ci ON TRUE
)
SELECT
    id_product,
    ROUND(SUM(total), 2)                                           AS receita_total,
    ROUND(SUM(CASE WHEN tem_prejuizo THEN prejuizo ELSE 0 END), 2) AS prejuizo_total,
    ROUND(
        SUM(CASE WHEN tem_prejuizo THEN prejuizo ELSE 0 END)
        / SUM(total),
        4
    )                                                              AS percentual_perda
FROM custo_vigente
GROUP BY id_product
HAVING SUM(CASE WHEN tem_prejuizo THEN prejuizo ELSE 0 END) > 0
ORDER BY prejuizo_total DESC
LIMIT 10;
```

> **Nota:** no notebook (Seção 9.3), a lógica de custo vigente foi implementada em Python via função `get_custo_vigente()` com apply e depois validada via SQL no DuckDB (Seção 9.5). A `dim_cambio` corresponde ao dicionário `cache_cambio` construído na Seção 9.2 — 726 datas únicas consultadas via API BCB/PTAX com fallback de 4 dias, cobertura de 100%.

#### Q4.2 — Validação

> **`id_produto` com maior percentual de perda relativa:** produto **72** — percentual de perda: **63,37%** da receita total

#### Q4.3 — Interpretação

**Câmbio utilizado:** cotação de venda PTAX do Banco Central do Brasil (endpoint `CotacaoDolarDia`), selecionando o último valor disponível no dia (`dados[-1]['cotacaoVenda']`). Para fins de semana e feriados bancários, foi aplicada busca retroativa de até 4 dias úteis — prática consistente com o comportamento do BCB, que não divulga PTAX em dias não úteis. Das 726 datas únicas na base de vendas, 100% obtiveram taxa de câmbio.

**Definição de prejuízo:** uma transação apresenta prejuízo quando `custo_brl > total`, ou seja, quando o custo de importação convertido para BRL na data da venda supera o valor de venda registrado.

**Suposição relevante:** o custo foi calculado como `custo_usd × taxa_cambio × qtd`, ignorando impostos e frete conforme premissa do desafio. Em contexto produtivo, esses componentes deveriam ser incluídos para refletir o custo total de desembaraço.

O resultado revela que **62,8% das transações foram realizadas abaixo do custo de importação**. O produto 72 concentra R$ 39,9 milhões em prejuízo absoluto e também o maior percentual de perda (63,37%), evidenciando um problema estrutural de precificação — não eventos pontuais, mas uma defasagem sistemática entre preços de venda e variação cambial acumulada.

---

### Q5 — Análise de Clientes Elite
> **Notebook:** Seções 10.1 a 10.5

**Cenário:** a Diretoria quer identificar os clientes fiéis — alto ticket médio e diversidade de categorias — para replicar esse comportamento em outros segmentos.

#### Q5.1 — SQL

Extraído e adaptado da **Seção 10.1** do notebook (query `query_q5` executada via DuckDB):

```sql
WITH vendas_com_categoria AS (
    SELECT
        v.id_client,
        v.id              AS venda_id,
        v.total,
        v.qtd,
        p.actual_category
    FROM vendas_2023_2024 v
    LEFT JOIN produtos_normalizados p
        ON v.id_product = p.code
),
metricas_cliente AS (
    SELECT
        id_client,
        SUM(total)                                        AS faturamento_total,
        COUNT(DISTINCT venda_id)                          AS frequencia,
        ROUND(SUM(total) / COUNT(DISTINCT venda_id), 2)   AS ticket_medio,
        COUNT(DISTINCT actual_category)                   AS diversidade_categorias
    FROM vendas_com_categoria
    GROUP BY id_client
),
clientes_elite AS (
    SELECT *
    FROM metricas_cliente
    WHERE diversidade_categorias >= 3
    ORDER BY ticket_medio DESC, id_client ASC
    LIMIT 10
)
SELECT * FROM clientes_elite;
```

Para identificar a categoria mais vendida entre os Top 10 (pergunta da Seção 10.2):

```sql
WITH clientes_elite AS (
    -- CTE acima
),
categoria_top10 AS (
    SELECT
        p.actual_category,
        SUM(v.qtd) AS total_itens
    FROM vendas_2023_2024 v
    LEFT JOIN produtos_normalizados p
        ON v.id_product = p.code
    WHERE v.id_client IN (SELECT id_client FROM clientes_elite)
    GROUP BY p.actual_category
    ORDER BY total_itens DESC
)
SELECT * FROM categoria_top10;
```

#### Q5.2 — Validação

> **Categoria mais vendida (maior `sum(qtd)`) entre os Top 10 clientes elite:** **propulsão** — 6.030 itens

#### Q5.3 — Explicação

**Limpeza das categorias:** aplicada via `unicodedata.normalize('NFKD')` combinado com correspondência parcial por substrings (`eletron`, `propul`, `ancor`), conforme implementado na Seção 7.2. A abordagem captura variações extremas como `E L E T R Ô N I C O S` e `Eletronicoz` sem depender de dicionário fixo — resiliente a novas grafias futuras. O `LEFT JOIN` com `produtos_normalizados` (base já tratada na Q2) garantiu que a diversidade fosse calculada sobre categorias padronizadas.

**Filtro de diversidade mínima:** `WHERE diversidade_categorias >= 3` foi aplicado após o agrupamento, com diversidade calculada como `COUNT(DISTINCT actual_category)` já sobre categorias normalizadas — garantindo que variações de grafia não inflem artificialmente a contagem.

**Contagem de itens restrita ao Top 10:** implementada com `WHERE v.id_client IN (SELECT id_client FROM clientes_elite)`, assegurando que apenas as transações dos 10 clientes selecionados entrassem no cálculo de `SUM(qtd)` por categoria.

---

### Q6 — Dimensão de Calendário
> **Notebook:** Seções 11.1 a 11.3

**Cenário:** o Sr. Almir quer saber qual dia da semana tem a menor média de vendas. Um estagiário calculou a média com `GROUP BY` direto na tabela de vendas e inflou o resultado por ignorar os dias em que a loja operou sem registrar vendas.

#### Q6.1 — SQL

Extraído da **Seção 11.1** do notebook (query `query_q6` executada via DuckDB), com sintaxe equivalente em SQL padrão:

```sql
WITH calendario AS (
    SELECT
        CAST(
            generate_series(
                (SELECT MIN(sale_date) FROM vendas_2023_2024),
                (SELECT MAX(sale_date) FROM vendas_2023_2024),
                INTERVAL '1 day'
            ) AS DATE
        ) AS data
),
vendas_diarias AS (
    SELECT
        c.data,
        COALESCE(SUM(v.total), 0) AS valor_venda
    FROM calendario c
    LEFT JOIN vendas_2023_2024 v
        ON c.data = v.sale_date
    GROUP BY c.data
),
dia_semana AS (
    SELECT
        data,
        valor_venda,
        CASE EXTRACT(DOW FROM data)
            WHEN 0 THEN 'Domingo'
            WHEN 1 THEN 'Segunda-feira'
            WHEN 2 THEN 'Terça-feira'
            WHEN 3 THEN 'Quarta-feira'
            WHEN 4 THEN 'Quinta-feira'
            WHEN 5 THEN 'Sexta-feira'
            WHEN 6 THEN 'Sábado'
        END AS nome_dia
    FROM vendas_diarias
)
SELECT
    nome_dia,
    ROUND(AVG(valor_venda), 2) AS media_vendas,
    COUNT(*)                    AS total_dias
FROM dia_semana
GROUP BY nome_dia
ORDER BY media_vendas ASC;
```

#### Q6.2 — Validação

> **Dia com menor média de vendas:** **Domingo** — média de **R$ 3.229.614,16**

#### Q6.3 — Explicação

**Por que usar uma tabela de datas (calendário) em vez de agrupar diretamente a tabela de vendas?**

A tabela de vendas contém apenas os dias em que houve transações. Quando se aplica `GROUP BY` diretamente sobre ela, os dias sem venda são ignorados — como se não existissem. Isso infla as médias de todos os dias da semana, pois os dias com zero vendas não entram no denominador do cálculo. O calendário gerado com `generate_series` cobre todos os dias do período. O `LEFT JOIN` mantém esses dias no resultado, e o `COALESCE(..., 0)` substitui os nulos por zero, produzindo uma média correta e auditável. O notebook documenta exatamente esse raciocínio na Seção 11 — o título da seção começa com "Um erro comum em análises de sazonalidade é calcular médias diretamente sobre a tabela de vendas."

**O que aconteceria se um dia da semana tivesse muitos dias sem venda?**

A média seria artificialmente elevada. Se a loja não registrou vendas em 20 das 40 terças-feiras do período, uma análise sem calendário calcularia a média apenas sobre as 20 terças com vendas, ignorando as outras 20 com valor zero. O resultado pareceria indicar que terça-feira é um bom dia de vendas, quando na realidade metade das terças não gerou receita alguma.

---

### Q7 — Previsão de Demanda
> **Notebook:** Seções 12.1 a 12.4

**Cenário:** o Sr. Almir quer um modelo preditivo para ajustar as compras com fornecedores do produto "Motor de Popa Yamaha Evo Dash 155HP" e evitar tanto rupturas de estoque quanto excesso de itens encalhados.

#### Q7.1 — Código Python

Extraído fielmente das **Seções 12.1 e 12.2** do notebook:

```python
import pandas as pd

PRODUTO = 'Motor de Popa Yamaha Evo Dash 155HP'

# Identificar id do produto
id_produto = df_produtos[df_produtos['name'] == PRODUTO]['code'].values[0]
# id_produto = 54

# Agregar vendas diárias do produto
df_produto = df_vendas[df_vendas['id_product'] == id_produto].copy()
df_diario = (
    df_produto
    .groupby('sale_date')['qtd']
    .sum()
    .reset_index()
)
df_diario.columns = ['data', 'qtd_vendida']

# Criar calendário completo (incluindo dias sem venda)
datas_completas = pd.date_range(
    start=df_diario['data'].min(),
    end='2024-01-31',
    freq='D'
)
df_diario = (
    df_diario
    .set_index('data')
    .reindex(datas_completas, fill_value=0)
    .reset_index()
    .rename(columns={'index': 'data'})
)

# Separar treino e teste
treino = df_diario[df_diario['data'] <= '2023-12-31'].copy()
teste  = df_diario[df_diario['data'] >= '2024-01-01'].copy()

# Baseline: média móvel de 7 dias sem data leakage
serie_completa = df_diario.set_index('data')['qtd_vendida']

previsoes = []
for data in teste['data']:
    # Apenas dados anteriores à data — sem data leakage
    historico = serie_completa[serie_completa.index < data].tail(7)
    previsao  = historico.mean() if len(historico) > 0 else 0
    previsoes.append({
        'data':     data,
        'real':     serie_completa.get(data, 0),
        'previsao': round(previsao, 2)
    })

df_previsoes = pd.DataFrame(previsoes)

# Métrica MAE
mae = (df_previsoes['real'] - df_previsoes['previsao']).abs().mean()
print(f"MAE (Mean Absolute Error): {mae:.4f}")

print(f"\nSoma total prevista (01/01 a 07/01): "
      f"{df_previsoes[df_previsoes['data'] <= '2024-01-07']['previsao'].sum():.0f}")
print(f"Soma total real    (01/01 a 07/01): "
      f"{df_previsoes[df_previsoes['data'] <= '2024-01-07']['real'].sum():.0f}")
```

#### Q7.2 — Validação

> **Soma total da previsão para a 1ª semana de janeiro de 2024 (01/01 a 07/01):** **3 unidades** (valor real registrado: 10 unidades)

#### Q7.3 — Explicação

**Como o baseline foi construído:** para cada dia do período de teste (janeiro de 2024), o modelo calcula a média das vendas diárias dos 7 dias imediatamente anteriores à data prevista. A série inclui os dias sem venda (valor zero) — o mesmo princípio do calendário completo aplicado na Q6 — garantindo que a janela de 7 dias reflita o comportamento real do produto, inclusive os períodos de baixa demanda.

**Como o data leakage foi evitado:** o filtro `serie_completa.index < data` usa estritamente menor do que, conforme documentado na Seção 12.2 do notebook. Isso garante que a própria data prevista nunca entre no cálculo da janela de 7 dias. Substituir `<` por `<=` incluiria o valor real do dia sendo previsto no cálculo, contaminando o modelo com informação futura e invalidando completamente a avaliação de performance.

**Uma limitação do modelo:** o produto apresenta demanda intermitente — alta frequência de dias com zero vendas e picos esporádicos. A média móvel simples suaviza esses picos e sistematicamente subestima a demanda nos períodos críticos. Na primeira semana de janeiro, o modelo previu 3 unidades contra 10 reais, representando 70% de erro exatamente no período mais relevante para decisões de reposição. Conforme indicado na Seção 12.4, modelos específicos para demanda intermitente, como Croston ou Prophet com sazonalidade semanal, são o próximo passo natural — usando o MAE de 1,64 como benchmark mínimo a superar.

---

### Q8 — Sistema de Recomendação
> **Notebook:** Seções 13.1 a 13.4

**Cenário:** a Marina quer implementar uma vitrine "Quem comprou isso, também levou..." no site, baseada em similaridade de comportamento de compra dos clientes, sem depender de infraestrutura de Big Data.

#### Q8.1 — Código Python

Extraído fielmente das **Seções 13.1 e 13.2** do notebook:

```python
from sklearn.metrics.pairwise import cosine_similarity
import pandas as pd

# ── 13.1 Construção da Matriz Usuário x Produto ──────────────────────────
# Construir matriz binária de interação
df_matriz = (
    df_vendas
    .groupby(['id_client', 'id_product'])['qtd']
    .sum()
    .reset_index()
)
df_matriz['comprou'] = 1  # binarização — ignora quantidade

matriz = df_matriz.pivot_table(
    index='id_client',
    columns='id_product',
    values='comprou',
    fill_value=0
)

print(f"Dimensão da matriz: {matriz.shape}")
# Saída: (49, 150) — 49 clientes × 150 produtos

# Calcular similaridade de cosseno entre produtos
# Transposta: produtos nas linhas, clientes nas colunas
matriz_T    = matriz.T
similaridade = cosine_similarity(matriz_T)
df_similaridade = pd.DataFrame(
    similaridade,
    index=matriz_T.index,
    columns=matriz_T.index
)

# ── 13.2 Ranking de Similaridade — GPS Garmin Vortex Maré Drift ──────────
PRODUTO_REF = 'GPS Garmin Vortex Maré Drift'

id_gps = df_produtos[df_produtos['name'] == PRODUTO_REF]['code'].values[0]
print(f"ID do produto de referência: {id_gps}")

similares = (
    df_similaridade[id_gps]
    .drop(id_gps)               # remove o próprio produto
    .sort_values(ascending=False)
    .head(5)
    .reset_index()
)
similares.columns = ['id_product', 'similaridade']

# Adicionar nome do produto
similares = similares.merge(
    df_produtos[['code', 'name']],
    left_on='id_product',
    right_on='code'
).drop('code', axis=1)

display(similares[['id_product', 'name', 'similaridade']])
```

> **Top 5 produtos mais similares ao GPS Garmin Vortex Maré Drift:**
> 1. Motor de Popa Volvo Magnum 276HP (`id_product = 94`) — similaridade: 0,871
> 2. GPS Furuno Swift Leviathan Poseidon (`id_product = 11`) — similaridade: 0,871
> 3. Radar Furuno Swift (`id_product = 35`) — similaridade: 0,850
> 4. Cabo de Nylon Delta Force Magnum Leviathan (`id_product = 115`) — similaridade: 0,850
> 5. Transponder AIS Maré Magnum (`id_product = 1`) — similaridade: 0,850

#### Q8.2 — Validação

> **`id_produto` com MAIOR similaridade ao "GPS Garmin Vortex Maré Drift":** produto **94** — Motor de Popa Volvo Magnum 276HP (similaridade: **0,871**)

#### Q8.3 — Explicação

**Como a matriz foi construída:** agrupadas todas as transações por `id_client` e `id_product`, o valor de cada célula foi binarizado — 1 se o cliente comprou o produto ao menos uma vez, 0 caso contrário. A quantidade foi deliberadamente ignorada para que a similaridade reflita padrão de comportamento de compra e não volume, evitando que produtos de alta rotatividade dominem por quantidade e não por co-ocorrência real. A matriz resultante tem dimensão 49 clientes × 150 produtos.

**O que significa a similaridade de cosseno nesse contexto:** a similaridade de cosseno mede o ângulo entre os vetores de dois produtos no espaço de clientes. Cada produto é representado por um vetor binário com 49 posições — uma por cliente — onde 1 indica que aquele cliente comprou o produto. Dois produtos são similares quando tendem a ser comprados pelos mesmos clientes. Um valor de 0,871 entre o GPS e o Motor de Popa Volvo indica perfis de compradores quase idênticos: clientes que investem em navegação de precisão tendem a equipar embarcações com motores de alta potência, sugerindo perfil de uso voltado a navegação de longo curso.

**Uma limitação desse método:** a base de apenas 49 clientes reduz a robustez estatística das similaridades. Com poucos clientes, valores elevados podem ser resultado de 2 ou 3 compradores em comum — não de um padrão comportamental genuíno e generalizável. Conforme documentado na Seção 13.3 do notebook, o sistema deve ser interpretado como prova de conceito. A expansão da base de clientes é o principal requisito para uso em produção.

---

## Visuais gerados

| Arquivo | Conteúdo | Seção |
|---|---|---|
| `visual_prejuizo_produtos.png` | Top 15 produtos por prejuízo absoluto | 9.6 |
| `visual_clientes_elite.png` | Volume de compras por categoria — Top 10 clientes | 10.3 |
| `visual_clientes_faturamento.png` | Faturamento acumulado dos Top 10 clientes | 10.4 |
| `visual_vendas_dia_semana.png` | Média de vendas por dia da semana (com dias sem venda) | 11.2 |
| `visual_recomendacao.png` | Previsão de demanda: real vs. previsto — janeiro 2024 | 12.3 |
| `visual_categoria_elite.png` | Top 5 produtos mais similares ao GPS Garmin Vortex Maré Drift | 13.2 |

---

## Principais achados

| Dimensão | Resultado |
|---|---|
| Integridade dos dados | 9.895 registros · zero nulos · cobertura 2023–2024 |
| Produtos normalizados | 157 → 150 após remoção de 7 duplicatas |
| Custos normalizados | 1.260 registros tabulares a partir do JSON aninhado |
| Transações com prejuízo | 6.213 de 9.895 — **62,8% do total** |
| Maior prejuízo absoluto | Produto 72 — **R$ 39,9 milhões** (63,37% da receita) |
| Ticket médio — clientes elite | Entre **R$ 290.063** e **R$ 336.859** |
| Categoria líder — clientes elite | **Propulsão** — 6.030 itens |
| Pior dia de vendas | **Domingo** — R$ 3.229.614,16 |
| Melhor dia de vendas | **Sexta-feira** — R$ 3.776.151,25 |
| MAE do modelo baseline | **1,64 unidades/dia** (demanda intermitente) |
| Produto mais similar ao GPS | Motor de Popa Volvo Magnum 276HP — similaridade **0,871** |

---

## Relatório executivo

O arquivo `index.html` na raiz do repositório contém o relatório executivo completo para o público não técnico (Marina e Sr. Almir), disponível via **GitHub Pages**.

Estrutura: Resumo Executivo · O Problema de Negócio · Principais Descobertas · Impacto Financeiro · Recomendações de Ação (0–30 / 30–90 / 90+ dias) · Oportunidades de Crescimento · Apêndice Técnico.

---

## Stack técnico

| Biblioteca | Papel no projeto |
|---|---|
| `pandas` | Manipulação e transformação de dados tabulares |
| `numpy` | Operações numéricas e vetorização |
| `duckdb` | SQL sobre DataFrames em memória — validação cruzada e consultas analíticas |
| `requests` | Consumo da API PTAX/BCB para taxa de câmbio oficial |
| `scikit-learn` | Similaridade de cosseno para o sistema de recomendação |
| `unicodedata` | Normalização de strings para padronização de categorias |
| `matplotlib` / `seaborn` | Geração dos visuais analíticos |
