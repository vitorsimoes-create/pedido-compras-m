# MC MOTO → Recebimentos / Meta / CMV

Apesar de ser conteúdo financeiro, esta sub-aba (**📦 Recebimentos**) vive dentro da categoria **Compras** de `index.html` (não dentro da categoria "Financeiro" da MC MOTO, que só contém "Contas a Pagar" — ver [`02-vendas.md`](02-vendas.md)). É conteúdo **nativo** de `index.html`, não do iframe do Mapa de Vendas.

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
