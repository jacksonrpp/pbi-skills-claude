# DAX Power BI — Padrões & Receitas (MCP-first)

DAX correto e pronto para copiar, mais os payloads do `powerbi-modeling-mcp` que o
validam, criam e medem (benchmark). Leia a seção de que precisa.

Toda ferramenta é `<nome>_operations` e recebe `{ "request": { "operation": ... } }`.
Chame `operation: "Help"` em qualquer ferramenta para ver os campos exatos.
Escritas aceitam `options.useTransaction` (padrão `true`) e a maioria das operações
aceita listas.

## Conteúdo

1. Ciclo de vida da medida: validar → criar → benchmark
2. Variáveis & DIVIDE
3. Time intelligence (YoY, YTD, média móvel)
4. Grupos de cálculo (time intelligence escalável)
5. Segmentação por percentil de clientes (corrigida)
6. Ranking (RANKX)
7. Anti-padrões: antes → depois

---

## 1. Ciclo de vida da medida: validar → criar → benchmark

```json
// Valide se a fórmula compila antes de escrever qualquer coisa
{ "request": { "operation": "Validate",
  "query": "EVALUATE ROW(\"v\", DIVIDE([Profit], [Sales]))" } }   // dax_query_operations
```

```json
// Crie a medida quando validar
{ "request": { "operation": "Create", "definitions": [ {
  "name": "Profit Margin",
  "tableName": "Sales",
  "expression": "DIVIDE([Profit], [Sales])",
  "formatString": "0.0%",
  "displayFolder": "Profitability"
} ] } }                                                          // measure_operations
```

```json
// Benchmark: rode com server timings, sem payload de linhas, cache frio
{ "request": { "operation": "ClearCache" } }
{ "request": { "operation": "Execute",
  "query": "EVALUATE SUMMARIZECOLUMNS('Date'[Year], \"M\", [Profit Margin])",
  "getExecutionMetrics": true, "executionMetricsOnly": true } }  // dax_query_operations
```

Rode a consulta de métricas para a fórmula antiga e a nova e compare os tempos —
reporte o delta real em vez de afirmar "isto é mais rápido".

---

## 2. Variáveis & DIVIDE

```dax
// Compute o valor do ano anterior uma vez, reutilize, divida com segurança
Sales YoY % =
VAR CurrentSales  = [Total Sales]
VAR PriorSales    = CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Date'[Date] ) )
RETURN
    DIVIDE ( CurrentSales - PriorSales, PriorSales )
```

Um `VAR` é avaliado no contexto onde é **definido**, não onde é usado — útil para
capturar um valor antes de `CALCULATE` mudar o contexto.

---

## 3. Time intelligence (precisa de uma tabela de datas marcada)

```dax
YTD Sales = CALCULATE ( [Total Sales], DATESYTD ( 'Date'[Date] ) )
```

```dax
Sales PY = CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Date'[Date] ) )
```

```dax
// Média móvel de 3 meses das vendas mensais (grão: mês)
Sales 3M Avg =
VAR Period = DATESINPERIOD ( 'Date'[Date], MAX ( 'Date'[Date] ), -3, MONTH )
RETURN
    CALCULATE ( AVERAGEX ( VALUES ( 'Date'[Year Month] ), [Total Sales] ), Period )
```

Prefira essas embutidas a fazer contas de data na mão quando o calendário é padrão.
Para calendários fiscal / 4-4-5 / dia útil, use faixas `DATESBETWEEN` explícitas ou
calendários lógicos (`calendar_operations`).

---

## 4. Grupos de cálculo (aplicar time intelligence a toda medida)

Em vez de escrever cópias YTD/YoY/PY de cada medida, defina um grupo de cálculo
cujos itens transformam `SELECTEDMEASURE()`. Crie-o com
`calculation_group_operations` (chame `Help` para os nomes exatos dos campos); as
expressões dos itens ficam assim:

