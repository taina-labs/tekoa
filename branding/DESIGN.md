# Sistema de Design Tainá

- **Status:** Vivo (aplicável já)
- **Escopo:** sistema de design do ecossistema Tainá: o produto (superapp / PWA
  server-rendered em Phoenix LiveView), o site/landing e a documentação (este
  repositório, Tekoá).
- **Idioma:** pt-BR primeiro. Nomes de marca em Tupi-Guarani, vocabulário
  social em pt-BR familiar, encanamento em inglês (ver seção 10).
- **Fonte de verdade:** o Penpot (**"Cofre da Comunidade - UI v1"**), espelhado
  1:1 em `taina/assets/css/tokens.css`. Este documento é a **leitura legível**
  desse sistema, não um sistema novo.

Este documento complementa o `briefing.md` (identidade visual e narrativa) e o
`tom-de-voz.md` (voz do produto). Onde ele cruza com os dois, ele resume e
aponta, não substitui.

---

## 0. Escopo e precedência

A ordem de autoridade é fixa. Em qualquer conflito, o de cima vence:

1. **Penpot** ("Cofre da Comunidade - UI v1"): a fonte de verdade do desenho.
2. **`taina/assets/css/tokens.css`**, `base.css`, `layouts.css`,
   `components.css`: o espelho 1:1 do Penpot no código.
3. **Este `DESIGN.md`**: a especificação legível de intenção, semântica e regras
   de comportamento.
4. **`briefing.md` / `tom-de-voz.md` / RFCs**: a narrativa e a lei do produto
   (zonas, papéis, transparência, privacidade honesta).

**O código é o desenho.** O `taina/AGENTS.md` diz isso literalmente: a interface
já existe, distribuída em 16 LiveViews, e a tarefa de quem constrói tela é
**executar um desenho já cometido, não inventar um**. Este documento descreve
esse desenho para que uma pessoa (ou agente) que chega pelo GitHub entenda o
sistema sem precisar ler 2.500 linhas de CSS.

Regras que decorrem disso:

- Não se cria um sistema paralelo. Um valor novo **nasce como token semântico**
  no Penpot primeiro, e depois no `tokens.css`.
- Componentes consomem **token semântico**, nunca primitivo, nunca hex ou px
  cru em `.heex` ou CSS de componente.
- Nomes de classe são **BEM** para componente (`bloco__elemento--modificador`,
  ex. `photo-grid__item`) e **utilitários** para layout e espaçamento (`row
  between`, `col gap-5`, `type-*`).

---

## 1. Atmosfera

O Tainá é a **caixa de uma comunidade que decidiu cuidar dos próprios dados**.
A interface é o **cofre noturno**: céu profundo, silêncio, e a estrela da manhã
como única fonte de cor quente. Não é um app barulhento de engajamento; é um
lugar calmo para guardar e cuidar.

Três dialetos definem o tom. Servem para calibrar decisões, não como métrica:

- **Densidade: 4 de 10** (app do dia a dia). Respiro entre blocos, sem vazios
  caros de galeria de arte, sem cockpit apertado. A exceção é a linha de tempo
  de fotos e o chat futuro, onde a densidade sobe por natureza do conteúdo.
- **Variância: 4 de 10** (simétrico com deslocamentos leves). Listas e grades
  organizadas. Assimetria só quando o conteúdo pede (painel, landing). Nada de
  caos artístico.
- **Movimento: 3 de 10** (estático contido). CSS-first, um momento de impacto
  por navegação (`page-in`), o resto em estados de `hover`/`active`. Design for
  less usage, not more (ver `tom-de-voz.md` e `RFC_ARQUITETURA.md`).

Palavras para manter na cabeça ao desenhar: **noturno, ancestral, técnico,
acolhedor, brasileiro sem caricatura, simples sem ser minimalista de vitrine.**

---

## 2. Identidade

### A marca

