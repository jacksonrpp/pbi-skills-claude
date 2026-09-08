# Modelagem Power BI — Padrões de Implementação (MCP-first)

Receitas prontas para copiar dos padrões do `SKILL.md`, com os payloads reais do
`powerbi-modeling-mcp`. Leia a seção de que precisa.

Toda ferramenta é `<nome>_operations` e recebe `{ "request": { "operation": ... } }`.
Quando estiver em dúvida sobre os campos, chame `operation: "Help"` naquela
ferramenta — ela retorna a própria referência de parâmetros. Escritas aceitam
`options.useTransaction` (padrão `true`) e a maioria das operações aceita **listas**
para lote (batch).

## Conteúdo

1. Referência rápida do MCP (operações por ferramenta)
2. Inspecionar um modelo antes de aconselhar
3. Relações (criar / corrigir / role-playing)
4. Medidas (criar + validar)
5. Segurança em nível de linha (estática + dinâmica)
6. Partições — atualização incremental & composite hot/cold
7. Agregações sobre DirectQuery
8. Dimensões de variação lenta (Tipo 2)
9. Muitos-para-muitos via tabela ponte

---

## 1. Referência rápida do MCP

| Ferramenta | Operações principais |
|---|---|
| `database_operations` | List, Create, ExportTMDL, ExportTMSL, ImportFromTmdlFolder, ExportToTmdlFolder, DeployToFabric |
| `table_operations` / `column_operations` | List, Get, Create, Update, Delete, Rename, ExportTMDL |
| `relationship_operations` | List, Get, Create, Update, Delete, Rename, Activate, Deactivate, Find |
| `measure_operations` | List, Get, Create, Update, Delete, Rename, Move, ExportTMDL |
| `dax_query_operations` | Validate, Execute, ClearCache |
| `security_role_operations` | Create/Update/Delete papel, CreatePermissions, GetEffectivePermissions |
| `partition_operations` | List, Get, Create, Update, Delete, Refresh, Rename |
| `named_expression_operations` | Create/Update, CreateParameter, UpdateParameter (M / RangeStart / RangeEnd) |
| `calendar_operations` | Create/Update calendários lógicos (nível de compatibilidade ≥ 1701) |

Leituras (List/Get/Find/Validate/Execute/ExportTMDL) são seguras. Create/Update/
Delete/Rename/Refresh/Deploy mudam o modelo do usuário — confirme antes.

---

## 2. Inspecionar um modelo antes de aconselhar

```json
// A que estou conectado?
{ "request": { "operation": "List" } }              // database_operations
```

```json
// Todas as relações com cardinalidade, direção, flag de ativa
{ "request": { "operation": "Execute",
               "query": "EVALUATE INFO.VIEW.RELATIONSHIPS()" } }  // dax_query_operations
```

Também úteis: `INFO.VIEW.TABLES()`, `INFO.VIEW.COLUMNS()`, `INFO.VIEW.MEASURES()`
para achar colunas de alta cardinalidade, chaves ausentes, e relações
bidirecionais perdidas. Ou `relationship_operations` → `List` para a mesma
informação de relação como objetos estruturados.

---

## 3. Relações

```json
// Criar Dim → Fato um-para-muitos, direção única
{ "request": { "operation": "Create", "definitions": [ {
  "name": "Sales_Product",
  "fromTable": "Sales",  "fromColumn": "ProductKey",  "fromCardinality": "Many",
  "toTable":   "Product","toColumn":   "ProductKey",  "toCardinality":   "One",
  "crossFilteringBehavior": "OneDirection",
  "isActive": true,
  "relyOnReferentialIntegrity": false
} ] } }                                              // relationship_operations
```

Datas role-playing: crie a segunda relação como `"isActive": false`, depois
exponha-a numa medida com `USERELATIONSHIP` (abaixo). Para alternar uma existente:

```json
{ "request": { "operation": "Deactivate", "references": [ { "name": "Sales_ShipDate" } ] } }
```

Só defina `"crossFilteringBehavior": "BothDirections"` quando uma necessidade
específica de M:M ou segurança exigir, e confirme que nenhum caminho ambíguo
resulta disso.

---

## 4. Medidas (validar, depois criar)

```json
// 1) Valide o DAX primeiro
{ "request": { "operation": "Validate",
  "query": "EVALUATE ROW(\"x\", CALCULATE(SUM(Sales[Amount]), USERELATIONSHIP('Date'[DateKey], Sales[ShipDateKey])))" } }  // dax_query_operations
```

```json
// 2) Crie a medida quando validar
{ "request": { "operation": "Create", "definitions": [ {
  "name": "Sales by Ship Date",
  "tableName": "Sales",
  "expression": "CALCULATE([Total Sales], USERELATIONSHIP('Date'[DateKey], Sales[ShipDateKey]))",
  "formatString": "\\$#,0",
  "displayFolder": "Sales \\ By Date Role"
} ] } }                                              // measure_operations
```

