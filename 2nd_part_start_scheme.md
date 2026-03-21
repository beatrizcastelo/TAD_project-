Aqui está o documento com as três sugestões integradas nos sítios certos:

***

## Decisão 1 — O que fica na Fact Table?

Um pedido pode ter vários itens, mas tem apenas **uma** *review* e um conjunto de pagamentos que se referem ao pedido como um todo. Se usássemos uma só *fact table* ao nível do item, o `review_score` e o valor total do pagamento iriam repetir-se em cada linha do mesmo pedido. Por exemplo, um pedido com 3 itens teria o mesmo `review_score` nas três linhas. Isso **não é adequado para análise**, porque métricas do pedido ficariam replicadas e poderiam distorcer qualquer agregação sobre *reviews* ou pagamentos.

Por isso separamos a granularidade em dois níveis:

- A **`Fact_Order_Items`** fica com o que pertence ao **item** (preço, frete, produto, vendedor).
- A **`Fact_Orders`** fica com o que pertence ao **pedido** como um todo (*review*, pagamento agregado, cliente, atraso de entrega).

No modelo de Data Warehousing, isto chama-se **Fact Constellation**: várias tabelas de factos que partilham as mesmas dimensões conformadas.

***

### Fact Table 1: `Fact_Order_Items`

Esta tabela foca-se no **produto** e no **vendedor**.

- **Granularidade:** 1 linha = 1 item vendido dentro de um pedido.
- **Porquê?** Porque um pedido pode ter produtos de categorias diferentes e vendedores diferentes. Este nível de detalhe é necessário para análises de performance por produto e por vendedor.
- **Métricas principais:**
  - `price` — preço do item
  - `freight_value` — custo de frete desse item específico
  - `shipping_limit_date` — data limite de envio pelo vendedor (atributo de data usado para derivar métricas de pontualidade do vendedor)
- **Dimensões ligadas:** `Dim_Produto`, `Dim_Vendedor`, `Dim_Tempo`

***

### Fact Table 2: `Fact_Orders`

Esta tabela foca-se na **experiência do cliente** e na **logística final**.

- **Granularidade:** 1 linha = 1 pedido completo.
- **Porquê?** Porque o cliente avalia a encomenda como um todo (`review_score`) e o atraso de entrega é medido à chegada ao cliente. O pagamento também é agregado ao nível do pedido.
- **Métricas principais:**
  - `total_payment_value` — soma de todos os métodos de pagamento do pedido (calculado no ETL)
  - `payment_type` — tipo de pagamento dominante, ou seja, o de maior valor (calculado no ETL via **Sort** por `payment_value` descendente + **First Row** por `order_id` no Pentaho)
  - `payment_installments` — número máximo de parcelas (calculado no ETL)
  - `num_payment_methods` — número de métodos de pagamento distintos usados (calculado no ETL)
  - `total_freight_value` — soma dos fretes de todos os itens do pedido (calculado no ETL)
  - `review_score` — pontuação de 1 a 5
  - `is_negative_review` — flag binária: 1 se `review_score ≤ 2`, senão 0 (calculado no ETL)
  - `delivery_delay_days` — calculado no ETL: data real de entrega − data estimada de entrega
  - `is_late` — flag binária: 1 se `delivery_delay_days > 0`, senão 0 (calculado no ETL)
  - `number_of_items` — número de itens do pedido (calculado no ETL)
  - `order_status` — dimensão degenerada (coluna direta: delivered, shipped, canceled, etc.)
- **Dimensões ligadas:** `Dim_Cliente`, `Dim_Tempo`

> **Nota sobre `payment_type`:** Como um pedido pode ter múltiplos métodos de pagamento, extrai-se o tipo dominante (o de maior valor) no Pentaho com um **Sort Rows** por `payment_value` descendente seguido de **First Value** agrupado por `order_id`. Isto garante um único valor representativo por pedido sem perder informação relevante.

> **Nota sobre `total_freight_value`:** O `freight_value` por item está na `Fact_Order_Items`. Para responder à **Q2** (relação entre portes e avaliação negativa) sem fazer JOIN entre duas fact tables — o que é uma **má prática em DW** segundo Kimball — agregamos o frete total no ETL e guardamo-lo diretamente em `Fact_Orders`. Isto torna a Q2 numa query de tabela única.

