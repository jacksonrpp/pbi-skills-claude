# Skills Power BI para Claude Code

Conjunto de **4 skills** que transformam os agentes de Power BI (originalmente
"chat modes" do VS Code/Copilot) em skills do Claude Code. Três operam sobre o
**MCP de modelagem do Power BI** (`powerbi-modeling-mcp`); uma trabalha na camada
de relatório (arquivos + orientação). Elas se referenciam entre si para rotear
cada problema à camada certa, sem duplicação.

## Instalação

Copie as pastas para uma destas localizações:

- **Global (todos os projetos):** `~/.claude/skills/`
- **Por projeto:** `.claude/skills/` na raiz do repositório

Cada skill é uma pasta com `SKILL.md` + `references/`. Ex.:
`~/.claude/skills/power-bi-data-modeling/SKILL.md`.

As três skills de modelo/DAX/performance pressupõem o `powerbi-modeling-mcp`
conectado. A de report-design não usa o MCP.

---

## As quatro skills

### 1. power-bi-data-modeling — modelo de dados
- **Camada:** modelo semântico (Tabular). **Superfície:** MCP.
- **Dispara em:** esquema estrela, tabelas fato/dimensão, relações e
  cardinalidade, modos de armazenamento (Import/DirectQuery/Dual/Composite),
  partições, agregações, atualização incremental, RLS, SCD, dimensões
  role-playing, pontes muitos-para-muitos; reduzir tamanho do modelo.
- **Faz:** inspeciona o modelo ao vivo, propõe, valida DAX, aplica em transação e
  verifica (laço: conectar → inspecionar → propor → validar → aplicar → conferir).
- **Referência:** `references/patterns.md` — payloads de relação/medida/papel/
  partição, RLS estática+dinâmica, incremental, composite hot/cold, SCD Tipo 2,
  ponte, `INFO.VIEW.*`.

### 2. power-bi-dax — autoria e tuning de DAX
- **Camada:** medidas/cálculos. **Superfície:** MCP.
- **Dispara em:** escrever/revisar/corrigir/otimizar medidas, colunas/tabelas
  calculadas, time intelligence (YTD/YoY/MTD/média móvel), CALCULATE e contexto,
  transição de contexto, variáveis, grupos de cálculo, "por que essa medida está
  errada/lenta".
- **Faz:** valida o DAX (`Validate`) antes de criar, cria/atualiza a medida, prova
  a correção e mede performance real (`getExecutionMetrics` + `ClearCache`) em vez
  de só afirmar.
- **Referência:** `references/dax-patterns.md` — ciclo validar→criar→benchmark,
  variáveis & DIVIDE, YoY/YTD/média móvel, grupos de cálculo, percentil de
  clientes (corrigido para `PERCENTILEX.INC`), ranking, anti-padrões antes/depois.

### 3. power-bi-report-design — relatórios e visualização
- **Camada:** relatório/visual. **Superfície:** arquivos PBIR + tema (NÃO usa MCP).
- **Dispara em:** qual gráfico usar, layout de página, UX, acessibilidade, mobile,
  tooltips/drillthrough/filtragem cruzada, tema JSON, formatação condicional,
  páginas lentas/poluídas, editar arquivos de relatório num projeto `.pbip`/PBIR.
- **Faz:** orienta o design, autora o tema JSON (validado) e edita os JSON de
  visual/página do PBIR. É a **dona da performance de camada de relatório**.
- **Referência:** `references/theme-and-files.md` — tema JSON anotado, layout de
  arquivos PBIR, e (nicho) embedding com `powerbi-client`.

### 4. power-bi-performance — triagem e diagnóstico
- **Camada:** transversal (orquestradora). **Superfície:** MCP (trace/métricas) +
  consultivo (capacidade/monitoramento).
- **Dispara em:** algo "lento" sem causa clara — relatório/página lenta, timeout,
  atualização longa, uso alto de capacidade/memória, lentidão de DirectQuery.
- **Faz:** mede onde o tempo vai (trace de servidor: Storage Engine vs Formula
  Engine) e **encaminha** — fórmula→dax, scan/modelo→data-modeling, visuais→
  report-design; trata refresh, e orienta capacidade/gateway/KQL (fora do MCP).
- **Referência:** `references/diagnostics.md` — payloads de `trace_operations`,
  leitura SE×FE, diagnóstico de refresh, campos de estatísticas de evento,
  consultas KQL do Log Analytics.

---

## Como as skills se conectam (roteamento)

```
                    power-bi-performance
                   (mede o gargalo e roteia)
                    /        |         \
      Formula Engine    Storage Engine   muitos visuais /
      pesado            pesado / scan     página sem filtro
          |                  |                  |
   power-bi-dax   power-bi-data-modeling   power-bi-report-design
          \__________________|__________________/
                 (referências cruzadas entre si)
```

- **data-modeling → dax / report-design / performance** (seção "Entre skills").
- **dax → data-modeling** (modelo é a causa raiz frequente) **/ performance / report-design**.
- **report-design → dax / data-modeling / performance**.
- **performance** delega a todas as três com a evidência do trace.

## Convenções (todas as skills)

- **MCP-first com segurança:** leituras livres; **confirmação antes de qualquer
  escrita** (`Create`/`Update`/`Delete`/`Rename`/`Refresh`/`Deploy`); cuidado
  redobrado com `Delete`, `DeployToFabric`, `ClearCache` e `Refresh`.
- **Descoberta:** `operation: "Help"` em qualquer `*_operations` retorna os
  parâmetros exatos — usar em vez de adivinhar.
- **Doc da Microsoft:** usa o MCP do Microsoft Docs se conectado, senão
  `learn.microsoft.com`; se nada disponível, avisa que não verificou ao vivo.
- **Idioma:** prosa em pt-BR; identificadores (nomes de operação MCP, funções DAX,
  código, JSON, KQL) e o campo `name:` das skills permanecem em inglês.

## MCP: `powerbi-modeling-mcp` (referência das ferramentas)

Padrão de chamada: `{ "request": { "operation": ... } }`. Operações comuns:
`Help, List, Get, Create, Update, Delete, Rename, ExportTMDL`.

| Ferramenta | Usada por | Para quê |
|---|---|---|
| `connection_operations` | performance, todas | achar/selecionar a instância |
| `database_operations` | data-modeling | listar/exportar/importar TMDL/TMSL, DeployToFabric |
| `table_operations` / `column_operations` | data-modeling, dax | inspecionar tabelas/colunas/tipos |
| `relationship_operations` | data-modeling, dax | relações (List/Create/Activate…) |
| `measure_operations` | dax, data-modeling | criar/editar/mover medidas |
| `dax_query_operations` | dax, data-modeling, performance | Validate / Execute / ClearCache / métricas |
| `calculation_group_operations` | dax | itens de time intelligence |
| `security_role_operations` | data-modeling | papéis e RLS (CreatePermissions) |
| `partition_operations` | data-modeling, performance | partições / Refresh |
| `named_expression_operations` | data-modeling | parâmetros M (RangeStart/RangeEnd) |
| `calendar_operations` | data-modeling, dax | calendários lógicos (CL ≥ 1701) |
| `trace_operations` | performance | traces de servidor (SE×FE, DirectQuery) |

A `power-bi-report-design` não usa o MCP: opera em arquivos PBIR do `.pbip` e no
tema JSON.