```dax
-- Item: "YTD"
CALCULATE ( SELECTEDMEASURE (), DATESYTD ( 'Date'[Date] ) )

-- Item: "PY"
CALCULATE ( SELECTEDMEASURE (), SAMEPERIODLASTYEAR ( 'Date'[Date] ) )

-- Item: "YoY %"
VAR Cur = SELECTEDMEASURE ()
VAR Py  = CALCULATE ( SELECTEDMEASURE (), SAMEPERIODLASTYEAR ( 'Date'[Date] ) )
RETURN DIVIDE ( Cur - Py, Py )
```

Consulte-os como qualquer dimensão:

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'Date'[CalendarYear], 'Date'[EnglishMonthName],
    "Current", CALCULATE ( [Sales], 'Time Intelligence'[Time Calculation] = "Current" ),
    "YTD",     CALCULATE ( [Sales], 'Time Intelligence'[Time Calculation] = "YTD" ),
    "PY",      CALCULATE ( [Sales], 'Time Intelligence'[Time Calculation] = "PY" )
)
```

---

## 5. Segmentação por percentil de clientes (CORRIGIDA)

A versão bugada comum usa `PERCENTILE.INC(table, expr, k)` — função errada para
tabela+expressão. Use **`PERCENTILEX.INC(table, expr, k)`**, e compare o cliente
*em contexto* contra limiares computados sobre todos os clientes.

```dax
Customer Value Tier =
VAR CurrentRevenue = [Total Revenue]
VAR AllCustomers =
    ADDCOLUMNS ( ALL ( Customer[CustomerKey] ), "@Rev", CALCULATE ( [Total Revenue] ) )
VAR P80 = PERCENTILEX.INC ( AllCustomers, [@Rev], 0.80 )
VAR P50 = PERCENTILEX.INC ( AllCustomers, [@Rev], 0.50 )
RETURN
    SWITCH (
        TRUE (),
        CurrentRevenue >= P80, "High Value",
        CurrentRevenue >= P50, "Medium Value",
        "Standard"
    )
```

Por que a original falhava: `PERCENTILE.INC` recebe uma única coluna, não uma
tabela + expressão; e um `SUMX` sobre `VALUES(Customer[...])` produz um único total
geral, então não havia nada para comparar por cliente.

---

## 6. Ranking (RANKX)

Uma alternativa limpa e robusta aos padrões de TOPN-numa-variável feitos à mão:

```dax
Customer Revenue Rank =
RANKX ( ALL ( Customer[CustomerName] ), [Total Revenue],, DESC, DENSE )
```

```dax
// Flag "Top 5 clientes", seguro para empates
Is Top 5 Customer =
IF ( [Customer Revenue Rank] <= 5, "Top 5", "Other" )
```

---

## 7. Anti-padrões: antes → depois

```dax
-- Tratamento de erro ineficiente
-- ANTES
Profit Margin = IF ( ISERROR ( [Profit] / [Sales] ), BLANK (), [Profit] / [Sales] )
-- DEPOIS
Profit Margin = DIVIDE ( [Profit], [Sales] )
```

```dax
-- Cálculo repetido
-- ANTES
Sales Growth =
DIVIDE (
    [Sales] - CALCULATE ( [Sales], PARALLELPERIOD ( 'Date'[Date], -12, MONTH ) ),
    CALCULATE ( [Sales], PARALLELPERIOD ( 'Date'[Date], -12, MONTH ) )
)
-- DEPOIS
Sales Growth =
VAR Prior = CALCULATE ( [Sales], PARALLELPERIOD ( 'Date'[Date], -12, MONTH ) )
RETURN DIVIDE ( [Sales] - Prior, Prior )
```

```dax
-- BLANK-para-zero desnecessário
-- ANTES
Sales with Zero = IF ( ISBLANK ( [Sales] ), 0, [Sales] )
-- DEPOIS
Sales = SUM ( Sales[Amount] )   -- deixe BLANK continuar BLANK
```

```
-- Disciplina de referência
-- ANTES:  Sales[Total Sales]  (medida, qualificada errado)   [Amount]  (coluna, não qualificada)
-- DEPOIS: [Total Sales]                                       Sales[Amount]
```
