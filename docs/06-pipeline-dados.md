# Pipeline de dados — como o painel é mantido atualizado

O painel é 100% estático: não há consulta a banco de dados feita pelo navegador. Todo dado exibido vem de um **snapshot gravado dentro do próprio HTML publicado**, atualizado por scripts Python locais rodando em tarefas agendadas na máquina do usuário.

## Tarefa agendada

**Regra estrutural (decisão de 20/07/2026, substituindo o esquema anterior de 2 rotinas por banco): existe exatamente 1 rotina diária, `atualizacao-diaria-painel` (~06:00), que roda os dois bancos juntos e publica num commit único.** Toda aba/sub-aba nova criada no painel deve ser incluída no escopo dessa rotina (editando o `SKILL.md` da tarefa em `C:\Users\Vitor\.claude\scheduled-tasks\atualizacao-diaria-painel\`) — nunca criar uma segunda rotina.

A rotina tem duas partes:

| Parte | Banco | O que atualiza |
|---|---|---|
| A — MC MOTO | `mc_moto` | `RAW_DATA` (campos `e`, `k`, `cf`, `f`, `nc`), `RECEBIMENTOS_DB`, `CMV_MENSAL` em `index.html`; roda `atualizar_mapa.py` para regenerar `Mapa de Vendas.html` e ressincroniza a cópia publicada `mapa-vendas.html`; ressincroniza `painel-caixa.html` a partir do snapshot local "painel_caixa feito.html" quando ele muda; roda `gerar_contas_receber_mcmoto.py` (→ `contas-receber-mcmoto.html`) e `gerar_vendas_historicas_mcmoto.py` (→ `vendas-historicas.html`) |
| B — SEVEN | `projeto_f7` (read-only) | Roda os 3 scripts da SEVEN: `gerar_risco_cliente.py` (→ `risco-cliente.html`), `gerar_raw_data_seven.py` (→ `RAW_DATA_SEVEN` em `index.html`) e `gerar_contas_seven.py` (→ `contas-pagar-seven.html` + `contas-receber-seven.html`) |

Nota histórica: a antiga Tarefa Agendada do Windows que rodava `atualizar_mapa.py` às 19h ficou redundante — desde 20/07/2026 o Mapa é gerado pela própria rotina diária.

Credenciais de conexão **não são reproduzidas aqui** — ficam em `conexaomc.md` (arquivo local, fora do controle de versão). Cada parte da rotina só acessa o seu próprio banco.

A rotina segue a regra geral do projeto: só commita os arquivos que realmente mudaram (`git add` por nome, nunca `-A`), nunca cria commit vazio, e faz `git push` automaticamente ao final.

## Fluxo 1 — Parte A da rotina diária (banco `mc_moto`)

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

## Fluxo 2 — Regeneração do Mapa de Vendas (`Mapa de Vendas.html`, dentro da Parte A)

Processo separado, local (Windows Task Scheduler), roda `atualizar_mapa.py`:
1. Conecta em `mc_moto` e `projeto_f7`.
2. Reconstrói **o arquivo inteiro** a partir de um template Python embutido (`HTML_TEMPLATE`), com substituição de tokens (`__PAYLOAD__`, `__GERADO_EM__`, `__DADOS_ATE__`) — não é edição incremental.
3. Escreve o resultado em `Mapa de Vendas.html` (original — nunca editado à mão, nunca commitado).
4. A sincronização para o arquivo publicado (`mapa-vendas.html`) acontece na sequência, dentro da própria rotina.

Qualquer mudança de conteúdo/regra de negócio do Mapa de Vendas (abas Vendas, Financeiro→Contas a Pagar, Unidades de Negócio) precisa ser feita editando o template dentro de `atualizar_mapa.py` e rerodando o script — nunca editando a saída diretamente.

## Fluxo 3 — Parte B da rotina diária (banco `projeto_f7`)

Uma única rotina roda, em sequência, os 3 scripts locais da SEVEN (todos conectam somente em `projeto_f7`, read-only; nenhum é versionado):

1. `gerar_risco_cliente.py` → sobrescreve `risco-cliente.html` (análise de risco de inadimplência; metodologia completa em [`04-rhs-seven.md`](04-rhs-seven.md)) e anexa uma linha em `atualizar_risco.log`.
2. `gerar_raw_data_seven.py` → substitui in-place a linha `const RAW_DATA_SEVEN = [...];` de `index.html` (~21.800 itens das unidades 3/4/5; pico máximo-por-mês, filtros de item ativo/fornecedor demo — ver [`04-rhs-seven.md`](04-rhs-seven.md)). Não toca no `RAW_DATA` da MC MOTO.
3. `gerar_contas_seven.py` → sobrescreve `contas-pagar-seven.html` e `contas-receber-seven.html` (em aberto com saldo líquido + quitadas 12 meses; **deduplicação obrigatória da `TPAG_BAIXADO`**; aging recalculado no navegador — ver [`04-rhs-seven.md`](04-rhs-seven.md)).

Depois: valida sintaxe JS dos 4 arquivos alterados, confere via `git status` que só eles mudaram, commit único ("Atualização diária automática SEVEN...") e push. Os mesmos scripts podem ser rodados manualmente a qualquer momento para um refresh fora de hora — os passos são idênticos.

## Observação geral sobre "atualidade" dos dados

Como não existe consulta ao vivo, qualquer número exibido no painel reflete apenas a última execução bem-sucedida da tarefa correspondente — não o estado atual do banco no momento em que o usuário está olhando a tela. Cada painel gerado por script carrega seu próprio timestamp de geração (`data-atualizacao` em `index.html`; "Gerado em" no Mapa de Vendas; "Gerado em" no relatório de Risco/Cliente) para permitir conferir a defasagem.
