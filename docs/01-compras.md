# MC MOTO → Compras

Todo o conteúdo desta aba é **nativo de `index.html`** (não passa pelo iframe do Mapa de Vendas). Tem 4 sub-abas: Montar Pedido, Cotação, Pedidos Salvos, Recebimentos (Recebimentos é documentado à parte em [`03-financeiro-mc-moto.md`](03-financeiro-mc-moto.md), por ser conteúdo financeiro).

## Fonte de dados: `RAW_DATA`

Array JS embutido em `index.html`, ~18.945 itens de catálogo, um objeto por produto:

| Campo | Significado | Atualizado pela tarefa diária? |
|---|---|---|
| `c` | Código interno do item (chave primária, inteiro) | — (identificador, não muda) |
| `d` | Descrição do item (texto, maiúsculo) | Não — estático |
| `r` | Código/referência "raw" legado (string numérica, ex. `"8027.0"`) — usado só na busca textual, nunca exibido | Não — estático |
| `p` | Preço/custo unitário (R$) | Não — estático |
| `k` | **Pico de vendas mensal líquido** — ver fórmula exata abaixo | **Sim**, diariamente |
| `e` | Estoque atual (`SALDO`, pode ser negativo) | **Sim**, diariamente |
| `s` | **Sugestão de compra** = `pico − estoque` (`k − e`) | **Sim** — gravada diariamente pela rotina e também recalculada ao vivo no navegador (ver nota abaixo) |
| `nc` | Flag de restrição de compra / faixa de comissão (0, 1 ou 2) — ver regra "Comprar?" abaixo | **Sim**, diariamente |
| `cf` | Código do fabricante (exibido na coluna "Cód. Fabricante") | **Sim**, diariamente |
| `f` | Nome do fornecedor | **Sim**, diariamente |
| `g` | Grupo de produto | Calculado no navegador ao carregar a página (não vem no JSON, é preenchido por `getGrupo(item.d)` — ver seção seguinte) |

### Nota sobre o campo `s` (sugestão de compra)

**Fórmula: `s = pico − estoque` (`k − e`).** É calculada em dois lugares que sempre concordam: (1) **no navegador ao carregar a página** (na mesma passada em que `item.g` é preenchido, logo após o `RAW_DATA`) — esse é o mecanismo que garante que nunca fica defasada; e (2) **gravada no próprio JSON pela rotina diária** (`atualizacao-diaria-painel`, Parte A), logo após atualizar `k` e `e`, só para manter o dado armazenado consistente com o exibido. Pode ser negativa quando o item está superestocado (estoque maior que o pico mensal) — nesse caso a interface mostra o badge de sugestão em vermelho.

Histórico do bug (20/07/2026): antes desta correção, `s` vinha **congelado** no JSON do `RAW_DATA` e **não era recalculado** pela rotina diária, enquanto `k` e `e` eram atualizados todo dia — então `s` ia ficando defasado (ex.: item 13262 mostrava sugestão 1 quando o correto, com pico 8 e estoque 2, era 6). O diagnóstico confirmou que 87,7% dos itens ainda batiam com `k − e` exatamente (provando que essa sempre foi a fórmula) e que o restante estava apenas desatualizado. A correção passou a calcular `s` ao vivo a partir de `k`/`e`, então ele **nunca mais fica defasado** e o valor congelado do JSON é irrelevante (é sobrescrito no load).

## Classificação automática de grupo de produto (`getGrupo`)

Executada uma vez no carregamento da página, para cada item de `RAW_DATA`:

```js
RAW_DATA.forEach(item => { item.g = getGrupo(item.d); });
```

`getGrupo(desc)` percorre uma lista ordenada de regras (`GRUPO_RULES`), cada uma com um grupo-alvo e uma lista de palavras-chave; a **primeira regra cuja palavra-chave aparece na descrição** (via `startsWith`, ou como palavra separada por espaço, ou substring) decide o grupo do item. A ordem das regras importa (primeira correspondência vence).

Grupos existentes: `MOTOR`, `ELETRICA`, `FREIO`, `PLASTICO`, `METAL`, `ACESSORIOS`, `BOUTIQUE`, `CABOS`, `SUSPENSAO`, `PNEUMATICO`, `PARTES`, `QUIMICOS`, `MIUDEZAS`, e o grupo catch-all **`DIVERSOS`** para qualquer descrição que não bata com nenhuma regra.

## Pico de vendas mensal líquido (campo `k`)

Regra de negócio (histórico: já houve um bug corrigido em produção que somava as quantidades de vários meses em vez de pegar o pico de um único mês — cuidado ao reimplementar isso):

