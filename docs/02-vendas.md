# MC MOTO → Vendas

A maior parte do conteúdo desta aba vive dentro do iframe `mapa-vendas.html` (cópia publicada de `Mapa de Vendas.html`, gerado por `atualizar_mapa.py` — ver [`06-pipeline-dados.md`](06-pipeline-dados.md)). O `index.html` faz deep-link para as sub-abas internas desse iframe via `trocarAba(id)` (Painel Mensal, Vendas Diárias, Grupo de Produto, Consistência por Grupo, Venda por Fornecedor). **Exceção**: a sub-aba **Vendas Históricas** é uma aba nativa de `index.html` (não passa pelo iframe do Mapa) — ver seção própria ao final.

Termo recorrente: **"fator de desconto" (`descontoPct`)** — um campo global, editável pelo usuário e persistido em `localStorage`, que ajusta o valor de venda "bruto" para simular o efeito de descontos concedidos. É aplicado como `f = 1 - descontoPct/100`. **Importante: esse fator não é aplicado de forma consistente em todas as sub-abas** — ver seção "Inconsistências conhecidas" ao final.

## Painel Mensal (vendedores)

Fonte: uma consulta por vendedor que já **líquida devoluções** — vendas e custo são reduzidos pelas devoluções vinculadas a vendas do próprio período, desde que a venda original não esteja cancelada.

- **MC (margem de contribuição)**: `calcMc(vendas, custo) = vendas*(1 - descontoPct/100) - custo`
- **Projeção do mês**: `realizado / diasUteisDecorridos * diasUteisTotal`
- **Dias úteis**: contados excluindo domingos e datas cadastradas como feriado (textarea configurável). `diasUteisDecorridos` **sempre conta até ontem**, nunca até hoje — para não distorcer a projeção com um dia de faturamento ainda incompleto.
- **Metas**: por vendedor e por campo (vendas/MC/pedidos/itens), guardadas em `localStorage` (chave `mapa_meta_{vendedor}_{campo}`).
- **% da meta**: `realizado / meta`, colorido de verde (`pct-ok`) se `≥ 100%`, vermelho (`pct-bad`) caso contrário.
- Colunas e sub-colunas (Realizado/Projeção/Meta/%) são reordenáveis por arrastar-e-soltar, com a ordem persistida em `localStorage`.

## Vendas Diárias

Dados por dia: pedidos, venda bruta (`VL_TOTAL`, **sem** líquido de devoluções), custo, itens. No render:
```
mc = venda * f - custo   // custo NÃO é ajustado pelo fator de desconto
```

## Grupo de Produto

Soma vendas/quantidade/MC por grupo para o mês corrente. **Diferente do Painel Mensal, aqui NÃO há líquido de devoluções** — a consulta não faz join com a tabela de devoluções. `mc = venda - custo`, sem qualquer ajuste de desconto neste nível de agregação.

Produtos sem grupo definido (`DIVERSOS`/sem grupo) passam por uma reclassificação por palavra-chave na descrição, em duas listas ordenadas: frases compostas primeiro (ex. "DISCO FREIO" → grupo FREIO), depois palavras isoladas (ex. "PNEU" → grupo PNEUMATICO) — primeira correspondência vence.

## Consistência por Grupo

A tabela mais rica em regras de negócio do painel. Colunas: Grupo | Vendas no mês | MC no mês | MC % | Projeção do mês | Média histórica/mês | Proj. vs média | Particip. atual | Particip. histórica | Situação.

### Fórmulas por linha

```
f              = 1 - descontoPct/100
vendaAtual     = venda_do_grupo_no_mes * f
mcAtual        = mc_do_grupo_no_mes * f      // mc já vem pré-calculado (venda - custo) do lado do servidor
mcPct          = vendaAtual > 0 ? mcAtual / vendaAtual : 0
projecao       = diasUteisDecorridos > 0 ? vendaAtual / diasUteisDecorridos * diasUteisTotal : 0
media          = mediaHistoricaMensal_do_grupo * f     // fonte depende do filtro de período, ver abaixo
variacao       = media > 0 ? (projecao/media - 1) : (projecao > 0 ? 1 : null)
shareAtual     = vendaAtual / total_de_vendaAtual_de_todos_os_grupos
shareHist      = participação histórica do grupo (pré-calculada)
```

