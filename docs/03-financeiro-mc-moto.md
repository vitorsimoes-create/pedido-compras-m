# MC MOTO → Financeiro (e Recebimentos/Meta/CMV)

A categoria **Financeiro** da MC MOTO tem duas sub-abas: **Contas a Pagar** (deep-link para dentro do Mapa de Vendas — documentada em [`02-vendas.md`](02-vendas.md); inclui o Painel de Caixa embutido) e **Contas a Receber** (documentada abaixo). Já a sub-aba **📦 Recebimentos**, apesar de ser conteúdo financeiro, vive dentro da categoria **Compras** de `index.html` — é a segunda metade deste documento.

## Contas a Receber (banco `mc_moto`)

Página `contas-receber-mcmoto.html`, gerada pelo script local `gerar_contas_receber_mcmoto.py` (não versionado; roda diariamente na rotina `atualizacao-diaria-painel`, Parte A) e embutida via iframe na aba nativa `app-tab-contasrecebermc`. Fonte: tabela `contas_receber` do ERP (sem duplicação — verificado; é o banco direto, não espelho).

- **Dois modos**: **"Em aberto"** (situação `A`, saldo = `VALOR - VALOR_RECEBIDO > 0`) e **"Recebidas (últimos 12 meses)"** (situação `Q`, com `DATA_PAGAMENTO` na janela; valor considerado = `VALOR_RECEBIDO`, ou `VALOR` quando `VALOR_RECEBIDO` está zerado).
- **Aging dos títulos em aberto** com a mesma regra padrão do projeto (vencida / ≤30 dias / futura, recalculada no navegador contra a data atual) + KPIs + gráfico por mês de vencimento.
- Detalhe do modo aberto: tabela de títulos (cliente, documento/NF, emissão, vencimento, situação, saldo) — o volume é naturalmente pequeno, pois a venda da MC MOTO é majoritariamente à vista.
- Detalhe do modo recebidas: **agrupado por mês** (um accordion por mês, mais recente primeiro, mês corrente aberto por padrão); dentro de cada mês, os clientes aparecem ordenados pelo maior valor recebido, com total do mês no cabeçalho. Os dados vão **pré-agregados por (mês, cliente)** no gerador (payload `pagosMesCliente`) — diferente das páginas da SEVEN, não há filtro de unidade aqui, então não é preciso embutir título a título; a página fica leve.
- Nota de leitura: o cliente `000001` ("CLIENTE CONSUMIDOR") concentra as vendas de balcão e por isso domina o topo do ranking de recebidas.

# Recebimentos / Meta / CMV (dentro da categoria Compras)

## Fontes de dados

### `RECEBIMENTOS_DB` (sistema, somente leitura)
Array embutido em `index.html`, alimentado por notas de entrada reais (NFs de fornecedores), regenerado diariamente pela tarefa agendada. Cada registro:
```js
{ id: "db_...", fornecedor, data: "YYYY-MM-DD", valor, obs: "NF numero/serie", origem: "sistema" }
```
Registros de origem `sistema` (prefixo `id` = `db_`) **não podem ser excluídos** pela interface — só os manuais podem.

### Registros manuais
Guardados em `localStorage`, chave `mc_moto_recebimentos`, id `rec_{timestamp}`, origem `"manual"`. Adicionados/removidos livremente pelo usuário via formulário na própria aba.

`getRecebimentos()` combina as duas fontes em uma lista só para renderização.

### Exclusão de fornecedores "não mercadoria"
Lista de exclusão em `localStorage`, chave `mc_moto_forn_nao_mercadoria` — permite marcar fornecedores que não representam compra de mercadoria (ex.: contabilidade, serviços, transportadoras) para que **todos os totais da aba** (mês atual e quadro anual) ignorem as notas desses fornecedores. Gerenciado por um modal dedicado ("⚙️ Fornecedores de mercadoria").

### `CMV_MENSAL`
Objeto estático embutido em `index.html`, `{"YYYY-MM": valor, ...}` — Custo da Mercadoria Vendida por mês, também regenerado pela tarefa diária (cobre os últimos ~25 meses). Ver a consulta SQL exata em [`06-pipeline-dados.md`](06-pipeline-dados.md).

## Meta (objetivo de compras) — fórmula automática

```js
function mesAnterior(mes) { /* YYYY-MM do mês anterior a `mes` */ }
function metaAutoCMV(mes) { return CMV_MENSAL[mesAnterior(mes)] || 0; }
function metaEfetiva(mes) {
  if (existe override manual para `mes`) return { valor: override, auto: false };
  return { valor: metaAutoCMV(mes), auto: true };
}
```

**Regra: a meta padrão de compras de um mês = o CMV do mês imediatamente anterior.** Uma meta manual (definida via prompt, pré-preenchido com a sugestão automática como referência) sempre tem prioridade sobre o cálculo automático, e permanece válida até ser explicitamente apagada (campo em branco no prompt remove o override e volta ao cálculo automático). Override manual fica em `localStorage`, chave `mc_moto_metas_mensais`.

Na interface, valores automáticos aparecem com sufixo `" (CMV)"` em cinza; valores manuais aparecem sem sufixo, em cor normal.

## Regras de cor / threshold

### Card do mês selecionado
- `desvio = total_recebido - meta`
- Classe do card: `desvio > 0` (recebeu mais que a meta) → **âmbar**; caso contrário → **verde**.
  *(Nota de leitura: aqui "receber mais que a meta" é tratado como alerta — porque esta aba mede volume de **compras/entradas**, não vendas; passar da meta de compra pode ser indesejado.)*
- Barra de progresso: `pct = round(total/meta*100)`, largura limitada a 100%. Cor: **vermelho** se `pct > 100`; **âmbar** se `pct > 85`; **azul** caso contrário.

### Quadro anual (`renderQuadroAnual`)
Para cada um dos 12 meses do ano selecionado (opções 2024–2027):
- `recebidoMes` = soma filtrada (excluindo fornecedores não-mercadoria) daquele mês.
- `meta = metaEfetiva(mes).valor`.
- `desvio = meta > 0 ? recebidoMes - meta : null`.
- Linha esmaecida (`opacity: 0.4`) se o mês for **futuro** em relação ao mês atual.
- Cor do desvio: `null` → cinza; `> 0` → âmbar; `< 0` → verde.
- Linha de total do ano: soma `recebidoMes` de todos os 12 meses, mas soma `meta` **apenas dos meses com meta > 0**, com o mesmo esquema de cor no desvio total.