> `k` = a **maior** quantidade líquida vendida em **um único mês**, entre os últimos 12 meses. Nunca a soma de vários meses.

Cálculo (SQL + agregação Python, rodado pela tarefa agendada `atualizacao-diaria-painel` (Parte A) — ver [`06-pipeline-dados.md`](06-pipeline-dados.md) para a consulta exata):

1. Agrupa vendas por produto e por mês (`YYYY-MM`), somando `QUANTIDADE - QTD_DEVOLVIDA - QTD_ESTORNADA` (quantidade líquida de devoluções/estornos) dos últimos 12 meses, excluindo itens cancelados.
2. Para cada produto, `k` = o **maior** valor entre as linhas mensais desse produto (substituição, nunca soma cumulativa).
3. Produto sem nenhuma venda no período → `k = 0`.

## Regra "Comprar?" (campo `nc`)

Badge de recomendação de compra exibido em toda a interface (tabela de itens, exportações), com base no valor de `nc`:

| `nc` | Rótulo | Cor | Significado |
|---|---|---|---|
| `0` | ✅ OK | Verde | Sem restrição |
| `1` | ❌ Não comprar | Vermelho | Comissão à vista de 8% — item não deve ser comprado |
| `2` | ⚠️ A avaliar | Âmbar | Comissão à vista de 5% — avaliar antes de comprar |

Esta regra é aplicada de forma idêntica em pelo menos 6 pontos do código (tabela principal, exportação texto, CSV, PDF, aba de pedido em janela separada) — qualquer alteração na regra precisa ser replicada em todos esses pontos.

## Fluxo "Montar Pedido"

### Filtros disponíveis
- **Fornecedor** — multi-seleção (ver seção dedicada abaixo).
- **Grupo** — dropdown de grupo de produto único.
- **Busca livre** — texto, com modo **E** (AND — todas as palavras precisam aparecer) ou **OU** (OR — qualquer palavra aparecendo já inclui o item, e neste modo os resultados são reordenados por relevância = número de termos encontrados, ignorando a ordenação escolhida).
- **"Mostrar apenas"** — três opções: `sugestao` (padrão; só itens com `s > 0`), `todos` (sem filtro), `sem_estoque` (só itens com `e ≤ 0`).
- **Ordenação** — 12 opções (por sugestão, descrição, custo, estoque, pico, código — cada uma em ordem crescente/decrescente).

### Regras visuais (badges/cores)
- **Estoque**: negativo → vermelho; zero → neutro; positivo → verde.
- **Sugestão**: `s > 0` → azul; `s < 0` → vermelho; `s = 0` → cinza/neutro (mostra "—").
- Item sem sugestão (`s ≤ 0`) e ainda não adicionado ao pedido ganha um botão "+ manual" (âmbar) em vez do botão azul padrão.

### Adicionar item ao pedido
Regra de quantidade padrão ao adicionar um item (usada tanto em "Montar Pedido" quanto em "Cotação"):

```
quantidade_default = item.s > 0 ? item.s : 1
```

Ou seja: usa a sugestão de compra se ela for positiva; caso contrário, começa em 1 (ajuste manual). A quantidade nunca pode cair abaixo de 1 depois de adicionada (`Math.max(1, qty)` aplicado em toda edição).

"Adicionar todos" (adiciona todo o resultado filtrado de uma vez) pede confirmação extra se o filtro atual tiver mais de **200 itens**.

### Painel do pedido e totais
Desde 22/07/2026, o pedido de compra da MC MOTO é apresentado como **lista única de produtos** (não é mais separado em blocos por fornecedor) — decisão do usuário: "não quero múltiplos fornecedores, apenas os produtos". Mantém-se uma coluna **Fornecedor** por linha para referência, e há apenas o **total geral** (sem subtotais por fornecedor). Isso vale nas três superfícies: o modal `exportarPedido`, a janela separada `abrirPedidoAba` e os arquivos exportados (CSV/PDF via `buildCSV`/`buildPDFHtml`). Itens adicionados manualmente (`s ≤ 0`) recebem destaque visual (badge "MANUAL").

