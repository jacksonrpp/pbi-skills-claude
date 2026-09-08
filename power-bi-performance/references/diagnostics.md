# Performance Power BI — Diagnósticos (MCP + monitoramento)

Payloads e referência para a `power-bi-performance`. A captura de trace/métrica usa
o `powerbi-modeling-mcp`; capacidade e monitoramento de tenant ficam fora dele (KQL
abaixo).

Toda ferramenta MCP recebe `{ "request": { "operation": ... } }`; chame
`operation: "Help"` para os campos exatos. Traces são leitura, mas `ClearCache` e
`Refresh` afetam a instância em execução — confirme antes de usar em algo
compartilhado.

## Conteúdo

1. Capturar um trace de servidor (Start → rodar → Fetch → Stop)
2. Lendo o trace: Storage Engine vs Formula Engine
3. Frio rápido antes/depois com métricas de execução
4. Diagnóstico de atualização
5. Referência de campos de estatísticas de evento
6. Consultas do Log Analytics (KQL)

---

## 1. Capturar um trace de servidor

```json
// Escolha a instância (para tracear o modelo certo)
{ "request": { "operation": "ListLocalInstances" } }          // connection_operations
```

```json
// Comece a capturar eventos de query + VertiPaq SE + DirectQuery + métricas
{ "request": { "operation": "Start",
  "events": ["QueryBegin","QueryEnd",
             "VertiPaqSEQueryBegin","VertiPaqSEQueryEnd","VertiPaqSEQueryCacheMatch",
             "DirectQueryBegin","DirectQueryEnd","ExecutionMetrics","Error"],
  "filterCurrentSessionOnly": true } }                        // trace_operations
```

Agora rode a consulta lenta (abaixo), depois puxe os eventos com colunas úteis:

```json
{ "request": { "operation": "Fetch",
  "columns": ["EventClassName","EventSubclassName","Duration","CpuTime","TextData","StartTime","EndTime"] } }
```

```json
// Opcional: persista a captura completa para análise mais profunda
{ "request": { "operation": "ExportJSON", "filePath": "/tmp/pbi-trace.json", "clearAfterFetch": true } }
```

```json
// Sempre pare/limpe ao terminar — um trace rodando consome recursos
{ "request": { "operation": "Stop" } }
```

Rode a consulta sendo diagnosticada (limpe o cache primeiro para leitura fria — ver
§3):

```json
{ "request": { "operation": "Execute", "query": "<a consulta DAX lenta>",
  "getExecutionMetrics": true } }                              // dax_query_operations
```

---

## 2. Lendo o trace: Storage Engine vs Formula Engine

A partir dos eventos buscados:

- **Tempo SE** = soma do `Duration` de `VertiPaqSEQueryEnd`. É o tempo escaneando
  colunas comprimidas.
- **Total** = o `Duration` de `QueryEnd` da consulta inteira.
- **Tempo FE** ≈ Total − SE. É avaliação single-thread, iteração e materialização.

Interpretação → roteamento:

| Assinatura | Causa provável | Encaminhar para |
|---|---|---|
| SE alto, poucas consultas SE | Scan grande / alta cardinalidade / agregação ausente / modo de armazenamento | **power-bi-data-modeling** |
| FE alto, muitas consultas SE pequenas | Fórmula cara / materialização ruim / transição de contexto | **power-bi-dax** |
| `DirectQueryEnd` domina | SQL / indexação da fonte | Otimize a fonte ou a estratégia de armazenamento |
| Muitos `QueryEnd` por interação | Consultas de visual demais por clique | **power-bi-report-design** |
| `VertiPaqSEQueryCacheMatch` baixo em repetições | Filtros voláteis / padrão não cacheável | DAX ou modelo, conforme acima |

Uma consulta saudável é majoritariamente SE com um punhado de consultas SE. "Muitas
consultas SE pequenas alimentando um FE lento" é a impressão digital clássica de
DAX ruim.

---

## 3. Frio rápido antes/depois com métricas de execução

Para um A/B rápido sem um trace completo:

```json
{ "request": { "operation": "ClearCache" } }                  // dax_query_operations — ver cuidado
{ "request": { "operation": "Execute",
  "query": "EVALUATE <consulta>",
  "getExecutionMetrics": true, "executionMetricsOnly": true } }
```

