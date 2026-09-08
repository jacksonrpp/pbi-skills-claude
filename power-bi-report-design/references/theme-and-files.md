# Temas de Relatório, Arquivos PBIR & Embedding

Artefatos e detalhes de arquivo para a `power-bi-report-design`. Esta camada **não**
passa pelo MCP de modelagem do Power BI — você produz um tema JSON e/ou edita os
arquivos do projeto de relatório diretamente.

## Conteúdo

1. Tema de relatório JSON (autoria + validação)
2. PBIR: editando arquivos de relatório num projeto .pbip
3. Embedding (nicho): custom layout do powerbi-client

---

## 1. Tema de relatório JSON

Um tema é um arquivo JSON que o usuário importa (Exibir → Temas → Procurar). Ele
define a paleta, as classes de texto e a formatação padrão dos visuais para que o
estilo seja consistente e editável de forma central. Estrutura mínima mas real:

```json
{
  "name": "Corporate Theme",
  "dataColors": ["#31B6FD", "#4584D3", "#5BD078", "#A5D028", "#F5C040",
                 "#05E0DB", "#3153FD", "#4C45D3", "#5BD0B0", "#54D028"],
  "background": "#FFFFFF",
  "foreground": "#252423",
  "tableAccent": "#5BD078",
  "textClasses": {
    "title":    { "fontFace": "Segoe UI Semibold", "fontSize": 14, "color": "#252423" },
    "header":   { "fontFace": "Segoe UI Semibold", "fontSize": 12 },
    "label":    { "fontFace": "Segoe UI",          "fontSize": 10 },
    "callout":  { "fontFace": "Segoe UI",          "fontSize": 28 }
  },
  "visualStyles": {
    "*": {
      "*": {
        "wordWrap": [ { "show": true } ],
        "categoryAxis": [ { "gridlineStyle": "dotted" } ]
      }
    }
  }
}
```

Dicas de autoria:

- **`dataColors`** é a paleta de séries categóricas — a ordem importa; ponha a cor
  primária da marca primeiro. Mantenha matizes distintos suficientes e verifique-os
  quanto à segurança para daltônicos (não dependa de adjacência vermelho/verde para
  carregar significado).
- **`textClasses`** dirigem a hierarquia tipográfica globalmente — defina
  `title`/`header`/`label`/`callout` uma vez em vez de por visual.
- **`visualStyles`** usa `"visualType": { "styleName": { "property": [ {...} ] } }`;
  `"*"` significa "todos". Valores de propriedade quase sempre são um **array de
  objetos**, o que confunde as pessoas — case essa forma exatamente.
- O schema de tema **evolui**; ao adicionar propriedades menos comuns, verifique os
  nomes contra a doc atual de temas em `learn.microsoft.com` em vez de adivinhar.
- **Valide antes de entregar**: precisa ser JSON bem-formado (sem vírgulas finais /
  comentários). Faça o lint, ex.: `python -m json.tool theme.json`, e se possível
  confirme que importa limpo no Desktop.

---

## 2. PBIR: editando arquivos de relatório num projeto .pbip

Um projeto `.pbip` separa o relatório do modelo. O relatório fica sob uma pasta
`*.Report/definition/` em **PBIR** (Power BI Enhanced Report format), mais ou menos
assim:

```
MyReport.Report/
└── definition/
    ├── report.json           # configurações de nível de relatório, ref do tema
    ├── pages/
    │   ├── pages.json         # ordem das páginas / página ativa
    │   └── <pageName>/
    │       ├── page.json      # tamanho da página, filtros, nome de exibição
    │       └── visuals/
    │           └── <visualId>/visual.json   # um arquivo por visual
```

Orientação de edição:

- Cada **`visual.json`** guarda o tipo do visual, os mapeamentos de campo, a
  posição (`x`/`y`/`z`/`width`/`height`) e os objetos de formatação. Você pode
  editar layout e formatação aqui — ex.: alinhar/redimensionar visuais, definir
  cores, alternar um título.
- **Trabalhe numa cópia ou faça commit antes.** Um JSON de visual malformado pode
  impedir a página (ou o relatório) de abrir. Mantenha os diffs pequenos e reabra
  no Desktop para confirmar.
- Prefira mudar **tema/textClasses** para qualquer coisa global em vez de editar
  cada visual — menos a quebrar, mais fácil de revisar.
- Esses arquivos também são o que torna os relatórios **comparáveis (diff) em
  controle de versão** — bom para revisar mudanças de design num PR.

Nomes de campo/propriedade no PBIR acompanham o produto e podem mudar; em dúvida
sobre uma propriedade, confira a doc atual ou copie a forma de um visual existente
que funcione no mesmo projeto em vez de inventar chaves.

---

## 3. Embedding (nicho) — custom layout do powerbi-client

Só relevante ao incorporar um relatório num app web próprio (Power BI Embedded).
Custom layout permite posicionar/ocultar contêineres de visual programaticamente:

```javascript
let models = window["powerbi-client"].models;

let embedConfig = {
  type: "report",
  id: reportId,
  embedUrl: "https://app.powerbi.com/reportEmbed",
  tokenType: models.TokenType.Embed,
  accessToken: token,
  settings: {
    layoutType: models.LayoutType.Custom,
    customLayout: {
      pageSize: { type: models.PageSizeType.Custom, width: 1600, height: 1200 },
      displayOption: models.DisplayOption.ActualSize,
      pagesLayout: {
        "ReportSection1": {
          visualsLayout: {
            "VisualContainer1": {
              x: 1, y: 1, z: 1, width: 400, height: 300,
              displayState: { mode: models.VisualContainerDisplayMode.Visible }
            }
          }
        }
      }
    }
  }
};
```

Criação programática de visual (APIs de autoria):

```javascript
const layout = { x: 20, y: 35, width: 1600, height: 1200 };
const res = await page.createVisual("areaChart", layout, false /* autoFocus */);
```

Para qualquer coisa além de um trecho, aponte o usuário à documentação atual do
`powerbi-client` — a superfície da API de embedding muda e tokens/auth devem ser
tratados no servidor, nunca fixos no código.
