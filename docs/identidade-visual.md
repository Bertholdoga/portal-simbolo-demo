# Identidade Visual — Portal Símbolo²
### Estratégia de design system para a proposta de substituição do sistema atual

Documento de estratégia visual. **Não altera código.** Serve de base aprovável antes da fase de implementação/refinamento visual das telas. Referencia o protótipo já existente em [`prototipo/index.html`](../prototipo/index.html) como ponto de partida — este documento formaliza, corrige e estende o que já lá está, não propõe começar do zero.

> **Estado**: direção visual (secções 1–2) **aprovada**. Este documento incorpora um conjunto de refinamentos às regras do design system (tipografia, espaçamento, radius, sombras, acessibilidade, tokens, estados semânticos) antes da implementação. Nenhum CSS/componente foi alterado por esta revisão.

---

## 1. Diagnóstico da identidade visual atual

**Do sistema em produção (`app.simbolo2.pt`)**: identidade praticamente inexistente enquanto produto — navy genérico de framework por defeito (Bootstrap datado), sem paleta pensada, sem hierarquia tipográfica, sem sistema de estados coerente (cores de tabela e de menu não relacionadas entre si). Não transmite nada de propositado; é um formulário funcional, não um produto com marca.

**Do protótipo já construído neste projeto**: parte de uma base bastante mais sólida — já tem tokens de cor, tipografia emparelhada (Sora/IBM Plex Sans/IBM Plex Mono), modo claro/escuro, pills de estado, cartões de resumo e uma navegação lateral por categorias. Ainda assim, para servir de proposta formal à empresa, tem pontos fracos concretos:

- **Nunca foi documentado como sistema.** As decisões existem no CSS, mas não há um documento que explique a lógica — o que torna difícil manter consistência à medida que mais ecrãs forem adicionados (ex. Arquivo de férias, Extrato, novos tipos de despesa).
- **Paleta funcional mas ainda genérica.** O teal usado (`#0e7c86`) é uma escolha segura, mas não tem uma explicação de marca por trás — não está clara a ligação deliberada ao logótipo real da Símbolo² (a seta/estrela azul-turquesa do logótipo atual), para além de partilhar a mesma família de cor.
- **Semântica de estado sobreposta.** A mesma classe `.pill.ok` é usada para "Aprovado" (férias), "Aceite" (despesas) e "Presença" (assiduidade) — visualmente correto, mas conceptualmente estes são três tipos de "positivo" ligeiramente diferentes que nunca foram diferenciados de propósito; hoje funciona por coincidência de implementação, não por uma regra de sistema.
- **Ícones ad hoc.** Os SVGs inline foram desenhados um a um, sem grelha ou conjunto de referência — resultado visualmente aceitável, mas sem garantia de consistência de peso de traço (`stroke-width`) e proporção à medida que o número de ícones cresce.
- **Espaçamento não documentado.** Os valores de padding/gap "funcionam" mas não seguem uma escala nomeada — não há forma de garantir que um novo ecrã use os mesmos incrementos sem copiar visualmente um ecrã existente.
- **Sem hierarquia de elevação explícita.** Existem duas sombras (`--shadow-sm`, `--shadow-md`) mas sem regra de "quando usar qual" além do que já foi aplicado por instinto.

Resumindo: a base é boa e não deve ser deitada fora — falta formalizá-la, corrigir as inconsistências de semântica de estado, e dar-lhe uma justificação de marca deliberada em vez de "pareceu bem".

---

## 2. Direção visual recomendada

**Personalidade proposta: executiva, sóbria e utilitária — com um único ponto de cor deliberado.**

Não "mais sóbrio" ou "mais premium" isoladamente — a combinação certa para um sistema interno de gestão de despesas é: **sóbrio na superfície, rápido na leitura, com confiança visual concentrada num único acento de cor.** Isto significa:

- **Neutro faz o trabalho pesado.** A maior parte da interface (fundos, texto, bordas, tabelas) deve ser neutra e discreta — é onde o utilizador passa 95% do tempo (ler valores, preencher formulários), e neutros bem calibrados comunicam "sistema sério" mais do que cor.
- **Cor é reservada para significado, não decoração.** O acento de marca (teal) aparece só onde orienta ação (botões primários, links, foco) ou identidade (navegação ativa, wordmark). As cores de estado (sucesso/aviso/erro) são semânticas e nunca se confundem com a cor de marca.
- **Densidade moderada-alta.** É um sistema de trabalho diário para técnicos e contabilidade, não uma landing page — prioriza caber mais informação útil por ecrã sem parecer apertado, em vez de espaçamento generoso "para respirar" como um produto de marketing.
- **Nada de gradientes decorativos, ilustrações ou elementos "de produto de consumo".** Os únicos usos de gradiente aceitáveis são utilitários (ex. o painel de marca no login, já existente) — nunca em botões, cartões ou tabelas.