- Símbolo: **estrela estilizada** (a "estrela da manhã"), geometria tecnológica
  combinada com formas orgânicas, "ascensão" como metáfora do amanhecer, e uma
  **rede de nós conectados** que representa descentralização e comunidade.
- Variações necessárias (herdadas do `briefing.md`): logo completo (símbolo +
  tipografia), símbolo isolado, monocromática, para fundo claro e escuro,
  favicon e ícone de app.
- No código, a estrela aparece como o elemento `.spark`, sempre em
  `--brand-primary` (o âmbar do amanhecer). É o único lugar onde o brilho é
  permitido, e mesmo ele é um traço de cor, não um glow.

### Guardrails culturais (inegociáveis)

Do `briefing.md`, repetidos aqui porque são a parte que mais se erra:

- **Nada de clichê** (futebol, carnaval), nada de tribalismo literal, nada de
  apropriação cultural, nada de cores saturadas demais, nada de complexidade
  visual gratuita, nada de densidade oriental sem contexto.
- Brasilidade **sem estereótipo**; tecnologia **sem intimidação**; técnica mas
  **convidativa**.
- Arte indígena entra como **geometria, simbologia e conexão ancestral**, não
  como figurino.

---

## 3. Cor

Dois níveis, como no Penpot. **Primitivos** são a paleta bruta; **semânticos**
são o vocabulário que os componentes consomem. Componentes usam só o semântico.

### 3.1 Primitivos

| Família      | Passos                                                                                          | Papel                                    |
| ------------ | ----------------------------------------------------------------------------------------------- | ---------------------------------------- |
| `night`      | 950 `#060a14`, 900 `#0a101f`, 800 `#0f1729`, 700 `#15203a`, 600 `#1d2c4e`, 500 `#28395f`, 400 `#3a4d7a`, 300 `#56689b`, 200 `#8c9ac0`, 100 `#bfc8df`, 50 `#eef1f8` | Base neutra fria do céu noturno          |
| `dawn`       | 300 `#ffd9a3`, 400 `#fcbe74`, 500 `#f2a65a`, 600 `#dd8a3e`, 700 `#b96d2c`                        | Marca (amanhecer)                        |
| `ember`      | 400 `#f08a6c`, 500 `#e2674a`, 600 `#c44e33`                                                      | Calor secundário, horizonte de auth      |
| `star`       | 50 `#faf7f0`, 100 `#f1ecdf`, 200 `#e4dcc8`                                                       | Branco quente das estrelas (texto)       |
| `forest`     | 300 `#82cba4`, 400 `#5fb88a`, 500 `#3fa171`, 600 `#2f8159`, 700 `#246244`, 900 `#122e22`         | Acento de serviço: **Ybira** (arquivos)  |
| `moon`       | 300 `#f4dfa6`, 400 `#eccd80`, 500 `#e3bc66`, 600 `#c49a48`, 700 `#9c7836`, 900 `#46350f`         | Acento de serviço: **Jaci** (fotos)      |
| `sky`        | 300 `#92c3f8`, 500 `#5ba3f5`, 600 `#3f84d6`, 700 `#2f66ac`                                       | Acento de serviço: **Nhaman** (painel)   |
| `guara`      | 500 `#de5b3f`, 600 `#be4327`                                                                     | Acento de serviço: **Guará** (pós-MVP)   |
| funcionais   | `red-500` `#e5484d`, `red-600` `#c03538`, `amber-500` `#efb54a`                                   | Erro, atenção                             |

### 3.2 Semânticos (o contrato de consumo)