### Exportações
- **Texto** (modal com tabela formatada para copiar/colar).
- **CSV** — separado por `;`, com BOM UTF-8 (compatibilidade Excel pt-BR); colunas `Código;Descrição;Fornecedor;Grupo;Cód. Fabricante;…`; código de fabricante é envolvido em `="código"` para preservar zeros à esquerda; itens manuais recebem sufixo `[MANUAL]` na descrição.
- **PDF** — HTML A4 paisagem (tabela única com coluna Fornecedor), impresso via `window.print()`.
- **Aba separada (janela nova, `abrirPedidoAba`)** — abre o pedido em uma página independente, tabela única com campos de quantidade/sugestão editáveis ao vivo e recálculo automático do total geral. Além de **💾 Salvar Pedido** e **🖨️ Imprimir**, a janela tem botões **⬇ Excel (.csv)** e **🖨 PDF** (iguais aos da RHS/SEVEN): eles primeiro sincronizam as edições de qtd/sugestão de volta para `window.opener.pedido` e então chamam `exportarExcelDireto()`/`exportarPDFDireto()`.

## Pedidos Salvos

Armazenamento: **`localStorage` do navegador**, chave `mc_moto_pedidos_salvos` — **não é persistência de servidor**, os dados vivem só naquele perfil de navegador.

Estrutura de um registro salvo:
```js
{
  id: 'ped_' + Date.now(),
  nome,                 // prompt() do usuário, com default "Pedido {data}"
  pedido: {...},        // cópia profunda do pedido de trabalho
  total,
  nItens,
  nForn,                // número de fornecedores distintos
  criadoEm, atualizadoEm  // ISO datetime
}
```

- Salvar exige pelo menos 1 item no pedido de trabalho.
- Se o usuário estiver editando um pedido salvo já existente (fluxo "✏️ Editar"), a gravação **atualiza in-place** em vez de criar um novo registro; novos salvamentos são inseridos no início da lista (mais recente primeiro).
- Busca por nome, nome de fornecedor dentro do pedido, ou data de criação.
- Ações disponíveis por pedido salvo: Editar (recarrega no painel de trabalho), exportar (mesmas opções de Montar Pedido), Excluir (com confirmação).

## Cotação

Fluxo paralelo/independente ao Montar Pedido: permite selecionar itens, escolher fornecedores para cotar, coletar preços de cada um, e escolher automaticamente o mais barato por item.

Armazenamento: `localStorage`, chave `mc_moto_cotacoes`.

### Seleção
- Checkbox "Cotar" por item na tabela, independente do botão "+" do pedido — estado em `cotacaoSelecionados` (por código).
- Quantidade default ao selecionar: mesma regra do pedido (`s > 0 ? s : 1`).
- Seleção de fornecedores a cotar é uma lista separada (`fornecedoresCotacaoSelecionados`) da usada no filtro da tabela — permite também adicionar um fornecedor **ad-hoc** que não está em `RAW_DATA` (marcado como "(novo)" na interface).

### Criar cotação
Exige ≥1 item selecionado **e** ≥1 fornecedor selecionado. Registro salvo:
```js
{
  id: 'cot_' + Date.now(),
  nome, criadoEm,
  itens: [{c, d, cf, g, p, qty}, ...],   // snapshot dos itens no momento da criação
  fornecedores: [...],
  respostas: {}   // preenchido depois: { nomeFornecedor: { codigo: preco } }
}
```

### Coletar preços dos fornecedores — dois mecanismos
1. **Planilha CSV (ida e volta)** — baixa um CSV por fornecedor (`Código;Descrição;Cód. Fabricante;Qtd;Preço Unit. (R$)`, preço em branco) para o fornecedor preencher; a importação de volta faz parsing tolerante a cabeçalho/comentários, converte números no formato pt-BR (`,` decimal), e só aceita preços `> 0`.
2. **Link HTML autônomo** — gera um arquivo HTML independente (formulário de preços) pensado para ser publicado no GitHub Pages, em uma URL prevista dentro da pasta `cotacoes/` do repositório. **Esse passo de publicação é manual** (o próprio texto do alerta na interface pede que seja feito manualmente) — não há automação de deploy desse arquivo. Quando o fornecedor preenche e salva, gera um JSON de resposta (`resposta_{cotId}_{fornecedor}.json`) que é importado de volta na ferramenta.

### Escolher o mais barato (`comprarMaisBarato`)
Regra de negócio: **para cada item da cotação, entre todos os fornecedores que responderam com um preço para aquele item, escolhe o menor preço.** O item é então inserido no pedido de trabalho (`pedido`) com o fornecedor e o preço vencedores. Itens sem nenhuma resposta são ignorados (não entram no pedido). Esta é a única lógica de "melhor preço" existente em todo o painel.

### Listagem de cotações criadas
Mostra, por fornecedor dentro de cada cotação, quantas respostas de preço já foram recebidas (contagem de chaves em `respostas[fornecedor]`), com indicação visual "✅ N preço(s) recebido(s)" ou "⏳ aguardando resposta". Exclusão de cotação é permanente (com confirmação).