Porquê esta direção e não outra: um sistema de despesas empresarial vive ou morre pela confiança nos números e pela velocidade de preenchimento — não pela beleza da primeira impressão. "Premium" a mais (sombras pesadas, muita cor, motion excessivo) contradiz "rápido e sério"; "minimalista" a menos (sem nenhuma cor de estado, tudo cinza) prejudica a leitura rápida de que despesa está aprovada, em análise ou rejeitada. O equilíbrio certo é neutro dominante + acento único + semântica de estado disciplinada.

---

## 3. Sistema visual proposto

### 3.1 Paleta de cores

A paleta já implementada no protótipo está tecnicamente bem construída (tokens claros/escuros coerentes) — formalizo aqui a **lógica de uso**, com um ajuste de nomenclatura para separar estados que hoje partilham cor sem razão de fundo.

| Token | Papel | Valor (claro) | Valor (escuro) |
|---|---|---|---|
| `--bg` | Fundo da aplicação | `#f4f6f8` | `#0d151c` |
| `--surface` | Fundo de cartões, tabelas, modais | `#ffffff` | `#131e27` |
| `--surface-2` | Fundo secundário (cabeçalhos de tabela, hover) | `#eef1f5` | `#182430` |
| `--border` | Divisórias, contornos de input | `#dde3ea` | `#243342` |
| `--ink` | Texto principal | `#151f2b` | `#e8edf2` |
| `--muted` | Texto secundário, labels | `#5c6b7a` | `#96a5b3` |
| `--muted-2` | Texto terciário, placeholders | `#8492a1` | `#6f7f8f` |
| `--accent` | Cor de marca — ações primárias, navegação ativa, foco | `#0e7c86` | `#43bcb4` |
| `--accent-soft` | Fundo suave para estados ativos/selecionados | `#e2f3f2` | `rgba(67,188,180,.15)` |
| `--accent-2` | Secundária de marca — usada só no painel de login/branding, nunca em ações | `#2f5f8a` | `#7fa8d4` |
| `--success` | Estado positivo (Aprovado, Aceite, Presença) | `#1c8a5a` | `#42d190` |
| `--warning` | Estado de atenção (Em análise, revisão necessária) | `#b6790a` | `#e3ab4a` |
| `--danger` | Estado negativo (Recusado, erro) | `#c53c3c` | `#e57972` |

**Lógica de utilização (a regra que faltava documentar):**
- `--accent` é **exclusivo de ação e identidade** — nunca usar para comunicar "isto está aprovado" ou "isto é urgente". Se o teal aparecer numa pill de estado, o utilizador vai confundir "estado" com "botão clicável".
- `--success` / `--warning` / `--danger` são **exclusivos de estado** — nunca usar num botão de ação primária (só em botões destrutivos explícitos, ex. "Remover", que usam `--danger` de forma consciente).
- **Ajuste recomendado**: apesar de tecnicamente correto, `.pill.ok` está hoje a servir três significados (aprovação de pedido, aceitação contabilística, presença/assiduidade). Manter a mesma cor `--success` para os três é correto — mas o **texto da pill tem de ser sempre explícito** ("Aprovado", "Aceite", "Presença"), nunca um ícone/cor sozinha a comunicar o significado. Isto já é o comportamento atual — só precisa de ficar documentado como regra, para não se perder à medida que novos estados forem adicionados (ex. "Pago", "Reembolsado").
- `--accent-2` fica reservado à área de marca (login, cabeçalhos institucionais) — não deve aparecer em componentes de trabalho do dia a dia (tabelas, formulários), para não competir com `--accent` como segunda cor de ação.

### 3.2 Tipografia

Responsabilidade de cada família **formalizada** (refinamento aprovado — antes só estava descrita por uso típico, agora é regra):

- **Sora** — reservada a **identidade do produto**: wordmark, título de página (H1), título de secção principal quando funciona como "assinatura" do bloco (H2 em certos contextos), e números de destaque muito grandes onde o número *é* a mensagem (ex. o valor no stat tile de saldo em destaque). Não usar em texto de trabalho do dia a dia.
- **IBM Plex Sans** — reservada a **interface**: formulários, labels, tabelas, dados textuais, botões, navegação, texto auxiliar, H3 e abaixo. É a família que o utilizador lê 95% do tempo.
- **IBM Plex Mono** — dados tabulares/numéricos onde o alinhamento de algarismos importa (datas, valores, IDs de documento), com `font-variant-numeric: tabular-nums`. Função utilitária, não estética.

