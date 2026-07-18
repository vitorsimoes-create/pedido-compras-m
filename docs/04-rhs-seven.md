# RHS/SEVEN

Grupo de navegação separado da MC MOTO, cobrindo as unidades de negócio do grupo SEVEN. Fonte de dados principal: banco **`projeto_f7`** (espelho read-only do ERP da SEVEN, tabelas com prefixos `TCLI_`, `TVND_`, `TPED_`/`VPED_`, `TREC_`, `TPAG_`, `TMER_`, etc.). Credenciais não reproduzidas aqui — ver `conexaomc.md` (arquivo local, não versionado).

## Unidades de Negócio

Sub-aba padrão do grupo RHS/SEVEN. **Não é conteúdo próprio de `index.html`** — é o mesmo iframe `mapa-vendas.html` usado pela aba Vendas da MC MOTO, apenas trocado para sua sub-aba interna `unidades` (`trocarAba('unidades')`), agora mostrando dados agregados por unidade de negócio em vez de por vendedor/grupo de produto MC MOTO.

Mecânica compartilhada com o restante do Mapa de Vendas: mesmo cálculo de MC (`vendas*f - custo`), e uma marcação de **atraso de sincronização** — se a última nota fiscal de uma unidade estiver mais de **2 dias** (`LIMITE_ATRASO`) atrasada em relação à data de referência, a unidade recebe um badge âmbar de "atraso" na interface.

## CRM

Iframe separado, `crm-seven.html`, escopo declarado "Unidades 3 e 4". Painel próprio, independente do sistema de navegação por senha/categoria do restante do painel. Não detalhado nesta documentação (arquivo autocontido, fora do escopo revisado).

## Compras

Réplica da aba **MC MOTO → Compras** (mesmo fluxo, mesmas regras de interface), mas alimentada pelo banco **`projeto_f7`** da SEVEN, unidades **3, 4 e 5**. Conteúdo nativo de `index.html` (`app-tab-comprasseven`), com 3 sub-abas: Montar Pedido, Cotação, Pedidos Salvos. **Recebimentos ficou de fora por decisão explícita** — não existe fonte confiável de notas de entrada no espelho da SEVEN (a candidata `TMOV_EXTRA` filtrada por fornecedor tem só ~1 registro por unidade em ~3 anos).

### Fonte de dados: `RAW_DATA_SEVEN`

Array JS embutido em `index.html` (~21.800 itens), gerado pelo script local `gerar_raw_data_seven.py` (não versionado; **atualização manual** — não está em tarefa agendada). Um item por **produto + unidade** (o mesmo código de produto pode aparecer nas 3 unidades com estoque/custo próprios; a chave composta usada no JS é `codigo_unidade`).

| Campo | Significado | Origem |
|---|---|---|
| `c` | Código do produto | `TMER_ESTOQUE.TMER_CODIGO_PRI_FK_PK` |
| `u` | Unidade de negócio (3, 4 ou 5) | `TMER_ESTOQUE.TMER_UNIDADE_FK_PK` |
| `d` | Descrição | `TMER_MERCADORIA.TMER_NOME` |
| `p` | Custo médio | `TMER_ESTOQUE.TMER_CUSTO_MEDIO` |
| `k` | Pico de vendas mensal (máximo por mês nos últimos 12m, **nunca soma** — mesma regra da MC MOTO) | `vendas_para_ponto_de_pedido_12m` |
| `e` | Estoque atual | `TMER_ESTOQUE.TMER_ESTOQUE_ATUAL` |
| `s` | Sugestão de compra — **sempre 0** (não existe esse conceito nos dados da SEVEN; quantidade default ao adicionar é 1) | — |
| `cf` | Código original/fabricante | `TMER_MERCADORIA.TMER_CODIGO_ORIGINAL` |
| `f` | Fornecedor principal (fantasia, ou razão social) | `TFOR_FORNECEDOR` via `TMER_FORNECEDOR_PRINCIPAL_FK` |
| `g` | Grupo de produto — **vem classificado do próprio banco** (`TMER_GRUPO_MERCADORIA`), sem reclassificação por palavra-chave como na MC MOTO | mapeamento fixo código→nome |