| Token                     | Valor                          | Uso                                              |
| ------------------------- | ------------------------------ | ------------------------------------------------ |
| `--bg-app`                | `#0a101f` (night-900)          | Fundo padrão do app                              |
| `--bg-deep`               | `#060a14` (night-950)          | Fundo mais profundo (nav, layout de auth)        |
| `--surface-default`       | `#0f1729` (night-800)          | Superfície de base (cards, linhas)               |
| `--surface-raised`        | `#15203a` (night-700)          | Superfície elevada (item ativo, card `--raised`) |
| `--surface-high`          | `#1d2c4e` (night-600)          | Superfície de maior contraste                    |
| `--surface-overlay`       | `rgb(10 16 31 / 0.88)`         | Sobreposições (modal, menu)                      |
| `--border-subtle`         | `#15203a` (night-700)          | Linha estrutural de 1px                          |
| `--border-default`        | `#1d2c4e` (night-600)          | Borda de campo e card                            |
| `--border-strong`         | `#28395f` (night-500)          | Borda em destaque                                |
| `--text-primary`          | `#faf7f0` (star-50)            | Texto principal                                  |
| `--text-secondary`        | `#bfc8df` (night-100)          | Texto secundário                                 |
| `--text-muted`            | `#8c9ac0` (night-200)          | Texto de apoio, metadados                        |
| `--text-faint`            | `#56689b` (night-300)          | Texto mais fraco, placeholder                    |
| `--text-on-brand`         | `#060a14` (night-950)          | Texto sobre a cor de marca                       |
| `--brand-primary`         | `#f2a65a` (dawn-500)           | Ação primária, foco, estado ativo                |
| `--brand-primary-hover`   | `#fcbe74` (dawn-400)           | Marca no `hover`                                 |
| `--brand-primary-pressed` | `#dd8a3e` (dawn-600)           | Marca no `pressed`                               |
| `--brand-ember`           | `#e2674a` (ember-500)          | Calor secundário                                 |
| `--service-ybira`         | `#3fa171` (forest-500)         | Acento de serviço (arquivos)                     |
| `--service-ybira-soft`    | `#122e22` (forest-900)         | Fundo suave do serviço                           |
| `--service-jaci`          | `#e3bc66` (moon-500)           | Acento de serviço (fotos)                        |
| `--service-jaci-soft`     | `#46350f` (moon-900)           | Fundo suave do serviço                           |
| `--service-nhaman`        | `#5ba3f5` (sky-500)            | Acento de serviço (painel)                       |
| `--service-guara`         | `#de5b3f` (guara-500)          | Acento de serviço (chat, pós-MVP)                |
| `--feedback-success`      | `#5fb88a` (forest-400)         | Sucesso                                          |
| `--feedback-warning`      | `#efb54a` (amber-500)          | Atenção (ambiente, nunca alarme)                 |
| `--feedback-error`        | `#e5484d` (red-500)            | Erro                                             |
| `--feedback-info`         | `#5ba3f5` (sky-500)            | Informação                                       |

### 3.3 Regras

- **Um acento de marca, mais acentos de serviço.** A regra geral de "um único
  acento" vale para a marca: `--brand-primary` é o âmbar do amanhecer e só ele
  pinta ação primária, foco e estado ativo. A **exceção deliberada** são os
  acentos por serviço (`--service-*`), um por subsistema, para dar identidade
  própria a Ybira, Jaci e Nhaman dentro de uma casca comum. Acento de serviço
  **não** vira cor de ação; ele marca pertencimento.
- **Sem preto puro.** O mais escuro é `--bg-deep` (`#060a14`), um "quase preto"
  noturno, nunca `#000000`.
- **Dark-first, tema único.** Hoje o produto tem **um** tema (escuro): o
  `base.css` fixa `color-scheme: dark`. Não existe modo claro implementado, e
  este documento não finge que existe. Um tema claro é decisão futura, não um
  "faltou".
- **Cor nunca sozinha.** Todo estado carrega também um símbolo ou uma frase
  (ver seção 9). Um aviso é `--feedback-warning` **e** um ícone **e** uma
  frase.
- **Sem glow, sem gradiente decorativo.** O único gradiente sancionado é o
  horizonte do `auth-layout` (radial em `ember`/`dawn`), que evoca o
  amanhecer. Fora dele, gradiente é suspeito.

---

## 4. Tipografia

Três famílias, auto-hospedadas em `/fonts` como variáveis (`woff2-variations`).
Nenhuma requisição externa.