**Regra de mistura**: nunca alternar Sora/IBM Plex Sans dentro do mesmo bloco de texto sem uma função clara a justificar — cada troca de família tem de significar algo (mudança de nível hierárquico ou de tipo de conteúdo), nunca variedade visual gratuita.

**Escala tipográfica formal:**

| Nível | Família | Tamanho | Peso | Line-height | Uso |
|---|---|---|---|---|---|
| H1 | Sora | 24px | 700 | 1.25 | Título de página |
| H2 | Sora | 16px | 600 | 1.3 | Título de cartão/painel/secção principal |
| H3 | IBM Plex Sans | 13.5px | 600 | 1.3 | Subtítulo dentro de um painel (ex. cabeçalho de subsecção de formulário) |
| Body | IBM Plex Sans | 14px | 400 | 1.5 | Texto corrente |
| Body small | IBM Plex Sans | 12.5–13px | 400 | 1.45 | Texto auxiliar, descrições curtas, `panel-sub` |
| Label | IBM Plex Sans | 12–12.5px | 600 | 1.3 | Labels de campo, cabeçalhos de tabela |
| Caption | IBM Plex Sans | 11.5–12px | 500 | 1.3 | Notas de rodapé, timestamps, hints, `dz-hint` |
| Button | Sora | 13.5px | 600 | 1 | Texto de botão |
| Numeric / Statistic | IBM Plex Mono | 13–24px conforme contexto | 500–600 | 1.2 | Valores monetários, datas, IDs, stat tiles |

Regra geral mantida: **nunca mais de 3 pesos de fonte visíveis simultaneamente no mesmo ecrã** (tipicamente 400 corpo / 600 ênfase-labels / 700 só no H1) — a escala acima já respeita isto por construção.

### 3.3 Estilo de componentes

**Botões**
- Primário: fundo `--accent`, texto `--accent-ink`, sem borda. Único botão "cheio de cor" por ecrã na maioria dos casos — reforça que é a ação principal.
- Secundário/ghost: fundo transparente, borda `--border`, texto `--ink`. Usado para "Cancelar" e ações de apoio.
- Terciário/soft: fundo `--accent-soft`, texto `--accent` — para ações relevantes mas não a principal (ex. já usado nalguns contextos do protótipo).
- Destrutivo: fundo `--danger-soft`, texto `--danger` em contexto secundário; só usar vermelho cheio (`--danger` sólido) em confirmações finais de eliminação, nunca no botão "Remover" de uma linha de tabela (esse é secundário até ser confirmado).
- Disabled: opacidade reduzida (~50%), cursor `not-allowed`, sem hover — nunca remover o texto, só desativar a interação.
- Loading: manter o texto do botão, substituir/acompanhar por um spinner discreto — nunca trocar o texto por "A carregar…" de forma que mude a largura do botão (evita saltos de layout).

**Inputs e selects**
- Normal: borda `--border`, fundo `--surface`.
- Focus: borda `--accent` + anel suave `--accent-soft` (já implementado) — manter, é um padrão de acessibilidade sólido.
- Preenchido (detetado por OCR): borda `--accent` + etiqueta "Detetado"/"Confirmar" — já implementado no fluxo de despesas, é o padrão a replicar sempre que houver preenchimento automático no sistema.
- Erro: borda `--danger`, mensagem de erro em `--danger` por baixo do campo, nunca só a borda vermelha sem texto explicativo.
- Disabled: fundo `--surface-2`, texto `--muted-2`, sem borda de foco.

**Cards**
- Cartão de resumo (stat tile): usar o acento cheio (`--accent`/`--accent-2` em gradiente) só no cartão mais relevante **dentro do mesmo contexto de leitura** (ex. "Total provisório" dentro do painel de saldo) — os restantes ficam neutros (`--surface` + `--border`). Isto já é o padrão do protótipo; a regra é **no máximo 1 cartão com cor cheia por contexto de leitura principal**, não necessariamente 1 por ecrã inteiro se o ecrã tiver secções claramente distintas (ver regra de ação dominante, secção 4).
- Cartão informativo (painel): sempre `--surface` + `--border` + `--shadow-sm`; título em Sora 600.

**Tabelas**
- Cabeçalho: `--surface-2`, texto `--muted`, maiúsculas pequenas com leve letter-spacing (já implementado) — mantém a tabela "administrativa" e fácil de escanear.
- Linhas: sem zebra-striping (já assim) — usar só `hover: --surface-2` para indicar linha ativa; zebra-striping tende a competir visualmente com as pills de estado.
- Estado vazio: texto centrado em `--muted-2`, nunca só um espaço em branco — sempre com uma frase que explique o que fazer a seguir (ex. "Ainda não tem despesas registadas este mês").
- Leitura rápida: valores monetários sempre alinhados à direita com fonte tabular (já implementado); datas em formato ISO consistente em toda a aplicação.