Filtros aplicados na geração: só itens com `TMER_ATIVO_COMPRA='S'` e fornecedor principal ≠ 9999 (cadastro "DEMONSTRACAO"). Grupos `001`–`004` ("ERRO") caem em `DIVERSOS`.

### Diferenças em relação ao Compras da MC MOTO

1. **Filtro de Unidade** (checkboxes 3/4/5, todas marcadas por padrão) — não existe na MC MOTO.
2. **Sem badge "Comprar?"/regra de comissão** — o banco da SEVEN não tem o campo de comissão que alimenta essa regra na MC MOTO; a coluna foi removida por decisão explícita.
3. **Sem sugestão de compra** — o filtro padrão é "Todos os itens" (não "Com sugestão"), e a quantidade default ao adicionar é sempre 1.
4. Armazenamento separado em `localStorage`: `seven_pedidos_salvos` e `seven_cotacoes` (não se misturam com os da MC MOTO).
5. O modal de exportação é **compartilhado** entre MC MOTO e SEVEN — a variável `_pedidoAtivoFonte` decide título, builders de CSV/PDF (com coluna Unidade, sem Sugestão/Comprar?) e qual pedido o botão "Salvar" grava.
6. Na cotação, o preço respondido pelo fornecedor é **por código de produto** (não por unidade) — se o mesmo produto estiver na cotação para duas unidades, o preço importado vale para as duas linhas.

Todo o resto (busca E/OU, multi-seleção de fornecedor, "adicionar todos" com confirmação acima de 200 itens, exportações texto/CSV/PDF, fluxo completo de cotação com link HTML/planilha e "comprar pelo menor preço") segue as mesmas regras documentadas em [01-compras](01-compras.md).

## Financeiro

Painel nativo de `index.html` (não iframe), com 3 sub-categorias via `switchCategoriaFinanceiroSeven`:

### Contas a Pagar e Contas a Receber

Duas páginas geradas pelo script local `gerar_contas_seven.py` (não versionado; **atualização manual**, não agendada), embutidas via iframe: `contas-pagar-seven.html` e `contas-receber-seven.html`. Ambas cobrem as **unidades 3, 4 e 5** e compartilham a mesma estrutura:

- **Dois modos**, alternados por botões: **"Em aberto"** e **"Quitadas (últimos 12 meses)"**.
- **Filtro de unidade de negócio** (checkboxes 3/4/5, todas marcadas por padrão) — todos os KPIs, gráfico e detalhe são recalculados no navegador conforme a seleção.
- **Aging dos títulos em aberto** (mesma regra do Contas a Pagar do Mapa de Vendas): **vencida** (`vencimento < hoje`), **até 30 dias** (`vencimento < hoje+30`), **futura** (demais). O aging é recalculado contra a data em que a página está sendo vista — a classificação "envelhece" sozinha entre gerações; só os saldos exigem reexecução do script.
- KPIs (Total em aberto / Vencidas / Vencem em 30 dias / Futuras) + gráfico de barras por mês de vencimento, com bucket "Após 12m" para qualquer vencimento além de 12 meses (o que também absorve datas digitadas erradas no ERP, ex. um título com vencimento no ano 2502).
- Fontes e regras específicas:
  - **Pagar em aberto**: `TPAG_ABERTO` com `TPAG_SALDO_TITULO > 0` (saldo líquido de pagamentos parciais — situações `AB` e `PP`). Detalhe: lista de títulos agrupada por mês de vencimento (accordion), mês vencido/atual auto-expandidos, cabeçalho vermelho para mês com título vencido.
  - **Pagar quitadas (12m)**: `TPAG_BAIXADO`, situação `LQ`, pagamento (`TPAG_DATA_ULTIMO_PAGAMENTO`) nos últimos 12 meses. ⚠️ **Essa tabela-espelho tem duplicação massiva (~66x)** — verificado em 18/07/2026: 103.675 linhas correspondiam a apenas 1.575 títulos reais. A geração **deduplica obrigatoriamente** por chave natural (unidade, tipo, número, parcela, fornecedor). Detalhe: tabela agregada por fornecedor (títulos pagos, último pagamento, total pago), ordenada por total.
  - **Receber em aberto**: `TREC_ABERTO` com `TREC_RECEBER_PAGAR_FK='R'` e `TREC_SALDO_TITULO > 0`. Detalhe: tabela agregada **por cliente** (títulos, vencido, maior atraso em dias, vence em 30d, futuro, total), ordenada pelo maior valor vencido, linha vermelha para quem tem saldo vencido.
  - **Receber quitadas (12m)**: `TREC_BAIXADO`, situação `LQ`, baixa nos últimos 12 meses (essa tabela **não** tem a duplicação da TPAG — verificado; ainda assim a geração deduplica por segurança). Detalhe: tabela agregada por cliente (títulos recebidos, último recebimento, total recebido).

