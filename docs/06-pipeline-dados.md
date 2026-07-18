# Pipeline de dados — como o painel é mantido atualizado

O painel é 100% estático: não há consulta a banco de dados feita pelo navegador. Todo dado exibido vem de um **snapshot gravado dentro do próprio HTML publicado**, atualizado por scripts Python locais rodando em tarefas agendadas na máquina do usuário.

## Tarefas agendadas

| Tarefa (`taskId`) | Frequência | O que atualiza | Banco usado |
|---|---|---|---|
| `atualizar-painel-pedidos-diario` | Diária, ~06:08 | `RAW_DATA` (campos `e`, `k`, `cf`, `f`, `nc`), `RECEBIMENTOS_DB`, `CMV_MENSAL` em `index.html`; também ressincroniza `mapa-vendas.html` a partir de `Mapa de Vendas.html` | `mc_moto` |
| `analise-risco-inadimplencia-diaria` | Diária, ~06:26 | `risco-cliente.html` (roda `gerar_risco_cliente.py`) | `projeto_f7` (read-only) |
| *(fora do escopo desta doc)* `atualizar_mapa.py` | Diária, via Windows Task Scheduler local | `Mapa de Vendas.html` (o arquivo original) | `mc_moto` + `projeto_f7` |

Credenciais de conexão de cada banco **não são reproduzidas aqui** — cada tarefa está restrita, por definição, a um único banco (nunca deve acessar o outro), e as credenciais reais ficam em `conexaomc.md` (arquivo local, fora do controle de versão).

Cada tarefa segue a regra geral do projeto: só commita os arquivos que realmente mudaram, nunca cria commit vazio, e faz `git push` automaticamente ao final — sem pedir confirmação, por serem tarefas autônomas agendadas.

## Fluxo 1 — Atualização diária do painel de pedidos (`index.html`)

1. Conecta no banco `mc_moto`.
2. Busca e recalcula 3 conjuntos de dados (consultas validadas, reproduzidas abaixo por serem a referência "canônica" — já houve um bug de produção corrigido nessa lógica, então qualquer reimplementação deve seguir exatamente este padrão):

### a) Pico de vendas mensal líquido (campo `k`)
```sql
SELECT PRODUTO, DATE_FORMAT(DATA, '%Y-%m') AS ym,
       SUM(QUANTIDADE - COALESCE(QTD_DEVOLVIDA,0) - COALESCE(QTD_ESTORNADA,0)) AS qtd
FROM itens
WHERE (CANCELADO IS NULL OR CANCELADO <> 'S')
  AND DATA >= DATE_SUB(CURDATE(), INTERVAL 12 MONTH)
GROUP BY PRODUTO, ym
```
Uma linha por produto por mês. Em Python, para cada produto: `k = max(qtd entre as linhas desse produto)` — **substituição pelo maior valor, nunca soma cumulativa** (`+=` é o erro que já causou bug em produção). Produto sem nenhuma linha → `k = 0`.

### b) Notas de entrada (`RECEBIMENTOS_DB`)
```sql
SELECT FORNECEDOR, NUMERO, SERIE, DATA, TOTAL
FROM notas_entrada
WHERE DATA >= DATE_SUB(CURDATE(), INTERVAL 12 MONTH)
ORDER BY DATA DESC
```
Junta com `fornecedores` para obter o nome. Cada registro:
```
id: 'db_' + FORNECEDOR + '_' + NUMERO + '_' + SERIE + '_' + DATA(YYYY-MM-DD)
fornecedor: NOME
data: DATA (YYYY-MM-DD)
valor: round(TOTAL, 2)
obs: 'NF ' + NUMERO + '/' + SERIE
origem: 'sistema'
```

### c) CMV mensal (`CMV_MENSAL`)
```sql
SELECT DATE_FORMAT(DATA, '%Y-%m') AS ym,
       SUM(CUSTO * (QUANTIDADE - COALESCE(QTD_DEVOLVIDA,0) - COALESCE(QTD_ESTORNADA,0))) AS cmv
FROM itens
WHERE (CANCELADO IS NULL OR CANCELADO <> 'S')
  AND DATA >= DATE_SUB(CURDATE(), INTERVAL 25 MONTH)
GROUP BY ym
ORDER BY ym
```
Arredondado a 2 casas decimais, montado como `{"YYYY-MM": valor, ...}`.