`executionMetricsOnly: true` lê todas as linhas para tempo preciso mas omite o
payload de linhas. Rode para a versão antiga e a nova e compare. (Reescritas de
fórmula pertencem a **power-bi-dax**; isto é só como medi-las.)

> Cuidado: `ClearCache` degrada as próximas consultas para todos os usuários da
> instância. Só em Desktop/dev, ou com confirmação explícita em algo compartilhado.

---

## 4. Diagnóstico de atualização

```json
// Atualize UMA partição sob um trace, em vez de uma atualização completa
{ "request": { "operation": "Refresh", "refreshDefinitions": [
  { "tableName": "FactSales", "name": "Incremental", "refreshType": "Full" } ] } }  // partition_operations
```

Leia as estatísticas de atualização/comando (ver §5) para duração, paralelismo,
pico de memória, linhas. Roteamento: folding quebrado ou colunas largas/de alta
cardinalidade → **modeling**; muitas partições atualizando em série → política
incremental → **modeling**; fonte lenta → o sistema de origem. Agende atualizações
pesadas fora de pico em capacidade compartilhada.

---

## 5. Referência de campos de estatísticas de evento

Campos que você verá em eventos `ExecutionMetrics` / de comando (via o trace ou
`getExecutionMetrics`), e o que dizem:

```jsonc
// Evento de query
{
  "durationMs": 69143,                  // tempo de relógio da consulta
  "totalCpuTimeMs": 63,                 // CPU entre threads (SE é multi-thread)
  "queryProcessingCpuTimeMs": 16,
  "directQueryConnectionTimeMs": 3,
  "directQueryTotalTimeMs": 121872,     // alto => a fonte é o gargalo
  "approximatePeakMemConsumptionKB": 3632,
  "queryResultRows": 67,
  "directQueryRequestCount": 2
}

// Evento de comando de refresh
{
  "durationMs": 1274559,
  "mEngineCpuTimeMs": 9617484,          // tempo no motor Power Query (M)
  "totalCpuTimeMs": 9618469,
  "approximatePeakMemConsumptionKB": 1683409,   // observe vs. memória da capacidade
  "refreshParallelism": 16,             // baixo aqui + muitas partições => serializado
  "vertipaqTotalRows": 114
}
```

`directQueryTotalTimeMs` alto em relação a `durationMs` → banco de origem.
`mEngineCpuTimeMs` alto → transformações do Power Query (folding / passos pesados).
Pico de memória perto da capacidade → pressão de memória, reduza o modelo ou
escale.

---

## 6. Log Analytics (KQL) — tendências & análise no tenant

Encaminhe os logs de dataset para o Azure Log Analytics, depois consulte
`PowerBIDatasetsWorkspace`. (Fora do MCP — esta skill ajuda a escrever/interpretar
isto.)

```kusto
// Volume diário de log, últimos 30 dias
PowerBIDatasetsWorkspace
| where TimeGenerated > ago(30d)
| summarize count() by format_datetime(TimeGenerated, 'yyyy-MM-dd')
```

```kusto
// Duração média de query por dia
PowerBIDatasetsWorkspace
| where TimeGenerated > ago(30d)
| where OperationName == 'QueryEnd'
| summarize avg(DurationMs) by format_datetime(TimeGenerated, 'yyyy-MM-dd')
```

```kusto
// Percentis de duração (p50/p90) por hora — achar picos
PowerBIDatasetsWorkspace
| where TimeGenerated > ago(7d)
| where OperationName == 'QueryEnd'
| summarize percentiles(DurationMs, 50, 90) by bin(TimeGenerated, 1h)
```

```kusto
// Carga por workspace: consultas, usuários distintos, CPU médio & duração
PowerBIDatasetsWorkspace
| where TimeGenerated > ago(30d)
| where OperationName == "QueryEnd"
| summarize QueryCount = count(),
            Users      = dcount(ExecutingUser),
            AvgCPU     = avg(CpuTimeMs),
            AvgDuration = avg(DurationMs)
  by PowerBIWorkspaceId
| order by AvgDuration desc
```

Use estas para achar *qual* dataset/workspace/hora está lento, depois aprofunde
naquele modelo com um trace de servidor (§1–§2).
