# RHS/SEVEN

Grupo de navegação separado da MC MOTO, cobrindo as unidades de negócio do grupo SEVEN. Fonte de dados principal: banco **`projeto_f7`** (espelho read-only do ERP da SEVEN, tabelas com prefixos `TCLI_`, `TVND_`, `TPED_`/`VPED_`, `TREC_`, `TPAG_`, `TMER_`, etc.). Credenciais não reproduzidas aqui — ver `conexaomc.md` (arquivo local, não versionado).

## Unidades de Negócio

Sub-aba padrão do grupo RHS/SEVEN. **Não é conteúdo próprio de `index.html`** — é o mesmo iframe `mapa-vendas.html` usado pela aba Vendas da MC MOTO, apenas trocado para sua sub-aba interna `unidades` (`trocarAba('unidades')`), agora mostrando dados agregados por unidade de negócio em vez de por vendedor/grupo de produto MC MOTO.

Mecânica compartilhada com o restante do Mapa de Vendas: mesmo cálculo de MC (`vendas*f - custo`), e uma marcação de **atraso de sincronização** — se a última nota fiscal de uma unidade estiver mais de **2 dias** (`LIMITE_ATRASO`) atrasada em relação à data de referência, a unidade recebe um badge âmbar de "atraso" na interface.

## CRM

Iframe separado, `crm-seven.html`, escopo declarado "Unidades 3 e 4". Painel próprio, independente do sistema de navegação por senha/categoria do restante do painel. Não detalhado nesta documentação (arquivo autocontido, fora do escopo revisado).

## Financeiro

Painel nativo de `index.html` (não iframe), com 3 sub-categorias via `switchCategoriaFinanceiroSeven`:

### Contas a Pagar
⚠️ **Este rótulo existe em dois lugares diferentes do painel, com implementações diferentes:**
- **RHS/SEVEN → Financeiro → Contas a Pagar**: hoje é apenas um placeholder **"Em breve."** — não implementado.
- A funcionalidade de Contas a Pagar **real e funcional** do projeto vive em outro lugar: **MC MOTO → Financeiro → Contas a Pagar**, que é o deep-link para dentro do Mapa de Vendas (`mapa-vendas.html`). Essa versão:
  - Traz títulos em aberto (`SITUACAO='A'`) a partir de uma data-base fixa configurada no gerador.
  - Classifica cada título por vencimento: **vencida** (`vencimento < hoje`), **até 30 dias** (`vencimento < hoje+30`), ou **futura** (demais casos).
  - Mostra cards de KPI (Total em aberto, Vencidas, Vencem em 30 dias, Futuras >30 dias) e um gráfico de barras por mês de vencimento (agrupando qualquer coisa além de 12 meses em um bucket "Após 12m").
  - Lista detalhada agrupada/recolhível por mês de vencimento, com o mês atual e qualquer mês vencido auto-expandidos, e cabeçalho vermelho para meses vencidos.

### Contas a Receber
Placeholder **"Em breve."** — não implementado.

### Risco/Cliente
Único painel funcional dessa seção, embutido via iframe (`risco-cliente.html`), gerado diariamente pelo script `gerar_risco_cliente.py`. Ver metodologia completa abaixo.

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
- **`TREC_ABERTO`**: todos os títulos ainda em aberto hoje (sem data de baixa).
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