⚠️ Nuance na fórmula de `mcAtual`: diferente do Painel Mensal (que aplica o desconto só na receita: `venda*f - custo`), aqui o desconto multiplica a **margem já calculada** (`(venda-custo)*f`). Matematicamente equivalente apenas se `custo` também fosse multiplicado por `f`, o que não é o caso — ou seja, o valor de MC exibido aqui **não é diretamente comparável** ao MC calculado do mesmo jeito no Painel Mensal ou em Clientes.

A tabela inclui tanto grupos com venda no mês corrente quanto grupos que venderam historicamente mas nada este mês (aparecem com `vendaAtual = 0`).

### Filtro de período histórico: "Tudo" vs 3 / 6 / 12 meses

Existem **duas fontes de dado histórico diferentes**, não apenas uma janela de tempo diferente:

- **"Tudo"** → usa a média pré-calculada no lado do Python (`buscar_media_grupo`): soma todos os meses desde uma data-base fixa, **excluindo explicitamente o mês corrente**, dividido pelo número de meses completos. Cobre **todos os grupos**, sem limite de quantidade.
- **"3 / 6 / 12 meses"** → recalculado no navegador a partir de `P.histGrupo.meses` (histórico mensal completo, que **inclui o mês corrente, mesmo que parcial**), pegando os últimos N meses dessa lista. Esse histórico mensal só guarda os **top 7 grupos por receita + um grupo agregado "Outros"** — grupos menores fora do top-7 não têm dado sob os filtros 3/6/12 (aparecem com média zero/vazia), mesmo que apareçam normalmente sob "Tudo".

**Consequência prática**: trocar de "Tudo" para "3/6/12 meses" não é só encurtar a janela — muda se o mês corrente conta na média e restringe quais grupos têm histórico disponível.

### Banner de alinhamento (participação atual vs. histórica)

```
overlap = Σ min(shareAtual_grupo, shareHist_grupo)  para todos os grupos
alinhamento = round(overlap * 100)
```
- `≥ 90%` → "bem condizente" (verde)
- `≥ 80%` → "razoavelmente condizente" (âmbar)
- `< 80%` → "divergente" (vermelho)

O mesmo banner reporta contagens de grupos acima da média (`nAcima`), abaixo (`nAbaixo`) e com margem baixa (`nMargemBaixa`, ver abaixo).

### Coluna "Situação" (projeção vs. média histórica)

Banda de tolerância de **±10%** sobre `variacao`:
- `variacao > +10%` → 🔼 "Acima"
- `variacao < -10%` → 🔽 "Abaixo"
- entre -10% e +10% → ✅ "Dentro"
- sem dado (`variacao === null`) → "—"

### Destaque de margem baixa/alta (`margemBaixa`)

```
mcPctMedioGeral = Σ(mcAtual de todos os grupos) / Σ(vendaAtual de todos os grupos)   // média ponderada, não simples
margemBaixa = vendaAtual > 0 AND mcPct < mcPctMedioGeral - 10 pontos percentuais
```
Grupo com margem baixa recebe destaque vermelho (`pct-bad`, com ⚠️) na célula de MC%. Simetricamente, margem **mais de 10 p.p. acima** da média recebe destaque verde (`pct-ok`). O total de grupos com margem baixa aparece em negrito vermelho no banner-resumo.

### Interatividade
- Filtro de período: botões `Tudo / 3 / 6 / 12 meses`.
- Colunas clicáveis para ordenar (venda, MC, MC%), com indicador visual ▲/▼ do sentido atual.

## Venda por Fornecedor

Baseado nos últimos 12 meses (**sem** líquido de devoluções). Por fornecedor:
```
cmv_mensal = cmv_12m / 12
cobertura  = cmv_mensal > 0 ? estoque_a_custo / cmv_mensal : (estoque > 0 ? null : 0)   // meses de estoque
giro       = estoque > 0 ? cmv_12m / estoque : null
mcPct      = vendas > 0 ? (vendas - cmv) / vendas : null
```
- `cobertura === null` com estoque positivo → rotulado **"sem giro"** (vermelho, destaque forte — há estoque parado sem nenhuma venda no período).
- **"Estoque parado"** (filtro disponível na interface): `estoque > 0 AND (cobertura === null OR cobertura > 6)` — ou seja, **mais de 6 meses de cobertura projetada conta como estoque parado**, destacado em âmbar na coluna de cobertura.

## Clientes (ranking e histórico)