**Badges / estados**
- Pill sempre com texto explícito + cor semântica (nunca só cor, nunca só ícone) — ver regra da secção 3.1.
- Novos estados futuros (ex. "Pago", "Reembolsado", "Cancelado") devem ser mapeados para uma das 4 famílias semânticas existentes (`ok`/`pending`/`bad`/`neutral`) antes de se inventar uma cor nova — só criar uma 5ª cor de estado se nenhuma das 4 servir conceptualmente, e nesse caso documentar aqui.

**Modal "Nova despesa"**
- Manter a hierarquia já usada: cabeçalho fixo (título + fechar), corpo com scroll interno, rodapé fixo com ações — está correto e deve ser o padrão para todos os modais futuros do sistema, não só despesas.
- Agrupamento visual: campos relacionados lado a lado (já implementado com `field-grid`), campos condicionais (combustível/portagens) aparecem/desaparecem sem saltos bruscos — manter transição suave, não instantânea, para o utilizador perceber que o formulário reagiu à escolha dele.
- Secções dentro do modal devem manter espaçamento vertical consistente entre grupos de campos (já implementado via `gap` em vez de margens soltas) — é a abordagem correta, evita o erro clássico de margens que colapsam ou duplicam.

**Upload de comprovativo**
- Dropzone vazia: contorno tracejado, ícone + texto de instrução — já implementado, manter.
- Ficheiro carregado: substituir o conteúdo da dropzone por confirmação visual clara (nome do ficheiro + ícone de sucesso), nunca sobrepor à instrução original.
- Processamento: indicador de progresso + texto do que está a acontecer ("A ler o comprovativo…") — já implementado, é o padrão certo para qualquer operação assíncrona futura no sistema (não só OCR).
- Erro: mensagem específica do que falhou (ver casos de erro já definidos em `docs/arquitetura-ocr-ia.md`) + ação de recuperação visível (tentar novamente / preencher manualmente) — nunca um erro genérico sem saída.

### 3.4 Ícones e linguagem visual

**Direção: outline, simples, funcional, discreto.**

Especificação formal (corrige o problema de "ícones desenhados a olho" identificado no diagnóstico):

| Propriedade | Valor fixo |
|---|---|
| Grelha (viewBox) | 24×24, sempre |
| Espessura de traço (`stroke-width`) | 1.7–1.8, sem variação entre ícones |
| Preenchimento | Nenhum (`fill="none"`), exceto o ponto de estado de uma pill e pequenos indicadores de confirmação |
| Cantos | `stroke-linecap="round"` / `stroke-linejoin="round"` consistentes |
| Tamanho de render preferencial | 14–18px em contexto de tabela/botão, 20–22px em navegação/dropzone |

À medida que o número de ícones crescer além de uma dúzia, considerar migrar para uma biblioteca outline consistente (ex. espírito Lucide/Phosphor, mesma linguagem já usada) em vez de continuar a desenhar cada um à mão — a especificação acima é precisamente o que essa biblioteca teria de respeitar para encaixar sem se notar a troca.

Ícones nunca substituem uma label em ações importantes quando o significado não for evidente por si só — só são aceitáveis sozinhos (com `title`/tooltip) em ações secundárias e já familiares ao utilizador, como editar/remover uma linha de tabela.

Ícones devem **sempre acompanhar texto** em ações (nunca um ícone sozinho como único indicador de uma ação destrutiva ou irreversível) — já é o padrão nos botões de editar/remover linha de tabela, mas nesses casos específicos o ícone sozinho com `title`/tooltip é aceitável por serem ações secundárias e reversíveis (edição) ou com confirmação (remoção).

### 3.5 Espaçamento e layout

Escala formal em 8 níveis (refinamento aprovado — substitui a escala anterior de 6 níveis por uma mais completa e com uso por contexto explícito):

| Token | Valor | Uso recomendado |
|---|---|---|
| `spacing.1` | 4px | espaço entre ícone e texto, ajustes finos |
| `spacing.2` | 8px | espaço entre label e input |
| `spacing.3` | 12px | padding interno de inputs/botões, gap entre campos relacionados na mesma linha |
| `spacing.4` | 16px | padding interno de cartões/painéis, gutter lateral mínimo (mobile) |
| `spacing.5` | 24px | padding de modais, espaço entre grupos de campos dentro de um formulário |
| `spacing.6` | 32px | espaço entre secções/painéis empilhados numa página |
| `spacing.7` | 40px | gutter lateral de página em desktop, respiro de blocos maiores |
| `spacing.8` | 48px | separação entre blocos totalmente independentes (raramente necessário) |

