# RFC 004: Produto Universidade (Workspace soberano), adiado

- **Status:** Adiado (rascunho mínimo, nao implementar)
- **Autora:** Zoey de Souza Pessanha (`@zoedsoupe`)
- **Data:** 2026-06-24
- **Relacao:** Deriva do Taina, mas NAO faz parte do MVP do `RFC_002_MVP.md`. E um **produto separado e futuro** que herda a marca e o nucleo soberano do Taina. O desenvolvimento so comeca **depois** que o MVP do Taina lancar (gate da Fase 4 do RFC_002).

> Este RFC existe so para registrar a decisao e o escopo de forma deliberada, para que a ideia nao se perca nem contamine o MVP. Nada aqui e para construir agora.

---

## 1. Motivacao

Universidades publicas dependem fortemente do Google Workspace (Docs, Drive, Agenda, Meet) para reunioes, documentos e arquivos. E a mesma tese de soberania do Taina, em escala institucional: uma instituicao publica nao deveria ter a memoria e a colaboracao do dia a dia presas num provedor que a vigia e pode cortar o acesso.

A tentacao e pivotar o Taina para isso. A decisao e o oposto: **nao pivotar**. O Taina-comunidade (cofre de arquivos e fotos, experiencia comunitaria) e um produto; o substituto de Workspace e outro, mais pesado, para outro publico (instituicao, nao vizinhanca). Forcar os dois num so dilui o foco e mata o MVP.

## 2. O que este produto adiciona alem do cofre do MVP

O MVP do Taina e um cofre de arquivos e fotos. Substituir o Workspace acrescenta um conjunto de capacidades de outra classe de peso:

- **Edicao colaborativa em tempo real** de documentos (CRDT ou OT). E a coisa mais dificil de software colaborativo; mesmo adotando um motor pronto (Yjs, Automerge, ou integracao Collabora/OnlyOffice) e uma integracao grande, com estado no cliente, o oposto direto da tese "LiveView, sem estado de cliente" do RFC_002 D1.
- **Presenca em tempo real**, cursores, resolucao de conflito.
- **Agenda** (CalDAV, eventos recorrentes, convites, livre/ocupado).
- **Reunioes por video** (SFU WebRTC, TURN, gravacao). Uma disciplina de infraestrutura inteira por si so.
- **Barra de confianca institucional:** SSO, auditoria, retencao e, o ponto fatal, **confidencialidade**. O Taina hoje guarda bytes em texto puro (RFC_003 D7); nao da para prometer a uma instituicao "nao conseguimos ler seus documentos" sem criptografia E2E, que nao existe.

Qualquer um desses e um trabalho de varios meses; juntos, sao o roadmap de varios anos de um time financiado. Por isso: produto separado, depois.

## 3. O que herda do Taina (e por que vale herdar)

Em vez de comecar do zero, herda o nucleo soberano ja construido:

- **Maraca**: identidade, papeis, autorizacao, convites, recuperacao.
- **Ybira**: armazenamento enderecado por conteudo, cotas, lixeira; base do Drive institucional.
- **Modelo de zonas e governanca** (casa/praca, zelador/morador, assembleia): a colaboracao institucional precisa de espacos compartilhados e de governanca legivel; o Taina ja modela isso.
- **Pilar de transparencia** (mural, painel, soberano, nunca telefona para casa): diferencial real contra o Google numa instituicao publica.
- **A cunha de operabilidade** (instalar e atualizar sem dor): o mesmo valor, agora para o TI de um departamento.

A regra de fronteira do Taina vale aqui tambem: o produto institucional se **acopla** a um nucleo estavel; o nucleo nunca depende dele.

## 4. Por que adiado, e o gatilho para revisitar

Adiado porque: (a) os pesos da secao 2 estao fora de alcance de uma dev solo antes do MVP; (b) sem E2E, a promessa de confidencialidade institucional e desonesta; (c) construir isso agora abandona o diferencial real do Taina (governanca comunitaria para gente nao-tecnica) por um mercado (produtividade institucional) onde Nextcloud+Collabora e OnlyOffice ja estao.

**Gatilho para revisitar:** o MVP do Taina lancado (Fase 4 fechada) **e** uma instituicao real pedindo. Ate la, se uma universidade quiser sair do Google, a resposta honesta de curto prazo e "Nextcloud com Collabora", nao "espere o Taina criar um CRDT". O encaixe legitimo de curto prazo do proprio Taina e o **drive compartilhado com governanca** (o produto que ja existe), nao o Workspace.

## 5. Questoes em aberto (para quando sair do papel)

- Nome de marca: a definir, seguindo a convencao do RFC_003 secao 5 (Tupi-Guarani para nome proprio de produto). Nao inventar agora.
- Adotar motor de edicao pronto (Collabora/OnlyOffice/Yjs) versus construir: quase certamente adotar.
- Relacao com "Taina Hospedada" (multi-tenant gerenciado, RFC_002 D10): o produto institucional provavelmente nasce sobre o RLS multi-tekoa.
- Criptografia E2E no Ybira como pre-requisito da promessa de confidencialidade.