- **Display:** `Bricolage Grotesque` (eixo `wght` 200 a 800). Títulos e o
  logotipo tipográfico. Personagem geométrica com calor, nada de genérico.
- **Corpo:** `Schibsted Grotesk` (`wght` 400 a 900). Texto de interface e
  leitura.
- **Mono:** `JetBrains Mono` (`wght` 100 a 800). Código, metadados e números em
  contexto de dados.

**Proibido:** Inter, Roboto, Arial, fontes de sistema como escolha de marca,
serifas genéricas em UI. A característica é o cofre Tupi-Guarani escuro; não se
troca a fonte.

### Escala e classes

| Classe          | Família  | Tamanho                  | Peso | Detalhe                          |
| --------------- | -------- | ------------------------ | ---- | -------------------------------- |
| `type-display`  | display  | `2.75rem` (`--font-size-display`) | 700 | `tracking -0.5px`, `line-height 1.05` |
| `type-h1`       | display  | `2.25rem`                | 700 | `tracking -0.5px`, `lh 1.1`      |
| `type-h2`       | display  | `1.75rem`                | 600 | `lh 1.15`                        |
| `type-h3`       | display  | `1.125rem`               | 600 | `lh 1.25`                        |
| `type-body`     | corpo    | `1rem`                   | 400 | `lh 1.5`                         |
| `type-body-sm`  | corpo    | `0.875rem`               | 400 | `lh 1.45`                        |
| `type-label`    | corpo    | `0.875rem`               | 600 | `lh 1.2`                         |
| `type-caption`  | corpo    | `0.75rem`                | 500 | `lh 1.3`                         |
| `type-overline` | corpo    | `0.75rem`                | 600 | `tracking 1.5px`, uppercase      |
| `type-mono-sm`  | mono     | `0.8125rem`              | 400 | `lh 1.4`                         |

As classes `type-*` moram em `base.css`, não em `utilities.css` (é intencional,
não é para "consertar").

### Regras

- Hierarquia por **peso e cor**, não por escala gigante. Nada de H1 gritando.
- Números de dados (cotas, tamanhos, contagens) em `type-mono-sm` ou
  `--font-family-mono`, e em pt-BR ("1,1 GB", nunca "1073741824").
- Linha de leitura com teto pela família `--measure*` (480, 560, 640, 720px).
- Acentuação pt-BR é esperada e obrigatória ("não", "casa", "Saúde"). O código
  é ASCII; o texto de tela, não.

---

## 5. Escala (espaço, raio, tamanho, elevação)

### Espaço (base 4px)

`--space-1` 4, `--space-2` 8, `--space-3` 12, `--space-4` 16, `--space-5` 20,
`--space-6` 24, `--space-8` 32, `--space-10` 40, `--space-12` 48, `--space-16` 64.

### Raio

`--radius-sm` 8, `--radius-md` 12, `--radius-lg` 16, `--radius-xl` 24,
`--radius-pill` 999. Um raio por papel: campos e botões pequenos em `sm`/`md`,
cards em `lg`, superfícies grandes em `xl`, pílulas para chips e badges em
`pill`. Não misturar sem regra.

### Tamanhos de controle

`--size-touch` 44 (alvo mínimo de toque), `--size-control-sm` 36,
`--size-control-md` 48, `--size-control-lg` 56, `--size-nav-mobile` 72,
`--size-appbar` 64, `--size-nav-dot` 6.

### Bordas, opacidade e elevação

- `--border-hairline` 1px, `--border-thick` 2px (foco).
- `--opacity-disabled` 0.4, `--opacity-muted` 0.72, `--opacity-scrim` 0.64.
- `--elevation-soft` `0 4px 16px 0 rgb(6 10 20 / 0.5)`,
  `--elevation-lifted` `0 12px 32px -4px rgb(6 10 20 / 0.65)`. A sombra é
  tingida com a cor do fundo noturno, nunca preto puro. Só eleva quando elevar
  comunica hierarquia.

---

## 6. Layout e cascas