O objetivo é consistência, não uniformidade forçada — nem todo espaço precisa do mesmo token, mas todo espaço deve **vir** de um destes 8 valores, nunca de um número arbitrário (ex. 18px, 27px) escolhido "a olho" durante a implementação.

Layout: densidade moderada-alta (ver secções 2 e 3.10) — largura máxima de conteúdo (~1180px, já implementado) para não esticar tabelas/formulários em ecrãs largos; formulários em modal nunca ultrapassam ~520px de largura, para manter linhas de leitura curtas nos labels e inputs.

### 3.6 Border radius

Poucos níveis, para a aplicação parecer sempre o mesmo produto:

| Token | Valor | Uso |
|---|---|---|
| `radius.small` | 8px | Inputs, botões, controlos pequenos (ícones de ação de tabela) |
| `radius.medium` | 12px | Cartões, painéis, stat tiles |
| `radius.large` | 16px | Modais |
| `radius.pill` | 999px | Pills de estado, badges, tabs em formato cápsula |

**Ajuste face à sugestão inicial**: o pedido original mapeava "botões" para o nível médio — mantive **botões no nível `small`, junto com inputs**, porque é isso que já está implementado no protótipo e visualmente correto: um botão ao lado de um input (ex. campo de pesquisa + botão) já partilha o mesmo raio hoje, e separá-los introduziria uma inconsistência visível onde hoje não há nenhuma. `radius.pill` foi acrescentado como 4º nível — não é um raio arbitrário extra, é o caso já existente (999px) de pills/badges, que continua a precisar de um valor próprio por ser conceptualmente diferente (arredondamento total, não "canto suavizado").

### 3.7 Sombras

Produto empresarial → sombra é exceção, não padrão. Um único token:

| Token | Valor conceptual | Uso |
|---|---|---|
| `shadow.overlay` | elevação forte | **Apenas** modais, menus/dropdowns flutuantes, popovers — elementos que estão de facto sobrepostos ao resto da página |

Cartões, painéis e stat tiles **não devem depender de sombra** para se destacarem — usar borda (`--border`) e contraste de superfície (`--surface` vs `--surface-2` vs `--bg`) para criar hierarquia espacial. Isto é um ajuste face à implementação atual do protótipo (que hoje aplica `--shadow-sm` a cartões/painéis "de repouso", não só a elementos flutuantes) — fica registado aqui como algo a corrigir na fase de implementação, não uma alteração feita agora.

### 3.8 Acessibilidade

- **Contraste**: texto de corpo e labels devem procurar cumprir WCAG AA (mínimo 4.5:1 para texto normal, 3:1 para texto grande/ícones). O token `--muted-2` (texto terciário) é o mais arriscado da paleta atual por ser propositadamente mais claro — deve ser **validado com uma ferramenta de contraste na fase de implementação**, e nunca usado para texto que o utilizador precise de ler com confiança (ex. nunca num valor monetário ou num estado), só para placeholders/hints verdadeiramente secundários.
- **Nunca comunicar só por cor**: sucesso, aviso, erro e estado de despesa combinam sempre cor + texto, e ícone quando fizer sentido — ex. "✓ Aceite", não uma pill verde sem palavra. Isto já é o comportamento atual do protótipo (pills sempre com texto); esta secção torna-o um requisito formal, não uma coincidência de implementação.
- **Focus visível em todos os elementos interativos**: botões, inputs, selects, links, tabs (`year-tab`, `month-tab`) e qualquer elemento clicável de tabela — hoje o foco só está formalmente tratado em inputs; passa a ser requisito explícito estendê-lo a tabs e a qualquer linha/célula de tabela que se torne clicável no futuro.

### 3.9 Densidade

A densidade já implementada no protótipo (paddings de ~12–18px em cartões, altura de linha de tabela compacta, `body` a 14px) é o **alvo a preservar**, não um ponto de partida para "arejar" no futuro. Modernidade neste produto vem de tipografia cuidada, paleta consistente e hierarquia clara — não de mais espaço vazio. Isto aplica-se com especial atenção a tabelas, formulários, pesquisa e histórico, que são as áreas onde o técnico/contabilidade passa mais tempo e onde perder densidade custa velocidade operacional real.

### 3.10 Estados de interação