- Ranking do mês corrente: **líquido de devoluções**, top 100 clientes por venda.
- Histórico: agregação mensal por cliente (últimos 12 meses + mês corrente), também líquida de devoluções, mas restrito aos **top 150 clientes por venda histórica total** — clientes fora desse top não têm gráfico de evolução disponível.
- Filtro de período (`atual / 3 / 6 / 12 meses`) reagrega a partir do histórico retido.
- `mc = vendas*f - custo`; `pctMc = mc / (vendas*f)`; célula vermelha (`pct-bad`) apenas quando `pctMc < 0` (sem banda de ±10% aqui, diferente de Consistência por Grupo).
- Ticket médio = `vendas / pedidos`.
- Checkbox "CLIENTE CONSUMIDOR" permite excluir vendas de balcão/consumidor anônimo do ranking.

## Vendas Históricas (aba nativa de `index.html`)

Diferente das demais sub-abas de Vendas, esta **não** vive no iframe do Mapa — é a aba nativa `app-tab-vendashistoricas`, com iframe próprio (`vendas-historicas.html`), aberta por `abrirSubAbaVendasHist`. A página é gerada pelo script local `gerar_vendas_historicas_mcmoto.py` (não versionado; roda na rotina diária `atualizacao-diaria-painel`, Parte A) a partir do banco `mc_moto`.

- Mostra o **faturamento mensal de todo o histórico disponível** (65 meses, desde 2021 — não só 12).
- **Definição de "vendas líquidas"** idêntica à do Painel Mensal: `SUM(vendas.VL_TOTAL)` de vendas não canceladas (`STATUS <> '3'`) **menos devoluções** (`SUM(devolucoes.VL_TOTAL_LIQ)`), atribuídas ao mês da venda original. Custo líquido = `SUM(itens.QUANTIDADE*itens.CUSTO)` menos custo devolvido; `MC = vendas − custo`. *(O CMV mensal deste cálculo bate com a variável `CMV_MENSAL` do painel — validado.)*
- Filtro de período (12 / 24 meses / todo o histórico), gráfico de barras por mês, e tabela por mês com: pedidos, itens, vendas líquidas, MC, MC%, ticket médio (`vendas/pedidos`) e **variação vs. o mesmo mês do ano anterior**.
- O mês corrente é marcado como **parcial** e fica de fora da comparação anual (senão compararia mês incompleto com mês cheio).

## Vendas por Grupo de Comissão (aba nativa de `index.html`)

Também aba nativa (`app-tab-vendascomissao`, iframe `vendas-comissao.html`, aberta por `abrirSubAbaVendasComissao`), gerada pelo script local `gerar_vendas_comissao_mcmoto.py` (roda na rotina diária, Parte A) a partir do banco `mc_moto`.

- Agrupa as vendas pela **faixa de comissão do produto** (`produtos.COMISSAO_AVISTA`): 8%, 5%, 2%, 1%, 0,5%, 0%, "Sem comissão" (e o que mais existir). Cada faixa aparece separada.
- **Vendas líquidas** com a mesma netagem de devoluções/estornos do CMV e do pico (na própria linha de `itens`): `SUM(VL_UNIT × (QUANTIDADE − QTD_DEVOLVIDA − QTD_ESTORNADA))`; custo análogo; `MC = vendas − custo`.
- Filtro de período (3 / 6 / 12 meses / todo o histórico), **gráfico de barras por faixa** (valor + participação %), e tabela com participação %, MC, MC%, quantidade líquida e **Comissão** (= vendas líq. × alíquota da faixa), além de uma linha de total.
- Nota: a distribuição atual concentra-se na faixa **1%** (~59% das vendas) e **5%** (~30%); as faixas 8% e 5% são as que o restante do painel marca como "Não comprar"/"A avaliar" (regra `nc`).

**Visão "Por vendedor (comissão)"** (botão *Visão* na barra de controles): serve para **apurar a comissão a receber de cada vendedor**. O vendedor vem de `vendas.VENDEDOR` → `vendedores.DESCRICAO` (join de `itens` com `vendas`); a comissão de cada linha = **vendas líquidas da faixa × alíquota da faixa** (a alíquota é a própria `produtos.COMISSAO_AVISTA`; "Sem comissão" e "0%" = 0).
  - Seletor de vendedor. Em **"Todos"**: tabela-resumo com Vendedor | Vendas líq. | Qtd | Alíquota efetiva | **Comissão a receber**, ordenada pela comissão, com total geral (= "comissão total a pagar"). Clicar num vendedor abre o detalhamento dele.
  - Vendedor específico: detalhamento **por faixa** (Vendas líq. | Part.% | Qtd | Alíquota | Comissão) com total = comissão a receber daquele vendedor. Respeita o filtro de período.
  - Payload embute `linhasVend` = `[ym, vendedor, faixaLbl, faixaSort, vendas, mc, qtd]`; a visão "Por faixa" é derivada agregando os vendedores, então os totais de comissão das duas visões batem exatamente.
  - **Regra de cadastro (decisão do usuário):** `COMISSAO_AVISTA = 100` é sempre tratado como **1%** (correção de erro de cadastro). Aplicada em `label_com` de `gerar_vendas_comissao_mcmoto.py` **e** `gerar_graficos_mcmoto.py`, então esses itens caem na faixa 1% (rótulo, agrupamento e comissão a 1%) em ambas as páginas.