### A casca autenticada (`.shell`)

- **Mobile (até 767px):** `bottom-nav` fixa de 72px com 3 a 6 itens. No shell
  cometido: Início, Arquivos, Fotos, Mural, Saúde, Conta. Ícone **e** rótulo
  sempre visíveis, com o rótulo truncando em reticências em telas estreitas. O
  conteúdo reserva espaço para a nav e nunca rola na horizontal.
- **Desktop (768px+):** `sidebar` persistente de 248px com ícone e nome,
  `topbar` de 64px (busca global, notificações, perfil) e conteúdo com teto de
  largura.
- **Appbar** (`.appbar`): telas internas com voltar e título centralizado; nome
  longo trunca com reticências, não empurra as ações.
- Cada navegação remonta o conteúdo e dispara o `page-in` (ver seção 8).

### Layout de auth (`.auth-layout`)

Céu noturno com horizonte de amanhecer (radial em `ember`/`dawn`) e um campo de
estrelas em dois layers de `radial-gradient`. Card único de até 420px, centrado.
Usado em setup, login e convite.

### Breakpoints

320 (base mobile), 375, 425, 768 (tablet: vira sidebar), 1024 (desktop: mais
respiro, multi-painel onde ajuda), 1440 (desktop largo: conteúdo mais largo com
teto), 2560 (4K: o conteúdo **para no teto**, não estica de borda a borda).

### Regras de layout

- **Sem scroll horizontal da página, nunca**, em nenhum viewport.
- **Multipainel só onde ajuda de verdade.** Ybira (lista + detalhe/preview) e
  Jaci (grade larga + painel do visualizador) ganham; Maraca, resumos e listas
  curtas não. Resistir ao multipainel por padrão.
- **Tabelas viram linhas empilhadas** no mobile, nunca tabela espremida.
- **Teto de leitura** pelas `--measure*`; no 4K o espaço extra é respiro, não
  fonte maior com vazio.
- Utilitários de layout: `row`, `col`, `between`, `center`, `grid`, `gap-*`,
  `mt-*`/`mb-*`, `measure*`, `truncate`, `w-full`, `surface-*`, `text-*`.
- Componentes nunca trazem hex ou px cru; sempre token semântico.

---

## 7. Componentes

O inventário real vive em `components.css` e `core_components.ex`. **Reusar, não
reconstruir.** Visão de conjunto, por família:

- **Ações:** `.btn` (`--primary`, `--secondary`, `--ghost`, `--danger`,
  `--service`, `--sm`, `--md`, `--fab`), `.icon-btn` (`--danger`). Feedback
  tátil no `active`. Um CTA primário por tela.
- **Conteúdo:** `.card` (`--raised`), `.cards-grid`, `.list`, `.list-row`
  (`__title`, `__meta`, `__actions`), `.tiles` / `.tile`, `.painel-grid`,
  `.files-split__*` (lista + detalhe + preview de arquivos).
- **Mídia:** `.photo-grid` (`__item`, `__tag`, `__video`) com scroll infinito,
  `.media-frame`, `.doc-frame`, `.text-preview`, `.viewer` (`__stage`, `__rail`,
  `__nav`, `__actions`, `__caption`) para fullscreen com swipe e teclado.
- **Serviço e estado:** `.service-square` (`--ybira`, `--jaci`, `--nhaman`,
  `--guara`, `--doc`, `--video`, `--neutral`, `--lg`), `.service-card`,
  `.avatar` (`--sm`, `--off`, variantes de cor), `.badge`
  (`--active`, `--inactive`, `--soon`, `--zelador`, `--ybira`, `--jaci`),
  `.chip` (`--selected`), `.legend-dot--*` (documentos, fotos, vídeos, outros).
- **Formulário:** `.field` (`__control`, `__help`, `__error`, `--invalid`),
  `.radio-card` (`--selected`, `__ring`), `.segmented` (`__option`),
  `.search`, `.dropzone` (com `.is-drop`).