| Estado | Regra |
|---|---|
| Hover | Mudança subtil de fundo (`--surface-2`) ou cor de texto — nunca movimento/escala, só cor |
| Focus | Sempre visível (anel `--accent-soft` + borda `--accent`) — nunca remover o outline sem substituir por alternativa igualmente visível, por acessibilidade |
| Selected/Active | Fundo `--accent-soft` + texto `--accent` + peso de fonte mais forte (já é o padrão da navegação lateral) |
| Disabled | Opacidade reduzida + cursor `not-allowed`, nunca esconder o elemento |
| Loading | Indicador local ao componente (spinner no botão/dropzone), nunca bloquear a página inteira sem explicação |
| Sucesso | Toast discreto (já implementado) + ícone de confirmação — desaparece sozinho, não exige ação do utilizador para fechar |
| Erro | Nunca só um toast que desaparece sozinho — erros que bloqueiam uma ação devem ficar visíveis junto ao campo/ação até serem resolvidos |

Respeitar sempre `prefers-reduced-motion` (já implementado no spinner da dropzone) — regra a aplicar a qualquer animação futura no sistema.

---

## 4. Regras de utilização

1. **Existe uma única ação visualmente dominante dentro de cada contexto principal de decisão** — não uma única ação dominante por ecrã inteiro *(refinamento aprovado)*. Um cabeçalho de página pode ter a sua própria ação primária (ex. "Nova despesa") e um modal aberto a partir dele pode ter a sua própria ação primária (ex. "Submeter despesa") sem conflito — são contextos de decisão diferentes. O que continua proibido é mais do que uma ação com o mesmo peso visual **dentro do mesmo grupo de ações** (ex. dois botões cheios lado a lado a competir pela mesma decisão). Separadamente, cor cheia de destaque em cartões continua limitada a 1 por contexto de leitura principal (ver secção 3.3).
2. **Cor de estado nunca substitui texto** — toda a pill/badge tem palavra, a cor reforça, não substitui.
3. **Neutro é o padrão; cor é exceção deliberada.** Antes de colorir um elemento novo, perguntar "isto é uma ação, uma identidade de marca, ou um estado?" — se a resposta for nenhuma das três, fica neutro.
4. **Tipografia: 3 pesos por ecrã, não mais.**
5. **Tabular nums sempre que houver colunas de números.**
6. **Todo o componente interativo tem os 5 estados mínimos**: normal, hover, focus, disabled, e (quando aplicável) erro/loading — não implementar um sem pensar nos outros.
7. **Nenhuma nova cor de estado sem primeiro tentar mapear para uma das 4 famílias existentes** (sucesso/aviso/erro/neutro).
8. **Todo o modal segue a estrutura cabeçalho fixo / corpo com scroll / rodapé fixo com ações**, já estabelecida no modal de despesa.
9. **Nenhum valor de cor, espaçamento ou raio "à mão"** — todo o CSS novo referencia um token da secção 9 (Referência de tokens), nunca um hex ou pixel arbitrário escrito diretamente.
10. **Sombra é exclusiva de elementos flutuantes** (`shadow.overlay`) — cartões e painéis usam borda/contraste de superfície, nunca sombra, para parecerem "em repouso" e não "flutuantes" sem motivo.
11. **Todo estado (sucesso/aviso/erro/informativo) combina cor + texto**, nunca cor sozinha — e todo elemento interativo tem foco visível, sem exceção.

---

## 5. Aplicação prática nas telas existentes

**Saldo e registos**
- Já aplica corretamente a regra do cartão de destaque único ("Total provisório" com cor cheia, "Total aceite" neutro) — manter.
- Reforçar a tabela de movimentos com o padrão de estado vazio da secção 3.3 quando não houver despesas no período.

**Tabela de movimentos**
- Já segue o padrão de cabeçalho `--surface-2` + hover discreto — manter tal como está.
- Padronizar a ordem visual das pills de estado (ex. sempre a mesma posição de coluna) à medida que mais tipos de despesa/estado forem adicionados.

**Modal "Nova despesa"**
- Já segue a estrutura recomendada. Ponto a reforçar: os campos condicionais de combustível/portagens (matrícula, km, tipo de combustível) devem manter uma transição visual suave ao aparecer/desaparecer (hoje é instantâneo via `hidden`) — pequena melhoria de polimento, não estrutural.

**Upload de comprovativo**
- Já implementa os 3 estados (vazio/processamento/concluído) corretamente. Falta o estado de **erro** visual (ver `docs/arquitetura-ocr-ia.md`, secção de tratamento de erros) — hoje a simulação nunca falha; quando a integração real chegar, a dropzone precisa de um 4º estado visual para falha de leitura, seguindo a mesma linguagem (ícone + texto + ação de recuperação).