## Gráficos (aba nativa de `index.html`)

Aba nativa `app-tab-graficos` (iframe `graficos-mcmoto.html`, aberta por `abrirSubAbaGraficos`), gerada pelo script local `gerar_graficos_mcmoto.py` (não versionado; roda na rotina diária `atualizacao-diaria-painel`, Parte A, passo A11) a partir do banco `mc_moto`. É um **painel consolidado de gráficos SVG** (sem bibliotecas externas) com filtro de período **12 / 24 meses** (o payload embute 24 meses; o navegador reagrega).

- **KPIs do período:** venda total, margem de contribuição (R$ e MC%), ticket médio e nº de pedidos.
- **Venda histórica** — barras verticais do faturamento líquido por mês.
- **Margem de contribuição** — barras de MC (R$) por mês com linha sobreposta de MC% (eixo direito).
- **Ticket médio** — linha (com área) do valor médio por pedido por mês.
- **Grupo de mercadoria** — barras horizontais por grupo de produto (top 12 + "Outros"), com participação % e MC%.
- **Grupo de comissão** — gráfico de **linhas ao longo dos meses** (mesmo formato do Ticket médio), uma linha por faixa `produtos.COMISSAO_AVISTA` mostrando a evolução mensal das vendas líquidas, com legenda de cores.
- **% desconto na MC:** a página tem um campo *% desconto* que usa a **mesma chave `localStorage` `mapa_descontoPct` do Painel Mensal** (mesma origem = valor compartilhado; default 0). A MC de todos os gráficos e KPIs segue a fórmula do Painel Mensal — `MC = Vendas × (1 − % desconto) − Custo`. Como o payload traz `mc = vendas − custo` (sem desconto), a página aplica `mc_desc = mc − vendas × %desconto/100`. O desconto afeta **somente a MC**; venda histórica e ticket médio permanecem em valores reais (mesmo comportamento do Painel Mensal).
- **Definições (batem com as abas existentes):** a série mensal (venda histórica / MC / ticket) usa a **mesma base da aba Vendas Históricas** (vendas líquidas de devoluções no nível do pedido). Grupo de mercadoria e grupo de comissão usam a **mesma base da aba Vendas por Grupo de Comissão** (netagem por `QTD_DEVOLVIDA`/`QTD_ESTORNADA` na linha do item); grupo de mercadoria reaproveita a reclassificação de DIVERSOS/SEM GRUPO por palavra-chave do `atualizar_mapa.py`. Por usarem bases diferentes (nível pedido vs. nível item), os totais das duas visões não batem exatamente entre si — é a mesma inconsistência já documentada abaixo.
- O mês corrente é marcado como **parcial**.

## Inconsistências conhecidas (documentadas como fato, não como bug pendente)

Estas diferenças são reais no código atual e devem ser levadas em conta ao comparar números entre sub-abas — não são necessariamente erros a corrigir, mas comportamento que precisa ser conhecido:

1. **Aplicação do fator de desconto (`descontoPct`) é inconsistente por sub-aba:**
   - Painel Mensal / Diárias / Clientes: desconta a **receita**, depois subtrai custo (`venda*f - custo`).
   - Consistência por Grupo (coluna MC): desconta a **margem já calculada** (`(venda-custo)*f`).
   - Grupo de Produto (breakdown) e Venda por Fornecedor: **não aplicam** o desconto em nenhum momento.

2. **Líquido de devoluções (devoluções/estornos subtraídos da venda e custo) também varia por nível de agregação:**
   - Aplicado: Painel Mensal (vendedores), ranking/histórico de Clientes.
   - **Não aplicado**: Grupo de Produto, histórico mensal geral, histórico de comissão, Venda por Fornecedor.

3. Consequência prática: **não comparar diretamente** valores de venda/MC entre, por exemplo, "Grupo de Produto" e "Painel Mensal" para o mesmo período — as bases de cálculo diferem.