- **Sobreposição:** `.modal`, `.modal-overlay` (backdrop com blur), `.menu`
  (`__list`, `__item`, `--danger`).
- **Progresso e aviso:** `.progress` (`__fill` com `--ybira`, `--jaci`, `--sky`,
  `--success`, `--neutral`), `.flash-group` / `.flash` (`--error`, `--info`).
- **Estrutura:** `.breadcrumb` (`__sep`, `__current`), `.steps` (`__dot`,
  `--current`), `.qr-card` (convite), `.empty-state`, `.nav-dot`.

### Estados obrigatórios

**Toda tela entrega estado vazio, de erro e de permissão**, não só o caminho
feliz. Use `.empty-state` para isso.

- **Vazio:** convida e ensina o modelo do produto. Exemplo: "Ninguém acessa
  seus arquivos sem você deixar. Quando alguém pedir, aparece aqui." Nunca só
  "Nada aqui".
- **Carregando:** esqueleto no formato do layout final, nunca spinner circular
  genérico.
- **Erro:** inline, explica o que houve e oferece saída. Nunca culpa, nunca
  beco sem saída, nunca stack trace nem exit status.
- **Permissão:** fala como cuidado, não como recusa. "Quem cuida da máquina
  cuida do backup."

---

## 8. Movimento

CSS-first e contido. **Um momento de alto impacto por navegação vale mais que
micro-animações espalhadas.**

- Transição de página: `page-in`, `opacity 0 -> 1` e `translateY(8px) -> 0` em
  `220ms ease`, disparado na remontagem do conteúdo a cada `live_redirect`.
- Estados `hover`/`active` com transição curta; `active` faz o controle "afundar"
  1px para dar tato.
- Animar apenas `transform` e `opacity`. Nunca `top`, `left`, `width`,
  `height`.
- **`prefers-reduced-motion: reduce` desliga tudo**: a animação de página, os
  loops e as revelações viram estático. Isso é obrigatório, não polimento.
- Sem loop infinito decorativo, sem parallax, sem scroll-hijack. Movimento
  responde a uma ação da pessoa; ele não pede atenção sozinho.

---

## 9. Acessibilidade

A acessibilidade é **tom**, não acabamento (ver `tom-de-voz.md`), e faz parte
do pilar de transparência (ver `RFC_003`, seção 6).

- **WCAG AAA** como alvo: contraste **7:1** para texto e **3:1** para elementos
  de interface.
- **Alvo de toque** de 44px (`--size-touch`) em mobile. Nada só no `hover`: se
  funciona no `hover`, tem que funcionar no toque.
- **Foco visível**: `outline` de 2px (`--border-thick`) em `--brand-primary`,
  `offset 2px`. O foco não pode ser cortado por `overflow: hidden`.
- **Ícone sempre com rótulo** (visível ou `sr-only`); botão de ícone carrega
  `label`; imagem carrega `alt`.
- **Navegação por teclado e por swipe** onde importa (o padrão `ViewerNav`):
  setas e gesto funcionam no visualizador de fotos.
- **Números em pt-BR** e unidades humanas ("1,1 GB", "há 3 dias", "ontem").
- **Cor nunca é a única informação**: sempre acompanhada de símbolo ou frase.

---

## 10. Voz na interface

A voz é a de uma **vizinhança que se organiza**, não a de uma plataforma. O
detalhe completo está em `tom-de-voz.md`; aqui fica o que a interface precisa
respeitar.

### Convenção de nomes (uma fonte de verdade)

Três camadas, nunca misturar ao acaso (ver `RFC_003`, seção 5):

| Camada           | Quando                                              | Exemplos                                             |
| ---------------- | --------------------------------------------------- | ---------------------------------------------------- |
| Tupi-Guarani     | nomes próprios de subsistema e marca                | Tekoá, Maracá, Ybira, Jaci, Nhaman                   |
| pt-BR familiar   | vocabulário social do dia a dia (átomo = contexto = UI) | casa, praça, zelador(a), morador(a), mural, pedido, assembleia |
| Inglês           | só encanamento que a pessoa nunca lê                | Repo, Scope, Telemetry                               |