3. Um script Python temporário (prefixo `_`, apagado após uso) lê `index.html`, localiza `RAW_DATA`/`RECEBIMENTOS_DB`/`CMV_MENSAL` via regex, faz `json.loads`/atualiza/`json.dumps(separators=(',',':'))` de volta, e atualiza a data em `<strong id="data-atualizacao">`.
   - Regra explícita: `nc` deve **sempre** estar presente em cada item, mesmo quando `0` — nunca omitir a chave.
   - Checagem de sanidade obrigatória antes de publicar: conferir 3 produtos aleatórios com `k > 0` para confirmar que o valor bate com uma única linha do agrupamento mensal, não com uma soma; parar e revisar se `k` parecer anormalmente alto (ex. mais de 3x a mediana dos outros meses do mesmo produto).
   - Validação de sintaxe JS do resultado antes de publicar.
4. **Sincronização obrigatória do Mapa de Vendas** (passo adicional desta mesma tarefa, porque o processo local que gera `Mapa de Vendas.html` não atualiza sozinho a cópia publicada):
   a. Copia `Mapa de Vendas.html` → `mapa-vendas.html` (sobrescrevendo).
   b. Na cópia, troca a regra CSS `.tabs { display:flex; ... }` por `.tabs { display:none !important; }` (evita barra de abas nativa duplicada, já que a navegação é controlada pelo `index.html` externamente).
   c. Valida sintaxe JS da cópia.
5. `git status` para confirmar que **apenas** `index.html` e/ou `mapa-vendas.html` mudaram.
6. Commit e push por arquivo alterado (mensagens padrão: "Atualização diária automática de estoque, vendas, fornecedores, recebimentos e CMV" para `index.html`; "Atualização diária automática do Mapa de Vendas" para `mapa-vendas.html`).

## Fluxo 2 — Regeneração do Mapa de Vendas (`Mapa de Vendas.html`)

Processo separado, local (Windows Task Scheduler), roda `atualizar_mapa.py`:
1. Conecta em `mc_moto` e `projeto_f7`.
2. Reconstrói **o arquivo inteiro** a partir de um template Python embutido (`HTML_TEMPLATE`), com substituição de tokens (`__PAYLOAD__`, `__GERADO_EM__`, `__DADOS_ATE__`) — não é edição incremental.
3. Escreve o resultado em `Mapa de Vendas.html` (original — nunca editado à mão, nunca commitado).
4. A sincronização para o arquivo publicado (`mapa-vendas.html`) acontece no Fluxo 1 (tarefa `atualizar-painel-pedidos-diario`), não aqui.

Qualquer mudança de conteúdo/regra de negócio do Mapa de Vendas (abas Vendas, Financeiro→Contas a Pagar, Unidades de Negócio) precisa ser feita editando o template dentro de `atualizar_mapa.py` e rerodando o script — nunca editando a saída diretamente.

## Fluxo 3 — Risco de Inadimplência (`risco-cliente.html`)

1. Roda `gerar_risco_cliente.py` (script já pronto, local, não versionado), que conecta somente em `projeto_f7` (read-only) e sobrescreve `risco-cliente.html` do zero, além de anexar uma linha em `atualizar_risco.log`.
2. Metodologia completa da análise: ver [`04-rhs-seven.md`](04-rhs-seven.md).
3. Validação de sintaxe JS do HTML resultante.
4. `git status` para confirmar que **apenas** `risco-cliente.html` mudou (o script `.py` em si nunca é versionado).
5. Commit ("Atualização diária automática da análise de risco de inadimplência") e push, se houve mudança de conteúdo.

## Fluxo 4 — Compras SEVEN (`RAW_DATA_SEVEN` em `index.html`) — manual

1. Roda-se `gerar_raw_data_seven.py` (script local, não versionado) **manualmente, sob demanda** — este fluxo **não está em nenhuma tarefa agendada**; os dados da aba RHS/SEVEN → Compras ficam congelados até a próxima execução manual.
2. O script conecta somente em `projeto_f7` (read-only), monta os ~21.800 itens (unidades 3, 4 e 5) e substitui in-place a linha `const RAW_DATA_SEVEN = [...];` de `index.html`. Regras de geração (pico máximo-por-mês, filtros de item ativo/fornecedor demo): ver [`04-rhs-seven.md`](04-rhs-seven.md).
3. Depois: validar sintaxe JS do `index.html`, `git add index.html`, commit e push.

## Observação geral sobre "atualidade" dos dados

Como não existe consulta ao vivo, qualquer número exibido no painel reflete apenas a última execução bem-sucedida da tarefa correspondente — não o estado atual do banco no momento em que o usuário está olhando a tela. Cada painel gerado por script carrega seu próprio timestamp de geração (`data-atualizacao` em `index.html`; "Gerado em" no Mapa de Vendas; "Gerado em" no relatório de Risco/Cliente) para permitir conferir a defasagem.
