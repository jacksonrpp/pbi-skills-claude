---
name: power-bi-report-design
description: >-
  Design especialista de relatórios e dashboards Power BI — seleção de visual,
  layout de página, UX, acessibilidade, mobile, interatividade (tooltips,
  drillthrough, filtragem cruzada), temas de relatório e performance da camada de
  relatório — seguindo as melhores práticas da Microsoft. Use esta skill sempre
  que o usuário estiver projetando, revisando ou corrigindo a APARÊNCIA e o
  COMPORTAMENTO de um relatório: qual gráfico usar, como distribuir uma página,
  montar um tema JSON, páginas lentas/poluídas, formatação condicional, layout
  mobile, ou editar arquivos de relatório num projeto .pbip / PBIR. Diferente das
  skills de modelagem e DAX, esta é a camada de relatório e NÃO passa pelo MCP de
  modelagem do Power BI. Para o modelo de dados (tabelas/relações/armazenamento)
  use power-bi-data-modeling; para a lógica de medidas use power-bi-dax; para
  diagnóstico profundo de consulta/capacidade use power-bi-performance.
---

# Especialista em Design de Relatórios Power BI

Atue como especialista em design de relatórios e visualização de dados: ajude a
escolher o visual certo para a história do dado, distribua páginas para
compreensão rápida, torne relatórios acessíveis e amigáveis a mobile, e produza
artefatos concretos (um tema JSON, arquivos de relatório editados). O design serve
ao leitor — cada escolha deve tornar o insight mais rápido de alcançar, não apenas
mais bonito.

## Importante: esta é a camada de relatório, não o modelo

O MCP de modelagem do Power BI opera no **modelo semântico** (tabelas, medidas,
relações) — ele **não** cria nem edita visuais, páginas ou temas. Então a
superfície prática desta skill é diferente:

- **Orientação** — o grosso do trabalho: recomendar visuais, layout, interações,
  cor, acessibilidade, e as correções para um relatório específico.
- **Editar arquivos de relatório** — num projeto `.pbip` o relatório é armazenado
  como **PBIR** (`definition/` com `report.json`, `pages/`, e um JSON por visual).
  Você pode ler e editar esses JSON diretamente para mudar layout, formatação e
  interações. Mantenha as edições mínimas e válidas; faça backup ou trabalhe numa
  cópia, pois um JSON de visual malformado pode impedir o relatório de abrir.
- **Autorar um tema** — um tema de relatório JSON é um entregável real que você
  produz e devolve (ver `references/theme-and-files.md`).
- **Embedding** (Power BI Embedded, `powerbi-client`) é um caso de nicho — recorra
  à referência de embedding só quando o usuário estiver de fato incorporando.

Não dê a entender que você consegue criar visuais pelo MCP. Quando o problema real
for um número errado ou uma medida lenta, isso é a camada de **modelo/DAX** — passe
para as skills irmãs.

## Ancoragem na documentação da Microsoft

Capacidades de visual e o schema de tema mudam. Quando algo for específico de
versão/recurso (uma propriedade de tema, um visual novo, uma configuração de
acessibilidade), verifique: consulte um MCP de documentação da Microsoft se houver
um conectado (uma ferramenta referenciando `microsoft.docs`/`learn`), senão
busque/acesse `learn.microsoft.com`. Se nenhum estiver disponível, prossiga pelos
princípios abaixo e diga que não conseguiu verificar ao vivo.

## Comece pela história do dado, depois pelo leitor

Antes de recomendar visuais, defina duas coisas: **qual pergunta a página
responde** e **quem lê e como** (relance executivo vs. exploração de analista vs.
ação operacional). O tipo de relatório orienta a maioria das escolhas seguintes.

## Escolhendo o visual

Case o visual com a *relação* no dado, não com novidade. Barras e linhas respondem
à maioria das perguntas; visuais exóticos são último recurso.

- **Comparação entre categorias** → barra/coluna (barras horizontais quando os
  rótulos são longos ou há muitas categorias).
- **Tendência ao longo do tempo** → linha (ou área para acumulado); não use
  espaguete de muitas séries — pequenos múltiplos leem melhor.
- **Parte-de-um-todo** → prefira uma barra dos componentes; pizza/rosca só para
  poucas fatias (≈2–5, um todo) onde a proporção é o ponto central. Treemap para
  muitas partes hierárquicas.
- **Correlação / distribuição** → dispersão (correlação), histograma/box (forma),
  mapa de calor (densidade em duas dimensões).
- **Fluxo / mudança sequencial** → cascata (passos), Sankey (fluxos).
- **Um único número acompanhado** → cartão/KPI com um indicador de tendência
  claro.

Recorra a tabela/matriz quando o leitor precisa consultar valores exatos, não
comparar formas. Evite eixos duplos e 3D — enganam. Nota: rosca, como pizza, é um
todo dividido em fatias — não "múltiplas medidas".

## Layout e hierarquia

- **Comece pela resposta.** Conteúdo mais importante no topo à esquerda; KPIs numa
  faixa de cabeçalho; detalhe de apoio abaixo; filtros no topo ou à esquerda. O
  leitor varre em padrão Z/F — ponha o recado onde o olho pousa primeiro.
- **Agrupe visuais relacionados**, alinhe a uma grade, mantenha espaçamento
  consistente e dê respiro à página. Alinhamento e consistência transmitem
  "confiável".
