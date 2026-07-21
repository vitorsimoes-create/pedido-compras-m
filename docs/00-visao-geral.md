# Visão Geral do Painel MC MOTO / RHS-SEVEN

## O que é

Painel interno de gestão para a **MC MOTO** (varejo de peças/acessórios para motos) e para o grupo **RHS/SEVEN** (unidades de negócio relacionadas). É uma aplicação **estática** (HTML + CSS + JS puro, sem backend/servidor próprio), publicada via **GitHub Pages** a partir do repositório [`vitorsimoes-create/pedido-compras-m`](https://github.com/vitorsimoes-create/pedido-compras-m), branch `main`.

Não há API nem banco de dados acessado em tempo real pelo navegador. Todos os dados que o painel exibe são **snapshots gerados periodicamente** por scripts Python locais (rodando na máquina do usuário via tarefas agendadas) e depois "gravados" diretamente dentro dos arquivos HTML publicados. Ver [`06-pipeline-dados.md`](06-pipeline-dados.md) para o fluxo completo de atualização.

## Arquitetura de arquivos

| Arquivo | Papel | Como é gerado/atualizado |
|---|---|---|
| `index.html` | App principal — todo o shell de navegação, controle de acesso, e as abas de **Compras** (Montar Pedido, Cotação, Pedidos Salvos, Recebimentos) | Editado manualmente (features) + reescrita diária automática dos dados (`RAW_DATA`, `RECEBIMENTOS_DB`, `CMV_MENSAL`) por tarefa agendada |
| `mapa-vendas.html` | Iframe embutido em `index.html`, cobre as abas de **Vendas**, **Financeiro → Contas a Pagar** (MC MOTO) e **Unidades de Negócio** (RHS/SEVEN) | É uma **cópia sincronizada** de `Mapa de Vendas.html` (ver abaixo) — nunca editado diretamente |
| `Mapa de Vendas.html` | Arquivo **original**, fonte de verdade do conteúdo acima | Gerado do zero a cada execução do script local `atualizar_mapa.py` (agendado no Windows Task Scheduler). **Nunca deve ser editado à mão** — qualquer mudança de conteúdo tem que ir no template Python e o script tem que ser rerodado |
| `crm-seven.html` | Iframe embutido, aba RHS/SEVEN → CRM | Arquivo separado, não coberto em detalhe nesta documentação |
| `risco-cliente.html` | Iframe embutido, aba RHS/SEVEN → Financeiro → Risco/Cliente | Gerado diariamente pelo script `gerar_risco_cliente.py` |

Regra crítica de manutenção: **`Mapa de Vendas.html` (com espaço no nome) nunca é commitado nem editado diretamente.** Toda mudança de conteúdo do Mapa de Vendas precisa: (1) alterar o template dentro de `atualizar_mapa.py`, (2) rerodar o script para regenerar `Mapa de Vendas.html`, (3) copiar o resultado para `mapa-vendas.html` (nome sem espaço, esse sim publicado), reaplicando o ajuste de CSS `.tabs{display:none!important}` (a barra de abas nativa do arquivo é escondida porque a navegação é controlada de fora, pelo `index.html`), (4) commitar apenas `mapa-vendas.html`.

## Estrutura de navegação

Há dois níveis principais de agrupamento, mais sub-níveis dentro de cada um.

### Nível 1 — Grupo (`switchGrupo`, em `index.html`)

Dois grupos:
- **🏍️ MC MOTO** (`grupobtn-mcmoto`, padrão ao abrir o painel)
- **🏢 RHS/SEVEN** (`grupobtn-rhsseven`)

Cada clique passa primeiro pelo gate de senha (ver seção abaixo) antes de trocar de conteúdo.

### Nível 2 — Categorias / sub-abas

**Dentro de MC MOTO** (`switchCategoriaMcMoto`, pills em `#categorias-mcmoto`):
- 🛍️ **Compras** (`compras`, categoria padrão) → abre a aba nativa "Montar Pedido" dentro do próprio `index.html`, com sub-abas internas: Montar Pedido, Cotação, Pedidos Salvos, Recebimentos.
- 📈 **Vendas** (`vendas`) → as 5 primeiras sub-abas são deep-links para dentro do iframe `mapa-vendas.html` (`abrirSubAbaMapa`): Painel Mensal, Vendas Diárias, Grupo de Produto, Consistência por Grupo, Venda por Fornecedor. As últimas três são abas nativas de `index.html` com iframe próprio: **Vendas Históricas** (`vendas-historicas.html`, `abrirSubAbaVendasHist`), **Vendas por Grupo de Comissão** (`vendas-comissao.html`, `abrirSubAbaVendasComissao`) e **Gráficos** (`graficos-mcmoto.html`, `abrirSubAbaGraficos` — painel consolidado de gráficos SVG dos últimos 12/24 meses).
- 💰 **Financeiro** (`financeiro`) → duas sub-abas: **Contas a Pagar** (deep-link para dentro do iframe `mapa-vendas.html`, que inclui também o Painel de Caixa embutido via `painel-caixa.html`) e **Contas a Receber** (aba nativa de `index.html` com iframe próprio, `contas-receber-mcmoto.html`, aberta por `abrirSubAbaFinanceiroMc`).

**Dentro de RHS/SEVEN** (`abrirSubAbaRHSSeven`, botões em `#subtabs-rhsseven`):
- 🏢 **Unidades de Negócio** (`unidades`, padrão) → **também** é um deep-link para dentro do mesmo iframe `mapa-vendas.html` (chama `trocarAba('unidades')`) — ou seja, é o mesmo arquivo/iframe usado pela aba Vendas da MC MOTO, só que mostrando dados agregados por unidade de negócio (banco `projeto_f7`).
- 👥 **CRM** (`crm`) → mostra o iframe separado `crm-seven.html`.
- 💰 **Financeiro** (`financeiroseven`) → mostra um painel nativo de `index.html` com 3 pills (`switchCategoriaFinanceiroSeven`): Contas a Pagar (placeholder "Em breve"), Contas a Receber (placeholder "Em breve"), Risco/Cliente (iframe `risco-cliente.html`, funcional).

Se o iframe `mapa-vendas.html` ainda não terminou de carregar quando o usuário clica em uma sub-aba que depende dele, o `index.html` guarda o alvo pendente em `_mapaTabPendente` e dispara a troca assim que o evento `load` do iframe ocorre.

## Controle de acesso (gate de senha)

O painel tem um overlay de bloqueio em tela cheia (`#lock-overlay`) exibido ao carregar, exigindo uma senha antes de mostrar qualquer conteúdo. O esquema:

- Existe uma senha própria para o grupo **MC MOTO** e outra para o grupo **RHS/SEVEN**, mais uma **senha de administrador** que libera os dois grupos de uma vez. As três estão definidas em texto simples dentro do JavaScript de `index.html`, nas constantes `TAB_PASSWORDS` (mapa `{mcmoto: ..., rhsseven: ...}`) e `ADMIN_PASSWORD`. **As senhas em si não são reproduzidas nesta documentação** — consulte o código-fonte diretamente se precisar delas.
- O desbloqueio é lembrado apenas durante a sessão do navegador, via `sessionStorage` (chaves `aba_desbloqueada_mcmoto` e `aba_desbloqueada_rhsseven`, valor `'1'`). Fechar a aba/sessão do navegador exige senha de novo.
- Ao trocar de grupo (`switchGrupo`), se aquele grupo ainda não foi desbloqueado nesta sessão, aparece um `prompt()` nativo do navegador pedindo a senha (aceita tanto a senha específica do grupo quanto a senha de admin).
- **Isto não é segurança real.** É um "deterrent" client-side — qualquer pessoa com acesso ao código-fonte (Ctrl+U no navegador) vê as senhas em texto simples. Serve apenas para evitar acesso casual/acidental, não para proteger dados sensíveis de fato.

## Regras de manutenção que já regem este projeto

Estas regras vieram de decisões explícitas ao longo do desenvolvimento e devem ser respeitadas por qualquer pessoa (ou IA) que mexer no projeto:

1. **Nunca editar `Mapa de Vendas.html` diretamente.** É saída de `atualizar_mapa.py`; qualquer mudança de conteúdo vai no template Python.
2. **Nunca commitar `Mapa de Vendas.html`** (o original, com espaço no nome) nem arquivos sensíveis da pasta (ex.: `conexaomc.md`, que guarda credenciais de banco, planilhas locais). Apenas `mapa-vendas.html` (cópia sincronizada, sem espaço) é versionado.
3. **`git add` sempre por nome de arquivo explícito**, nunca `git add -A` ou `git add .` — a pasta de trabalho tem arquivos sensíveis não rastreados que não podem ser commitados por acidente.
4. **Scripts Python de exploração pontual** devem ter prefixo `_` (ex.: `_teste_algo.py`) e ser apagados depois de usados; scripts que rodam de forma recorrente/agendada (ex.: `atualizar_mapa.py`, `gerar_risco_cliente.py`) ficam sem prefixo, mas **permanecem apenas locais** — não são commitados no git.
5. **Acesso a bancos de dados é restrito ao escopo já autorizado.** Qualquer nova fonte de dados exige credenciais fornecidas explicitamente pelo usuário — nunca assumir acesso a um banco/tabela novo.
6. Após qualquer atualização do painel publicado (`index.html` e arquivos relacionados), o padrão é **sempre fazer `git push` automaticamente**, sem precisar perguntar.