Para realocar uma medida, use `Move` (Update não muda `tableName`).

---

## 5. Segurança em nível de linha (RLS) — estática e dinâmica

Filtre a **dimensão**; deixe propagar para o fato.

```json
// Crie o papel
{ "request": { "operation": "Create", "definitions": [
  { "name": "RegionalManager", "modelPermission": "Read" } ] } }  // security_role_operations
```

```json
// Estática: segmento fixo
{ "request": { "operation": "CreatePermissions", "permissionDefinitions": [ {
  "roleName": "RegionalManager",
  "tableName": "Geography",
  "filterExpression": "'Geography'[Region] = \"Europe\""
} ] } }
```

```json
// Dinâmica: resolve o usuário atual contra uma tabela de mapeamento oculta
{ "request": { "operation": "CreatePermissions", "permissionDefinitions": [ {
  "roleName": "RegionalManager",
  "tableName": "Geography",
  "filterExpression": "'Geography'[Region] = LOOKUPVALUE('UserRegion'[Region], 'UserRegion'[Email], USERPRINCIPALNAME())"
} ] } }
```

Verifique antes de publicar:

```json
{ "request": { "operation": "GetEffectivePermissions",
               "permissionReferences": [ { "roleName": "RegionalManager", "tableName": "Sales" } ] } }
```

---

## 6. Partições — atualização incremental & composite

Defina primeiro os parâmetros M reservados, mantendo o **query folding** vivo:

```json
{ "request": { "operation": "CreateParameter", "definitions": [
  { "name": "RangeStart", "kind": "M", "expression": "#datetime(2020,1,1,0,0,0) meta [IsParameterQuery=true, Type=\"DateTime\", IsParameterQueryRequired=true]" },
  { "name": "RangeEnd",   "kind": "M", "expression": "#datetime(2020,2,1,0,0,0) meta [IsParameterQuery=true, Type=\"DateTime\", IsParameterQueryRequired=true]" }
] } }                                                // named_expression_operations
```

M de partição folding-friendly (preferido — o filtro é empurrado para a fonte):

```json
{ "request": { "operation": "Create", "definitions": [ {
  "tableName": "FactInternetSales",
  "name": "Incremental",
  "mode": "import",
  "sourceType": "m",
  "expression": "let Source = Sql.Database(\"dwdev02\",\"AdventureWorksDW2017\"), Data = Source{[Schema=\"dbo\",Item=\"FactInternetSales\"]}[Data], f1 = Table.SelectRows(Data, each [OrderDate] >= RangeStart), f2 = Table.SelectRows(f1, each [OrderDate] < RangeEnd) in f2"
} ] } }                                              // partition_operations
```

Composite hot/cold: crie uma partição `"mode": "directQuery"` para o dado recente e
uma partição `"mode": "import"` para o histórico no mesmo fato, e coloque as
dimensões compartilhadas em **Dual** (modo de armazenamento via `table_operations` /
`column_operations`). Atualize uma partição específica:

```json
{ "request": { "operation": "Refresh", "refreshDefinitions": [
  { "tableName": "FactInternetSales", "name": "Incremental", "refreshType": "Full" } ] } }
```

> Alternativa com SQL nativo (`Value.NativeQuery(..., [EnableFolding=false])`)
> funciona mas **desativa o folding** — evite para atualização incremental a menos
> que seja forçado.

---

## 7. Agregações sobre DirectQuery

Importe uma tabela pequena pré-agregada sobre um fato grande em DirectQuery:

```dax
Monthly Sales Summary =
SUMMARIZECOLUMNS(
    'Date'[Year Month], 'Product'[Category], 'Geography'[Country],
    "Total Sales", SUM(Sales[Amount]),
    "Transaction Count", COUNTROWS(Sales)
)
```

Crie-a como uma tabela/partição Import, depois mapeie-a ao fato de detalhe para que
o motor redirecione consultas correspondentes ao cache e só o drill-through
alcance a fonte.

---

## 8. Dimensões de variação lenta (Tipo 2)

Adicione linhas por versão em vez de sobrescrever. Colunas a adicionar à dimensão:
**chave surrogate** (o fato junta por esta), **chave de negócio/natural**,
**EffectiveStartDate / EffectiveEndDate**, e uma flag **IsCurrent**. O Tipo 1
apenas sobrescreve o atributo no lugar — use quando o histórico não importa.

---

## 9. Muitos-para-muitos via tabela ponte

```
Customer  1──*  CustomerProductBridge  *──1  Product
```

Crie a ponte (uma linha por par válido, com suas próprias chaves) e duas relações
um-para-muitos convergindo das dimensões (`relationship_operations` → `Create`,
como no §3). Prefira isso a uma relação M:M crua; se precisar filtrar cruzado pela
ponte, ponha `BothDirections` só nas relações da ponte e confirme que não há
caminhos ambíguos.