- **Uma página, um trabalho.** Se uma página faz três coisas, são três páginas (ou
  abas, favoritos, drillthrough). Poluição prejudica tanto a compreensão quanto a
  velocidade.

Arquétipos de relatório: **dashboard executivo** (poucos KPIs + 2–3 visuais,
destaque de exceções, texto mínimo), **relatório analítico** (múltiplos níveis de
detalhe, filtragem rica, drillthrough, navegação por abas/favoritos), **relatório
operacional** (dado fresco, cores de status, orientado à ação, amigável a mobile).

## Interatividade

- **Tooltips** — adicionam detalhe de apoio, não uma segunda história. Tooltips de
  página de relatório (~320×240) são ótimos para um mini-painel de contexto;
  mantenha-os rápidos e na identidade visual.
- **Drillthrough** — de um ponto de resumo para o detalhe dele (um mês → suas
  transações; um produto → seu perfil completo). Sinalize que está disponível,
  aplique o filtro de contexto automaticamente, adicione um botão Voltar, e oculte
  a página de destino da navegação.
- **Filtragem cruzada** — ligada por padrão para visuais relacionados que de fato
  se informam; use *Editar interações* para desligar onde confunde ou atrasa. Um
  botão *Aplicar* em segmentação (slicer) evita reconsultar a cada clique em
  relatórios pesados.

## Acessibilidade (projete para ela, não a acrescente depois)

- **Contraste ≥ 4.5:1** para texto; não codifique significado só na cor — combine
  com ícone, rótulo ou posição. Use paletas seguras para daltônicos e teste-as.
- Defina **ordem de tabulação** e **texto alternativo** nos visuais; mantenha
  títulos e rótulos de eixo significativos. Tipografia legível: sem-serifa, ~≥10pt,
  hierarquia de tamanho clara.
- Acessibilidade e clareza são o mesmo objetivo — um relatório que passa em
  contraste e rotulagem é mais fácil de ler para *todos*.

## Cor e tipografia

- **Cor semântica e consistente**: reserve vermelho/verde para ruim/bom (e não
  dependa só deles), um neutro para o resto; aplique a paleta da marca pelo **tema**
  para ficar consistente em tudo.
- **Hierarquia tipográfica**: título de página grande/negrito, cabeçalhos de seção
  médios, corpo regular, legendas pequenas — poucos tamanhos, uma ou duas famílias.
  Rótulos concisos e orientados à ação; todo gráfico merece um título significativo.

## Formatação condicional

Use barras de dados, ícones e regras de cor de fundo/fonte para destacar as linhas
importantes e codificar status — sempre em escala consistente e, de novo, nunca só
por cor. Ótimo para tabelas/matrizes e cartões de KPI onde o leitor varre em busca
de exceções.

## Performance da camada de relatório (esta skill é dona)

Escolhas de layout dirigem boa parte da velocidade do relatório — este é o lado
design da performance (diagnóstico profundo de consulta/capacidade fica em
`power-bi-performance`):

- **Menos visuais por página** (regra de bolso ~6–8; ajuste ao seu dado/capacidade)
  — cada visual é pelo menos uma consulta. Divida páginas poluídas; use abas/
  favoritos/drillthrough.
- **Filtre cedo e estreito** — filtros de página/visual e padrões sensatos reduzem
  o dado varrido no carregamento; evite segmentações de alta cardinalidade;
  adicione botões *Aplicar* em páginas pesadas.
- **Enxugue interações** — desligue cross-highlight onde não é necessário; menos
  caminhos de interação = menos consultas por clique.
- **Meça** — verifique o **Performance Analyzer** no Desktop; se o *DAX de um
  visual* for o gargalo, isso é trabalho de `power-bi-dax`.

## Mobile

Projete um **layout mobile (retrato) dedicado**, não mande a página desktop para o
celular: destaque métricas-chave, use alvos de toque adequados, reduza densidade,
simplifique tipos de gráfico, e teste na visão de layout mobile e num aparelho
real.

## Visuais personalizados

Prefira os embutidos. Se um visual personalizado for necessário, pese a
**certificação** no AppSource, manutenção/atualizações, documentação e performance,
teste com dado real, respeite a governança da organização, e mantenha em mente um
fallback embutido.

## Checklist de testes

Funcionalidade (interações, filtros, drillthrough, exportação, mobile) ·
Performance (carga < ~10s, interação < ~3s — metas, não leis; sem erros de
renderização) · Usabilidade (navegação intuitiva, nível de detalhe certo,
acionável, acessível) · em Desktop/Service/Mobile e nos principais navegadores.

## Entre skills

- Números errados/em branco ou *medidas* lentas → **power-bi-dax**.
- Tamanho de modelo, relações, modo de armazenamento, RLS → **power-bi-data-modeling**.
- Diagnóstico de capacidade/gateway/motor de consulta e triagem → **power-bi-performance**.

## Arquivos de referência

- `references/theme-and-files.md` — **tema JSON** de relatório anotado (dataColors,
  classes de texto, visualStyles) com dicas de autoria/validação, o layout de
  arquivos **PBIR** para editar arquivos de relatório num projeto `.pbip`, e — para
  o caso de nicho de embedding — os trechos de custom-layout / createVisual do
  `powerbi-client`.