> **Nota sobre `is_negative_review`:** O threshold (≤ 2 estrelas) é pré-calculado no ETL em vez de ser aplicado em runtime. Isto é **Data Pre-aggregation during ETL** e torna a **Q4** (classificação de reviews negativas) diretamente disponível como coluna, sem transformação nas queries analíticas.

> **Nota sobre pagamentos:** A tabela `olist_order_payments_dataset` tem múltiplas linhas por pedido porque um cliente pode pagar com cartão + voucher em simultâneo. Por isso, **não existe uma `Dim_Pagamento`** separada — todos os atributos de pagamento são agregados no ETL e guardados diretamente como colunas em `Fact_Orders`. Não há nenhuma questão do projeto que precise de analisar pagamentos ao nível do método individual.

> **Nota sobre status do pedido:** O `order_status` tem muito poucos valores distintos (7 estados possíveis) e baixa cardinalidade. Em DW, este tipo de atributo é tratado como **dimensão degenerada** — uma coluna simples na fact table, sem tabela de dimensão própria.

***

### Dimensões Partilhadas (Conformadas)

As duas fact tables não são ilhas isoladas — partilham **dimensões conformadas**, o que permite cruzar análises entre os dois níveis:

1. **`Dim_Tempo`:** Ambas usam a mesma tabela de tempo. É possível comparar vendas por mês (`Fact_Order_Items`) com atrasos por mês (`Fact_Orders`).
2. **`order_id`:** Existe nas duas fact tables como chave de integração entre níveis de detalhe, embora o desenho analítico deva manter os factos separados nas queries.

***

### Mapa de Respostas às Questões

| Questão | Fact Table usada | Justificação |
|---|---|---|
| **Q1** (Categorias e Vendas) | `Fact_Order_Items` | Tem a categoria do produto por item |
| **Q2** (Atraso e Portes vs Reviews) | `Fact_Orders` | Tem `delivery_delay_days`, `total_freight_value` e `review_score` no mesmo registo — sem JOIN necessário |
| **Q3** (Performance Vendedores) | `Fact_Order_Items` | Tem o `seller_id` associado a cada item |
| **Q4, Q5, Q6** (Data Mining) | `Fact_Orders` | A unidade de análise é o pedido como um todo; `is_negative_review` e `is_late` já disponíveis como labels |

***

### Resumo

> Optámos por um modelo **Fact Constellation** com duas tabelas de factos para respeitar as diferentes granularidades dos dados:
> 1. **`Fact_Order_Items`**: Granularidade ao nível do item, permitindo análises de produtos e vendedores.
> 2. **`Fact_Orders`**: Granularidade ao nível do pedido, garantindo a integridade métrica do pagamento, do *review score* e dos indicadores logísticos.
>
> Ambas partilham a dimensão conformada `Dim_Tempo`, permitindo análises cruzadas ao longo do tempo. As métricas derivadas (`delivery_delay_days`, `is_late`, `is_negative_review`, `total_freight_value`) são todas pré-calculadas no ETL — **Data Pre-aggregation during ETL** — garantindo performance analítica e consistência.

***

## Dimensões

Agora que definimos as **Fact Tables** (o "quê"), definimos as **Dimension Tables** (o "quem, onde, quando e como").

As dimensões contêm **atributos descritivos** — texto, categorias, localizações. Servem para filtrar, agrupar e dar contexto às métricas das fact tables.

***

### 1. `Dim_Tempo` — O "Quando"

Construída no ETL a partir de `order_purchase_timestamp`. Em vez de guardar apenas a data, cria-se uma **hierarquia temporal completa**:

- **Atributos:** `date_key`, `Data`, `Ano`, `Mês`, `Nome_do_Mês`, `Trimestre`, `Dia_da_Semana` (ex: Segunda), `Fim_de_Semana` (Sim/Não)
- **Porquê?** Responde à **Q1** (evolução ao longo do tempo) e permite detectar padrões sazonais, como atrasos maiores em períodos de época festiva.
- **Pentaho:** Implementada com um step **Table Input** para ler `order_purchase_timestamp`, seguido de **Modified JavaScript Value** para extrair `Ano`, `Mês`, `Trimestre`, `Dia_da_Semana` e calcular o flag `Fim_de_Semana`.

> **Nota sobre Role-Playing (para a apresentação oral):** A `Dim_Tempo` está ligada à `Fact_Orders` apenas uma vez — para o momento em que a venda ocorreu (`order_purchase_timestamp`). O `delivery_delay_days` não requer uma segunda ligação à `Dim_Tempo` porque é um **valor numérico já calculado no ETL** (data real de entrega − data estimada). A ligação à `Dim_Tempo` serve exclusivamente para contextualizar temporalmente os factos — filtrar por mês, trimestre ou ano — não para calcular diferenças de datas em runtime.

***

### 2. `Dim_Produto` — O "O quê"

Construída cruzando `olist_products_dataset` com `product_category_name_translation`.

- **Atributos:** `product_key`, `Product_ID`, `Categoria_Nome_EN`, `Peso_gramas`, `Comprimento_cm`, `Altura_cm`, `Largura_cm`, `Fotos_quantidade`, `Volume_cm3`
- **Porquê?** Responde à **Q1** (categorias com maior volume de vendas) e à **Q5** (produtos mais pesados ou volumosos têm maior risco de atraso — utilizável como variável no modelo preditivo).
- **Pentaho:** Implementada com dois steps **Table Input** ligados por um **Merge Join** na coluna `product_category_name`, seguido de um step **Calculator** para calcular `Volume_cm3 = Comprimento_cm × Altura_cm × Largura_cm`, e **Table Output** para carregar a dimensão.

> **Nota sobre `Volume_cm3`:** Produtos volumosos têm frequentemente logísticas diferentes — transportadoras distintas, maior probabilidade de danos e atrasos. Criar este atributo derivado no ETL enriquece a `Dim_Produto` com uma variável composta que tem maior poder preditivo para a **Q5** do que as dimensões individuais isoladas.

***

### 3. `Dim_Vendedor` — O "Quem vende"

- **Atributos:** `seller_key`, `Seller_ID`, `City`, `State`, `Region`, `Latitude`, `Longitude`
- **Porquê?** Responde à **Q3** (vendedores com melhor e pior desempenho em tempo de entrega e satisfação). Os atributos geográficos estão diretamente nesta dimensão — ver **Decisão 2** abaixo.
- **Pentaho:** Implementada com **Table Input** para `olist_sellers_dataset`, seguido de **Database Lookup** para cruzar com `olist_geolocation_dataset` por `zip_code_prefix`, um step **Group By** para calcular a média de `latitude` e `longitude` por zip code, e **Modified JavaScript Value** para derivar o atributo `Region` a partir de `State`.

***

### 4. `Dim_Cliente` — O "Quem compra"

- **Atributos:** `customer_key`, `Customer_ID`, `City`, `State`, `Region`, `Latitude`, `Longitude`
- **Porquê?** Responde à **Q1** (regiões com maior volume de vendas) e à **Q6** (agrupar perfis de clientes por localização). Os atributos geográficos estão diretamente nesta dimensão — ver **Decisão 2** abaixo.
- **Pentaho:** Processo idêntico ao de `Dim_Vendedor`, aplicado à tabela `olist_customers_dataset`.

***

## Decisão 2 — Como tratar a Geografia: Star Schema Puro

### O Problema

O dataset Olist tem uma tabela de geolocalização separada (`olist_geolocation_dataset`) com coordenadas GPS, cidades e estados por código postal. A primeira abordagem natural seria criar uma `Dim_Location` independente e ligá-la tanto a `Dim_Vendedor` como a `Dim_Cliente`.

O problema é que isso cria **ligações entre dimensões** — o que transforma o modelo num **Snowflake Schema**. O Snowflake aumenta o número de JOINs nas queries analíticas, reduz a performance do OLAP e, acima de tudo, o enunciado do projeto pede explicitamente um **star model**. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/collection_c7b3d64f-8cd6-42f6-96c0-93cecd587598/ec17bed5-a830-4e8a-bf67-99c71e858d2f/2026-TAD-project.pdf)