O átomo do código **é** o rótulo da tela e o termo do contexto, sem camada de
tradução. Ninguém angliciza os nomes Tupi.

### Glossário do produto

| Conceito                     | Termo (UI = código)                   | Nunca dizer                                          |
| ---------------------------- | ------------------------------------- | ---------------------------------------------------- |
| Espaço pessoal e privado     | **casa**                              | "meus arquivos privados", "área pessoal"             |
| Espaço comum da comunidade   | **praça**                             | "público", "compartilhado com todos" (só na explicação) |
| Quem cuida da máquina        | **zelador(a)**                        | "admin", "administrador", "dono"                     |
| Membro da comunidade         | **morador(a)**                        | "usuário", "membro" (em copy)                        |
| Registro do que aconteceu    | **mural**                             | "log", "feed", "atividade"                           |
| Estado da máquina, legível   | **Saúde da comunidade** (painel)      | "dashboard", "métricas"                              |
| Pedir para abrir uma casa    | **pedir acesso**                      | "solicitar permissão"                                |
| Decisão coletiva (pós-MVP)   | **assembleia**                        | "votação", "governança"                              |
| Nome de acesso da pessoa     | **nome de usuário** (`username`)      | "login", "user", "e-mail"                            |
| Nome que aparece para todos  | **nome de exibição** (`display_name`) | "apelido", "perfil"                                  |

### Regras de escrita

- Pessoa: "você" no singular acolhedor; "nós/nossa" para os comuns. Voz ativa,
  presente. Frases curtas, uma ideia por frase.
- Verbos diretos: "Pedir acesso", "Publicar na praça", "Convidar", "Enviar".
  Nunca "Submeter", "Processar", "Gerenciar".
- Escrever para a pessoa **menos técnica** da comunidade: sem jargão, sem
  conhecimento presumido, sem beco sem saída. Ensine, não apenas informe.
- **Anti-engajamento:** nada de badge vermelho de urgência, streak, métrica de
  vaidade, feed algorítmico. Avisos são ambientes e opcionais, e **somem quando
  resolvidos**. Design for less usage, not more.
- **Privacidade honesta:** a casa é **"promessa de software, não cadeado
  matemático"**. O Tainá ainda **não** criptografa; nunca prometer que o
  zelador não consegue ler a casa.
- **Poder visível:** quem cuida é zelador(a), nunca dono ou chefe. "Quem cuida
  (Ana) aumentou a cota para 50 GB." Nada de "Admin alterou as configurações".
- Erros explicam e oferecem saída; vazios convidam e podem ensinar.

### Tipografia do texto

Nada de emoji e nada de "AI-typography": sem travessões (`—`/`–`), sem pontos
médios (`·`/`•`), sem reticências tipográficas, sem aspas curvas, sem `×`, sem
setas. Escreva vírgula, dois-pontos, ponto, `...`, aspas retas, `x` e `-` ou
`->`. Acentos pt-BR são exigidos; a regra é de tipografia, não de Latin-1.

---

## 11. Anti-padrões (proibido)

Do sistema de design e do produto. Tratar como lista de reprovação.

**Tainá (produto):**

- Chat (**Guará**) no MVP. Está cortado; nunca propor chat para "ser mais
  comunitário". O valor está em arquivos, fotos, pessoas e governança.
- Badge vermelho de urgência, streak, métrica de vaidade, feed algorítmico,
  notificação por padrão. Nada de isca de atenção.
- "Admin" na interface. A palavra é **zelador(a)**.
- Prometer criptografia que não existe. A casa é promessa de software.
- Telemetria que fala com SaaS. **Nunca telefonar para casa**: métrica, log e
  telemetria ficam na caixa da comunidade.
- Múltiplas comunidades por caixa, descoberta pública ou feed entre instâncias.
  É uma Tekoa.