Nota: **MC MOTO → Financeiro → Contas a Pagar** continua existindo em separado (deep-link para dentro do Mapa de Vendas, banco `mc_moto`) — são painéis distintos com fontes distintas.

### Risco/Cliente
Embutido via iframe (`risco-cliente.html`), gerado diariamente pelo script `gerar_risco_cliente.py`. Ver metodologia completa abaixo.

## Metodologia — Risco de Inadimplência por Cliente

Objetivo: identificar, toda semana, clientes cujo saldo devedor (títulos a receber em aberto) está crescendo rápido demais dia a dia, como sinal de possível inadimplência.

**Escopo**: unidades de negócio 3, 4 e 5 (constante `UNIDADES = (3, 4, 5)`). O relatório publicado tem um **filtro de unidade de negócio** (checkboxes "Unidade 3" / "Unidade 4" / "Unidade 5", todas marcadas por padrão) que recalcula tudo no navegador — saldo diário combinado, crescimento médio, flag de risco e KPIs — considerando apenas as unidades selecionadas naquele momento. O script Python busca e envia ao HTML o saldo diário **separado por unidade** para cada cliente (mais a média de compra semanal por unidade); a agregação/soma das unidades escolhidas e todo o recálculo do critério de risco acontecem 100% em JavaScript, sem nova consulta ao banco.

### Passo 1 — janela da semana atual
```python
segunda = hoje - timedelta(days=hoje.weekday())   # segunda-feira da semana corrente
dias = [segunda, segunda+1, ..., hoje]             # inclusive, de segunda até hoje
```

### Passo 2 — reconstrução do saldo devedor dia a dia, por unidade
Para cada cliente **e cada unidade** (3, 4, 5 separadamente), junta dois conjuntos de títulos a receber (`TREC_RECEBER_PAGAR_FK = 'R'`):
- **`TREC_ABERTO`**: todos os títulos ainda em aberto hoje (sem data de baixa). Usa `TREC_SALDO_TITULO` (saldo já líquido de eventuais pagamentos parciais em títulos com situação `PP`), não `TREC_VALOR_TITULO` — usar o valor cheio nesse caso supervaloriza o saldo de títulos parcialmente pagos.
- **`TREC_BAIXADO`**: títulos já pagos, mas apenas os baixados nos últimos ~95 dias antes da segunda-feira da semana analisada (janela de performance — títulos pagos há mais tempo não poderiam estar em aberto em nenhum dia da semana atual de qualquer forma).