**Estados das despesas**
- Já usa a família semântica correta (`ok`/`pending`/`bad`). Reforçar como regra formal (secção 3.1) para não se perder quando novos estados forem adicionados (ex. "Pago pela empresa", "Reembolsado ao técnico").

---

## 6. Melhorias prioritárias

**Alta prioridade** (afeta credibilidade da proposta à empresa)
- Aplicar consistentemente a regra refinada de ação dominante por contexto (secção 4.1) e o limite de 1 cartão com cor cheia por contexto de leitura, revendo Férias/Faltas/Presenças com o mesmo critério já usado em Despesas.
- Adicionar o estado de erro à dropzone de comprovativo (hoje inexistente visualmente).
- Mapear os estados de Férias/Faltas/Presenças para as famílias semânticas formalizadas na secção 9, confirmando que usam exatamente a mesma lógica de cor+texto que Despesas.
- Corrigir o uso de sombra em cartões/painéis "de repouso" para a nova regra (secção 3.7) — sombra só em elementos flutuantes.

**Média prioridade** (polimento visível mas não bloqueante)
- Transições suaves nos campos condicionais do formulário de despesa.
- Migrar os ícones para uma grelha/peso de traço formalmente verificado (hoje é consistente "a olho", não por regra).
- Rever espaçamento de todos os ecrãs contra a escala de 4px formalizada na secção 3.5, corrigindo pequenos desvios.

**Opcional** (não crítico para a proposta à empresa)
- Considerar uma biblioteca de ícones externa em vez de SVGs à mão, só quando o número de ícones justificar o esforço de migração.
- Explorar uma segunda cor de acento mais distinta para diferenciar visualmente áreas muito diferentes do sistema (ex. Despesas vs. Gestão de Pessoal) — atualmente ambas partilham o mesmo `--accent`, o que é aceitável mas não obrigatório manter assim.

---

## 7. Ordem recomendada de implementação

1. Formalizar as regras da secção 4 num ficheiro de referência rápida (tokens + regras, sem repetir toda a estratégia) — para consulta durante a implementação.
2. Rever os ecrãs já existentes (Férias, Faltas, Presenças, Documentos) contra as regras de estado/cor da secção 3.1 e corrigir divergências.
3. Adicionar o estado de erro à dropzone de comprovativo.
4. Adicionar as transições suaves aos campos condicionais do formulário de despesa.
5. Só depois destes quatro passos — que corrigem consistência no que já existe — considerar qualquer extensão visual nova (mais tipos de cartão, mais estados, ícones adicionais).

A lógica desta ordem: a proposta à empresa ganha mais credibilidade por **coerência entre todos os ecrãs existentes** do que por adicionar um ecrã novo bonito enquanto os outros ficam ligeiramente diferentes entre si.

---

## 8. Referência de tokens

Dicionário conceptual — nome do token a usar na implementação, ligado ao valor já existente no CSS do protótipo (custom property equivalente) quando já existe, ou assinalado como **novo** quando o refinamento o introduziu.

| Token conceptual | CSS custom property atual | Valor (claro) | Valor (escuro) |
|---|---|---|---|
| `color.brand.primary` | `--accent` | `#0e7c86` | `#43bcb4` |
| `color.brand.primary-hover` | `--accent-hover` | `#0b6169` | `#5bcac2` |
| `color.brand.on-primary` | `--accent-ink` | `#ffffff` | `#062522` |
| `color.brand.primary-soft` | `--accent-soft` | `#e2f3f2` | `rgba(67,188,180,.15)` |
| `color.brand.secondary` | `--accent-2` | `#2f5f8a` | `#7fa8d4` |
| `color.surface.page` | `--bg` | `#f4f6f8` | `#0d151c` |
| `color.surface.default` | `--surface` | `#ffffff` | `#131e27` |
| `color.surface.subtle` | `--surface-2` | `#eef1f5` | `#182430` |
| `color.border.default` | `--border` | `#dde3ea` | `#243342` |
| `color.text.primary` | `--ink` | `#151f2b` | `#e8edf2` |
| `color.text.secondary` | `--muted` | `#5c6b7a` | `#96a5b3` |
| `color.text.tertiary` | `--muted-2` | `#8492a1` | `#6f7f8f` |
| `color.status.success` / `.success-soft` | `--success` / `--success-soft` | `#1c8a5a` / `#e3f6ec` | `#42d190` / `rgba(66,209,144,.14)` |
| `color.status.warning` / `.warning-soft` | `--warning` / `--warning-soft` | `#b6790a` / `#fbf0dc` | `#e3ab4a` / `rgba(227,171,74,.14)` |
| `color.status.danger` / `.danger-soft` | `--danger` / `--danger-soft` | `#c53c3c` / `#fbe7e7` | `#e57972` / `rgba(229,121,114,.14)` |
| `color.status.info` / `.info-soft` | **novo** — não existe ainda no protótipo | `#2f6feb` (proposta) / `#e8eefd` | `#7ea6f5` (proposta) / `rgba(126,166,245,.14)` |
| `spacing.1`…`spacing.8` | não tokenizado no CSS atual (valores soltos) | `4/8/12/16/24/32/40/48px` | igual |
| `radius.small` | `--radius-sm` | `8px` | igual |
| `radius.medium` | `--radius-md` | `12px` | igual |
| `radius.large` | `--radius-lg` | `16px` | igual |
| `radius.pill` | `999px` (hoje hardcoded em `.pill`) | `999px` | igual |
| `shadow.overlay` | `--shadow-md` | — | — |