- Anglicizar nomes Tupi-Guarani, ou usar termo em inglês onde o vocabulário
  social pt-BR manda.
- Emoji e AI-typography no texto de tela.
- Controle que implique poder sobre o mural. O mural é append-only, read-only
  para todos, inclusive o zelador.

**Design / IA tells genéricos:**

- Fonte Inter, Roboto, Arial ou system como escolha de marca; serifa genérica
  em UI.
- Preto puro (`#000000`) e branco puro (`#ffffff`).
- Glow neon, sombra preta pura, gradiente decorativo sem função.
- Acento super-saturado, mais de um acento de marca na mesma tela.
- Cor como única informação de estado.
- Três cards idênticos lado a lado como "linha de features"; card dentro de
  card dentro de card.
- Layout esticado de borda a borda em 4K; vazio de vitrine no lugar de
  respiro.
- Scroll horizontal da página, alvo de toque menor que 44px, texto tocando a
  borda do container.
- Spinner circular genérico no lugar de esqueleto; "Nada aqui" no lugar de
  estado vazio que ensina.

---

## 12. Aplicação

### Produto (PWA superapp)

É onde este sistema nasce. Tema escuro como padrão, acento de serviço por
serviço, `bottom-nav` no mobile e `sidebar` no desktop, transições suaves entre
serviços sem reload, componentes compartilhados (preview de arquivo, upload,
notificação). Ver `RFC_002_MVP.md` para o escopo e `RFC_ARQUITETURA.md` para a
arquitetura de referência.

**Manifest do PWA:** usar os tokens do sistema para `theme_color` e
`background_color` (`--bg-deep` `#060a14` e `--surface-default` `#0f1729`),
ícones 192 e 512. Atenção: os valores citados na `RFC_ARQUITETURA.md`
(`#1F2937`/`#111827`) são de um rascunho antigo e **divergem** do sistema
cometido; onde houver conflito, vale o `tokens.css`.

### Site e landing

- Gradientes inspirados no amanhecer (com parcimônia, o horizonte do
  `auth-layout` é a referência).
- Ilustrações de comunidade e servidores, mockups do superapp, seção clara
  sobre os serviços do MVP (Ybira e Jaci; **sem Guará**).
- Estado inicial ("ligar, acessar `http://taina.local`, seguir o wizard") é o
  argumento de venda: a experiência de instalação é o produto.
- Landing não é o lugar do formulário denso nem da grade de dez cards
  idênticos. Uma ideia por seção, um CTA primário.

### Documentação (Tekoá)

- Leitura confortável com hierarquia tipográfica (`type-*`), busca eficiente e
  destaque para blocos de código (mono).
- Seção dedicada aos padrões do superapp e guias de contribuição de UI/UX.
- Mesma paleta noturna, mesma marca; a doc é a "aldeia digital" do projeto.

---

## 13. Checklist antes de mandar tela

- [ ] Consome só token semântico, nunca primitivo, hex ou px cru.
- [ ] Reusa componente de `core_components`/`components.css` em vez de refazer.
- [ ] Entrega estado vazio, de erro e de permissão, não só o feliz.
- [ ] Sem scroll horizontal; alvo de toque >= 44px; foco visível e não cortado.
- [ ] Cor acompanhada de símbolo ou frase; contraste 7:1 (texto) e 3:1 (UI).
- [ ] Movimento CSS-first, contido, e desligado em `prefers-reduced-motion`.
- [ ] Copy em pt-BR pelo glossário; nada de "admin"; erros com saída; vazios
      que ensinam.
- [ ] Sem emoji e sem AI-typography; acentos pt-BR presentes.
- [ ] Sem prometer criptografia; sem telefone-para-casa; sem Guará no MVP.
- [ ] Fala como vizinhança, não como plataforma.

---

_Alinhado a `briefing.md`, `tom-de-voz.md`, `RFC_002_MVP.md`,
`RFC_003_GOVERNANCA_E_TRANSPARENCIA.md` e ao sistema cometido em
`taina/assets/css/`. Documento vivo._
