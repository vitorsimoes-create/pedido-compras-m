# Referência rápida — fórmulas, thresholds e glossário

Tabela de consulta rápida com todas as regras numéricas do painel. Para o contexto completo de cada regra, ver o documento da área correspondente.

## Tabela de fórmulas e thresholds

| Regra | Fórmula / threshold | Onde vive | Documentado em |
|---|---|---|---|
| Quantidade padrão ao adicionar item ao pedido/cotação | `item.s > 0 ? item.s : 1` | `index.html`, `toggleItem`/`adicionarTodos`/`toggleCotacaoItem` | [01-compras](01-compras.md) |
| Filtro "com sugestão de compra" | `item.s > 0` | `index.html`, `filterData` | [01-compras](01-compras.md) |
| Filtro "sem estoque" | `item.e <= 0` | `index.html`, `filterData` | [01-compras](01-compras.md) |
| Cor do estoque | `e<0`→vermelho, `e=0`→neutro, `e>0`→verde | `index.html` | [01-compras](01-compras.md) |
| Cor da sugestão | `s>0`→azul, `s<0`→vermelho, `s=0`→cinza | `index.html` | [01-compras](01-compras.md) |
| Regra "Comprar?" | `nc=1`→Não comprar (comissão 8%), `nc=2`→A avaliar (comissão 5%), `nc=0`→OK | `index.html`, `comprarTexto()` (repetida em 6 lugares) | [01-compras](01-compras.md) |
| Confirmação de adição em massa | `itens_filtrados.length > 200` | `index.html`, `adicionarTodos` | [01-compras](01-compras.md) |
| Piso de quantidade | `Math.max(1, qty)` sempre | `index.html` | [01-compras](01-compras.md) |
| Cotação: melhor preço | menor preço entre respostas de fornecedores, por item | `index.html`, `comprarMaisBarato` | [01-compras](01-compras.md) |
| Pico de vendas mensal (`k`) | máximo (nunca soma) da quantidade líquida vendida por mês, últimos 12 meses | tarefa agendada + `RAW_DATA` | [01-compras](01-compras.md), [06-pipeline-dados](06-pipeline-dados.md) |
| MC (Painel Mensal / Diárias / Clientes) | `vendas*(1-desconto%) - custo` | `atualizar_mapa.py` | [02-vendas](02-vendas.md) |
| MC (Consistência por Grupo) | `(venda-custo) * (1-desconto%)` — nota: desconta a margem já pronta, não a receita | `atualizar_mapa.py` | [02-vendas](02-vendas.md) |
| Projeção do mês | `realizado / diasUteisDecorridos * diasUteisTotal` | `atualizar_mapa.py` | [02-vendas](02-vendas.md) |
| Dias úteis decorridos | conta até **ontem**, exclui domingos e feriados cadastrados | `atualizar_mapa.py` | [02-vendas](02-vendas.md) |
| % da meta (Painel Mensal) | `realizado/meta`, `≥100%`→verde, senão vermelho | `atualizar_mapa.py` | [02-vendas](02-vendas.md) |
| Consistência — filtro "Tudo" | exclui mês corrente, cobre todos os grupos | `atualizar_mapa.py` (Python) | [02-vendas](02-vendas.md) |
| Consistência — filtro 3/6/12 meses | inclui mês corrente parcial, só top-7 grupos + "Outros" | `atualizar_mapa.py` (JS) | [02-vendas](02-vendas.md) |
| Banner de alinhamento | overlap de participação: `≥90%`→bem condizente, `≥80%`→razoável, `<80%`→divergente | `atualizar_mapa.py` | [02-vendas](02-vendas.md) |
| Situação (projeção vs. média) | banda de ±10%: acima / dentro / abaixo | `atualizar_mapa.py` | [02-vendas](02-vendas.md) |
| Margem baixa/alta por grupo | ±10 pontos percentuais da MC% média ponderada geral | `atualizar_mapa.py` | [02-vendas](02-vendas.md) |
| Estoque parado (Venda por Fornecedor) | `estoque>0 AND (cobertura=null OR cobertura>6 meses)` | `atualizar_mapa.py` | [02-vendas](02-vendas.md) |
| "Sem giro" | há estoque, zero CMV no período (`cobertura=null`) | `atualizar_mapa.py` | [02-vendas](02-vendas.md) |
| Margem negativa (Clientes) | `pctMc < 0` → vermelho (sem banda de tolerância aqui) | `atualizar_mapa.py` | [02-vendas](02-vendas.md) |
| Meta automática de compras (Recebimentos) | `meta[mês] = CMV_MENSAL[mês anterior]`, override manual tem prioridade | `index.html` | [03-financeiro-mc-moto](03-financeiro-mc-moto.md) |
| Barra de progresso (Recebimentos) | `pct>100`→vermelho, `pct>85`→âmbar, senão azul | `index.html` | [03-financeiro-mc-moto](03-financeiro-mc-moto.md) |
| Atraso de sincronização (Unidades RHS/SEVEN) | última NF mais de 2 dias atrasada | `atualizar_mapa.py` | [04-rhs-seven](04-rhs-seven.md) |
| Aging de Contas a Pagar | vencida (`<hoje`) / até 30 dias / futura | `atualizar_mapa.py` | [04-rhs-seven](04-rhs-seven.md) |
| Risco de inadimplência — base mínima | ignora dias com saldo-base ≤ R$50 no cálculo de crescimento | `gerar_risco_cliente.py` | [04-rhs-seven](04-rhs-seven.md) |
| Risco de inadimplência — limiar | crescimento médio diário da semana **> 20%** | `gerar_risco_cliente.py` | [04-rhs-seven](04-rhs-seven.md) |
| Média de compra semanal (contexto) | soma de 13 semanas ÷ 13 (fixo, não ÷ semanas ativas) | `gerar_risco_cliente.py` | [04-rhs-seven](04-rhs-seven.md) |

## Glossário

- **MC (Margem de Contribuição)**: venda menos custo da mercadoria, em R$. Fórmula exata varia por sub-aba — ver tabela acima.
- **MC%**: MC dividido pela venda, em percentual.
- **CMV**: Custo da Mercadoria Vendida — soma do custo dos itens efetivamente vendidos em um período.
- **Pico de vendas**: maior quantidade vendida em um único mês (não soma de meses), dentro da janela dos últimos 12 meses.
- **Sugestão de compra (`s`)**: quantidade sugerida para reposição de um item. **Origem atual desconhecida** — valor estático não recalculado pela rotina diária (ver [01-compras](01-compras.md)).
- **Saldo devedor**: soma dos títulos a receber em aberto de um cliente em uma data de referência (reconstruído a partir de emissão/baixa de títulos, não de um snapshot diário real).
- **Cobertura (de estoque)**: quantos meses o estoque atual dura, no ritmo médio de CMV mensal (`estoque_a_custo / cmv_mensal`).
- **Giro**: quantas vezes o estoque "virou" no período (`cmv_12m / estoque`).
- **Fator de desconto (`descontoPct`)**: ajuste manual global aplicado às vendas do Mapa de Vendas para simular descontos concedidos; aplicado de forma inconsistente entre sub-abas (ver [02-vendas](02-vendas.md), seção "Inconsistências conhecidas").
- **Dias úteis**: segunda a sábado, excluindo feriados cadastrados manualmente; domingo nunca conta.
- **Fornecedor de mercadoria**: fornecedor cujas notas de entrada representam compra de produto para revenda (por oposição a serviços/despesas), usado para filtrar os totais de Recebimentos.