**Porque `color.status.info` é novo**: o pedido do utilizador incluía explicitamente este token, mas o protótipo atual só tem sucesso/aviso/erro — nunca precisou de "informativo" porque nenhum estado do sistema até agora era neutro-informativo (ex. um aviso que não é erro nem sucesso, só contexto). Proponho um azul distinto do `color.brand.primary` (teal) e do `color.brand.secondary` (slate) para não colidir com nenhum dos dois — a usar, por exemplo, em mensagens de ajuda contextual ou no futuro estado "Documento em processamento" do OCR (`docs/arquitetura-ocr-ia.md`) que não é nem sucesso nem erro. **Valor por confirmar na implementação**, aqui é só a reserva conceptual do token.

`shadow.overlay` fica sem par claro/escuro na tabela porque continua a ser a mesma sombra em ambos os temas (só a opacidade/blur muda, já tratado no CSS atual via `--shadow-md`).

## 9. Estados semânticos formalizados

**Famílias semânticas** (a base para qualquer estado, em qualquer módulo, presente ou futuro):

| Família | Token de cor | Ícone típico | Regra de utilização |
|---|---|---|---|
| Positivo | `color.status.success` | ✓ | Algo foi aprovado, aceite ou confirmado — ação concluída a favor do utilizador |
| Pendente | `color.status.warning` | ● ou relógio | Algo aguarda decisão/ação de terceiros — não é um erro, é um estado transitório |
| Negativo | `color.status.danger` | ✕ | Algo foi recusado, rejeitado ou falhou |
| Neutro | `color.text.secondary` sobre `color.surface.subtle` | — | Estado informativo sem carga positiva/negativa (ex. "Arquivado") |
| Informativo | `color.status.info` *(novo)* | ⓘ | Contexto/ajuda que não é estado de aprovação nem erro (ex. processamento em curso) |

**Mapeamento por módulo — mesma lógica em todos, nomes de estado diferentes:**

| Módulo | Estado no sistema | Família semântica | Label visível |
|---|---|---|---|
| Férias | Aprovado | Positivo | "Aprovado" |
| Férias | Em análise | Pendente | "Em análise" |
| Férias | Recusado | Negativo | "Recusado" |
| Faltas | Aprovado | Positivo | "Aprovado" |
| Faltas | Em análise | Pendente | "Em análise" |
| Faltas | Recusado | Negativo | "Recusado" |
| Despesas | Aceite | Positivo | "Aceite" |
| Despesas | Em análise | Pendente | "Em análise" |
| Despesas | Rejeitado | Negativo | "Rejeitado" |
| Presenças | Presença | Positivo | "Presença" |
| Presenças | Falta | Negativo | "Falta" |

**Regra explícita**: a mesma cor nunca pode ter significados contraditórios entre módulos — "verde" é sempre e só "Positivo" em qualquer ecrã do sistema, nunca usado para outra coisa (ex. nunca usar verde para indicar "categoria Combustível" ou qualquer classificação não relacionada com aprovação/estado). Qualquer estado novo que apareça no futuro (ex. "Pago", "Reembolsado", "Cancelado") tem de ser mapeado para uma das 5 famílias acima antes de se decidir a sua cor — nunca ganha uma cor nova só porque "parece diferente o suficiente".

---

## 10. Próxima ação concreta

**Rever o ecrã "Férias" e "Faltas" contra as regras de cor/estado da secção 9**, comparando lado a lado com "Saldo e registos" — é o primeiro passo da ordem de implementação (secção 7) e o que tem maior impacto imediato na sensação de "produto único e coerente" ao apresentar o sistema à empresa. A especificação (secções 3–10) está agora fechada; este passo só deve ser implementado em código depois de confirmação explícita para avançar.