### A Solução — Desnormalização no ETL com Pentaho

A solução é integrar os atributos geográficos diretamente em `Dim_Vendedor` e `Dim_Cliente` durante o ETL. No Pentaho, isto é feito com a seguinte sequência de steps:

1. **Table Input** — lê `olist_sellers_dataset` (ou `olist_customers_dataset`)
2. **Database Lookup** — cruza com `olist_geolocation_dataset` por `zip_code_prefix` para obter `city`, `state`, `latitude`, `longitude`
3. **Group By** — calcula a média de `latitude` e `longitude` por zip code (necessário porque a tabela de geolocalização tem múltiplas coordenadas por código postal)
4. **Modified JavaScript Value** — deriva o atributo `Region` a partir de `State` (ex: SP → Sudeste, BA → Nordeste)
5. **Table Output** — carrega a dimensão final no Data Warehouse

O resultado é que `Dim_Vendedor` e `Dim_Cliente` ficam **autossuficientes**, com todos os atributos geográficos necessários sem depender de nenhuma outra dimensão.

### Porquê Esta Decisão é a Correta

Esta técnica chama-se **desnormalização intencional** e é uma das características fundamentais do star schema segundo Kimball. Num sistema transacional (OLTP), duplicar dados é um problema. Num Data Warehouse (OLAP), é a norma — porque o objetivo é performance de leitura e simplicidade de queries, não eficiência de escrita.

| | Snowflake (`Dim_Location` separada) | Star Puro (desnormalização no ETL) |
|---|---|---|
| **JOINs nas queries** | Mais | Menos |
| **Performance OLAP** | Menor | Maior |
| **Complexidade do modelo** | Maior | Menor |
| **Alinhamento com enunciado** | ❌ | ✅ |
| **Redundância de dados** | Menor | Maior (aceitável em DW) |

> *"Optámos por um **Pure Star Schema**, desnormalizando os atributos geográficos diretamente em `Dim_Vendedor` e `Dim_Cliente` durante a fase de ETL no Pentaho. Isto elimina dependências entre dimensões, reduz a complexidade das queries analíticas e está alinhado com a metodologia de Kimball e com os requisitos do projeto."*

***

### Checklist de Questões vs. Dimensões

| Questão | Dimensões usadas |
|---|---|
| **Q1** (Regiões e Categorias) | `Dim_Cliente` (State, Region), `Dim_Produto` (Categoria), `Dim_Tempo` |
| **Q2** (Atraso e Portes vs Reviews) | `Fact_Orders` diretamente (`delivery_delay_days`, `total_freight_value`, `is_negative_review`); filtrar por `Dim_Tempo` e `Dim_Cliente` |
| **Q3** (Performance Vendedores) | `Dim_Vendedor` (com Region e State), `Dim_Tempo` |
| **Q4** (Prever Reviews Negativas) | `is_negative_review` como label; `Dim_Produto` (peso, `Volume_cm3`), `Dim_Cliente` e `Dim_Vendedor` (região) como features |
| **Q5** (Prever Atrasos) | `is_late` como label; `Dim_Produto` (peso, `Volume_cm3`), `Dim_Cliente` + `Dim_Vendedor` (distância interestadual) como features |
| **Q6** (Clustering de Encomendas) | `Dim_Produto`, `Dim_Cliente`, `Dim_Vendedor`, colunas de pagamento em `Fact_Orders` |

***

### Dica para o Poster: O Star Schema Visual

No poster, desenha as duas fact tables no centro e as 4 dimensões em volta (`Dim_Tempo`, `Dim_Produto`, `Dim_Vendedor`, `Dim_Cliente`), com linhas a ligá-las. Não há ligações entre dimensões — é um star puro. Destaca que `Dim_Tempo` é **conformada** — partilhada pelas duas fact tables — e que os atributos geográficos foram integrados nas dimensões de vendedor e cliente durante o ETL no Pentaho.