Saldo de um cliente/unidade em uma data de referência `d`:
```python
saldo_em(titulos, d) = soma do valor de cada título onde:
    data_emissao <= d  E  (nunca_foi_pago OU data_baixa > d)
```
Ou seja: título conta como "em aberto naquele dia" se já tinha sido emitido e, ou nunca foi pago, ou só foi pago **depois** daquela data (estava aberto naquele momento, mesmo que hoje já esteja quitado). O script grava, para cada cliente, um array de saldo diário por unidade (`unidades: {"3": [...], "4": [...], "5": [...]}`) — é esse array bruto que o HTML publicado usa para recalcular tudo conforme o filtro de unidade escolhido.

### Passo 3 — crescimento diário (recalculado no navegador conforme o filtro)
No JavaScript do relatório, para as unidades atualmente marcadas no filtro, soma-se o saldo diário de cada unidade selecionada em um único array de "saldo combinado" do cliente, dia a dia. Depois, para cada par de dias consecutivos da semana:
```
base = saldoCombinado(dia anterior)
if base > 50.00:   # SALDO_MINIMO_BASE — ignora bases pequenas para não gerar % explosivos por ruído
    crescimento = (saldoCombinado(dia) - base) / base
```
`crescimentoMedio` = média aritmética simples de todos os crescimentos diários válidos da semana, **para a combinação de unidades atualmente selecionada**. Se nenhum dia teve base > R$50 (ou o cliente não tem nenhum título nas unidades selecionadas), o cliente **não tem `crescimentoMedio` válido e é excluído da tabela** — não aparece como 0% nem como não-risco, simplesmente não é listado enquanto aquele filtro estiver ativo. Trocar o filtro pode fazer clientes aparecerem/desaparecerem e o "risco" de um cliente mudar, já que a base de cálculo muda.

### Passo 4 — critério de risco
```python
risco = crescimentoMedio > 0.20   # estritamente maior que 20%, não "maior ou igual"
```

### Contexto adicional — média de compras semanais (não entra no critério de risco)
Para cada cliente **e cada unidade**, calcula a média de valor de pedidos por semana nas últimas 13 semanas (`YEARWEEK(..., modo ISO)`, excluindo pedidos cancelados), **dividido sempre por 13** (não pelo número de semanas com pedido de fato — clientes novos ou com poucas semanas ativas terão essa média artificialmente reduzida). Ao aplicar o filtro de unidade, o valor exibido é a soma da média das unidades selecionadas. Serve apenas como referência de porte do cliente na tabela, não afeta se ele é marcado como risco.

### Filtro de unidade de negócio
Checkboxes "Unidade 3 / 4 / 5" no topo do relatório, todas marcadas por padrão. Cada mudança de seleção dispara uma nova renderização completa (`renderizar()`): recombina os saldos diários das unidades marcadas por cliente, recalcula `crescimentoMedio`/`saldoAtual`/`mediaCompraSemanal`/`risco` para cada um, reordena e reconstrói KPIs + tabela — tudo client-side, sem nova consulta ao banco. Desmarcar todas as unidades mostra uma mensagem pedindo para selecionar ao menos uma, sem tentar renderizar uma tabela vazia.

### Saída publicada (`risco-cliente.html`)
- Cabeçalho com data/hora de geração, semana analisada e as unidades atualmente selecionadas no filtro.
- Caixa de metodologia (versão resumida, em português simples, visível a qualquer usuário do painel).
- Filtro de unidade de negócio (3/4/5).
- 3 KPIs: nº de clientes em risco, nº de clientes analisados, soma do saldo devedor dos clientes em risco — todos recalculados conforme o filtro ativo.
- Tabela com **todos** os clientes analisáveis da semana para o filtro ativo (risco e não-risco), ordenados por `crescimentoMedio` decrescente — clientes em risco destacados em vermelho com tag "⚠️ RISCO". Colunas: cliente, saldo devedor atual, crescimento médio diário, evolução do saldo dia a dia (segunda → hoje), média de compra semanal (13 semanas).
- O relatório só reflete a última execução da tarefa agendada (a busca no banco e a montagem dos dados brutos por unidade) — o filtro em si é interativo e roda inteiramente no navegador, sem consulta ao vivo.
