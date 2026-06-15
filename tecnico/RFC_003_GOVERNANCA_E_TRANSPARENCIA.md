# RFC 003: Modelo Social, Governança e Transparência

- **Status:** Rascunho (proposto)
- **Autora:** Zoey de Souza Pessanha (`@zoedsoupe`)
- **Data:** 2026-06-15
- **Relação:** Refina e estende o `RFC_002_MVP.md`. As Fases 1-2 são _acabamento_ do modelo de permissões já especificado; a Fase 3 (Assembleia) é **expansão de escopo além do MVP** e só começa após aceite deste RFC.

---

## 1. Motivação

A documentação do Tainá prega uma "aldeia digital" comunitária (`RFC_ARQUITETURA.md` seção "Technology for Liberation, Not Capture"), mas o modelo implementado é **individualista-liberal**: cada `ava` é dono dos seus bytes e a comunidade não é dona de nada. Pior, três camadas discordam:

- **Especificação** (`maraca/behaviour.ex`): privado por padrão, negação padrão, _quem cuida (zelador) precisa pedir acesso_, "admin não é deus".
- **Código rodando** (`ybira.ex`): leituras são **amplas na tekoa** (só RLS, sem `authorize?`, sem filtro por `ava_id`), _comuns de fato_. Só `delete`/`lixeira` são restritos ao dono.
- **UI**: o modelo social **não existe na interface**, o motor de permissões (`Permission`, `AccessRequest`, `authorize?`, `request_access/approve/deny`, PubSub, auditoria via `granted_by_id`) está completo no backend e **sem nenhuma tela**.

Este RFC resolve a tensão **por escopo** (duas zonas), dá rosto ao motor que está no escuro, separa _quem cuida da máquina_ de _quem tem autoridade sobre as pessoas_, e mantém o poder honesto pela transparência, depois, pela decisão coletiva.

Princípio que orienta tudo: **a tensão entre "admin não é deus" (privado por padrão) e "aldeia/comuns" (compartilhado) é real.** Pedir permissão até para _ler_ tudo seria o oposto do comunitário, vira privacidade-máxima individualista e mata o convívio. A solução não é gatekeeping universal; é tornar legíveis **duas zonas** que espelham uma aldeia de verdade: a _praça_ e a _casa_.

---

## 2. Modelo de duas zonas: praça e casa

| Zona               | Padrão               | Quem lê                                                        | Ato de consentimento                     |
| ------------------ | -------------------- | -------------------------------------------------------------- | ---------------------------------------- |
| **praça** (comuns) | -                    | todo morador                                                   | publicar (mover casa->praça), reversível |
| **casa** (pessoal) | sim (arquivos novos) | só o dono + concessão explícita (**o zelador não tem atalho**) | pedir acesso -> dono aprova              |

- A **praça** é a memória coletiva (fotos da festa, estatuto, atas). Contribuir é dar para os comuns, _colocar_ algo na praça **é** o ato de consentimento, legível e reversível. Ninguém pede acesso para ler a praça.
- A **casa** é o espaço pessoal de cada `ava` guardado na caixa da comunidade. Privado por padrão. Para outra pessoa abrir, ela **pede e o dono decide**, vale também para o zelador.
- `Maraca.authorize?/4` é reaproveitado sem mudança: `praça OU dono OU permissão explícita`. O RLS continua sendo a fronteira externa da tekoa.

**Decisão técnica:** zona é um `Ecto.Enum` (`~w(casa praca)a`, padrão `:casa`) em `Ybira.File` e `Ybira.Folder`, uma coluna que decide a privacidade do item, explícita e estável a movimentações. A zona da pasta governa só a listagem da própria pasta; **não cascateia** para os filhos. Migração faz _backfill_ dos arquivos existentes para `:praca` (preserva a visibilidade comum de hoje; default `:casa` privatizaria o acervo atual silenciosamente).

---

## 3. Papéis: zelador(a) e morador(a)

"Admin não é deus" só é verdade se o admin não for, de fato, deus. Mas alguém **precisa** cuidar da máquina (reiniciar a caixa, rodar backup, gerar convites). A jogada comunitária é **separar operador-da-máquina de autoridade-sobre-pessoas**:

- **Zelador(a)**: cuida da infraestrutura (disco, backups, atualizações, convites). **Zero autoridade sobre dados, zero poder social unilateral.** Toda ação do zelador é visível no **mural**.
- **Morador(a)**: qualquer membro. Usa praça + casa, contribui com a memória, pede e concede acesso.

Uma tekoa pode ter **vários zeladores**, cuidar da máquina é trabalho que se distribui e roda, não um trono único. Novos zeladores entram por convite (Fase 1) e, na Fase 3, por proposta da assembleia. O zelador **não tem atalho** para a casa de ninguém, pede acesso como qualquer morador (já garantido: `authorize?` não dá nada automático).

**Decisão técnica:** Tainá é pré-alpha (sem release) -> **renomear o enum no banco** em migração limpa: `Ava.role` passa a `~w(zelador morador)a`. Sem ressalva de compatibilidade de sessão. Predicados `Maraca.zelador?/1` / `Maraca.morador?/1`.

---

## 4. Convenção de nomes (uma fonte de verdade)

O Tainá mistura três línguas; este RFC fixa **quando cada uma vale**, para não misturar ad-hoc:

1. **Tupi-Guarani**: **nomes próprios** de subsistemas/serviços: Tekoa, Maracá, Ybira, Jaci, Guará, Nhaman. São como nomes de marca, aprendidos uma vez no onboarding.
2. **pt-BR familiar**: o **vocabulário social do dia a dia** que o morador usa: casa, praça, zelador, morador, mural, pedido de acesso, assembleia. **Uma fonte de verdade**: a _mesma_ palavra é o átomo no banco (`:casa`, `:praca`, `:zelador`, `:morador`), o termo no contexto e o rótulo na UI, **sem camada de tradução** entre código e tela.
3. **Inglês**: apenas o encanamento invisível que o usuário nunca lê (`Repo`, `Scope`, `Telemetry`, verbos genéricos de helper).

**Por quê:** o público é a pessoa menos técnica da comunidade; conceitos do cotidiano precisam ser imediatamente familiares, Tupi é certo para _nomes de marca_, errado para _substantivos comuns_ do dia a dia. Uma fonte de verdade elimina a deriva entre o vocabulário do código e como todo mundo de fato fala (linguagem ubíqua, na língua da comunidade). Esta é a exceção deliberada ao "código em inglês", mesma justificativa do Tupi já intencional.

---

## 5. Transparência como pilar central (não é feature)

A transparência é o **esforço comunitário de verdade**, legibilidade como liberação. A comunidade pode _ver e entender_ tudo o que o sistema e seus membros fazem, em pt-BR simples e acessível (WCAG). Três camadas, todas no MVP:

1. **Mural**: transparência social: quem fez o quê (convites, publicar-na-praça, concessões de acesso, atos de governança). Tabela `maraca.events` **append-only**.
2. **Painel / "Saúde da comunidade"**: estado da máquina em linguagem simples: uso/saúde do disco, último backup, jobs pendentes (renditions, expurgo da lixeira), número de moradores, status de atualização. Tela voltada à comunidade; o zelador também tem o `Phoenix.LiveDashboard` técnico.
3. **Substrato**: `:telemetry` + `telemetry_metrics` + metadados estruturados no `Logger`.

**Princípio inegociável, SOBERANO: nunca telefonar para casa.** Toda telemetria/métrica/log fica **na caixa da comunidade**. Enviar dados da comunidade para qualquer SaaS (PostHog etc.) trairia a soberania de dados. É essa linha que faz a transparência ser _comunitária_ e não _vigilância_.

**Acessibilidade** em dois sentidos: legível para não-técnicos **e** WCAG (contraste, rótulos, foco). As correções de acessibilidade da revisão de UI pertencem a este pilar, não a um "polimento" à parte.

---

## 6. Privacidade honesta ("trusted host")

Hoje **não há criptografia**: o Ybira guarda bytes em texto puro; o `file_hash` SHA-256 é para dedup, não proteção. Quem opera a caixa tem acesso físico/root.

Logo a UI **não pode** prometer garantia criptográfica. A privacidade da casa é **promessa de software e de confiança**, "promessa de software, não cadeado matemático; o Tainá ainda não criptografa". Confie no servidor como em quem hospeda a caixa da comunidade.

Só uma futura criptografia **E2E** no Ybira (chaves que o host não lê, _não_ a convergente, em que o host detém as chaves) permitiria dizer honestamente que o zelador **não consegue** ver a casa. Não prometer isso na UI antes de existir.

---

## 7. Governança coletiva (Assembleia), expansão de escopo

> **Além do MVP do RFC_002**, que cortou governança explicitamente. Só começa após aceite deste RFC.

O RFC_002 não tem decisão coletiva: zelador decide convites/cota/remoção sozinho. Para uma "comunidade dona dos próprios dados", essa é a lacuna real. A resposta:

- **Transparência primeiro** (Fase 2, mural): todo poder do zelador é visível. É o cheque comunitário mais barato e a fundação.
- **Decisão coletiva depois** (Fase 3, assembleia): atos sensíveis viram **propostas** que a comunidade vê e vota.

**Esboço técnico:** contexto novo `Taina.Assembleia` (`@behaviour` próprio). `Proposal` (`type` é um de remove_member/appoint_zelador/revoke_zelador/change_quota, `payload`, `proposer_id`, `status` é um de open/passed/rejected/expired, `closes_at`) + `Vote` (único por `(proposal_id, ava_id)`). **Divisão de ações:** com voto = atos que afetam permanentemente uma pessoa (remover, nomear/destituir zelador); diretos-mas-registrados = convite, cota. **Regra:** maioria simples dos votos + quórum `ceil(moradores/2)`; expira por worker Oban. Na aprovação, a ação só executa com a proposta aprovada como _token_ de autoridade, **não há caminho unilateral**. Corte mínimo viável: só `remove_member`, provando o ciclo. (Nomear zelador via proposta é como a comunidade ganha mais zeladores além dos convidados na Fase 1.)

---

## 8. Decisões

- **D1**: Duas zonas (`:casa`/`:praca`) via `Ecto.Enum` em `Ybira.File`/`Folder`; padrão `:casa`; backfill existente -> `:praca`; zona de pasta não cascateia.
- **D2**: Aplicar a regra de leitura (`praça OU dono OU permissão`) em **todo** caminho de leitura do Ybira/Jaci e no controller de download/thumbnail (o thumbnail é o vazamento nº 1). `get_file` devolve `:forbidden` (diferente de `:not_found`) para casa não autorizada.
- **D3**: Papéis `:zelador`/`:morador`: **renomear o enum no banco** (pré-alpha, sem ressalva de sessão); **múltiplos zeladores** por tekoa permitidos.
- **D4**: Generalizar `request_access` para qualquer morador (hoje é admin-only); adicionar `list_my_requests/1`.
- **D5**: Convenção de nomes em três camadas, **uma fonte de verdade** para o vocabulário social (átomo == contexto == UI em pt-BR); Tupi só para nomes de marca; inglês só para encanamento.
- **D6**: Transparência (mural + painel + substrato) é pilar de MVP; **soberana, nunca telefona para casa**; acessibilidade WCAG faz parte dela.
- **D7**: Privacidade da casa é promessa de software+confiança, não criptográfica.
- **D8**: Governança coletiva (assembleia) é expansão pós-RFC_002; transparência primeiro, voto depois.

---

## 9. Impacto no RFC_002 / ROADMAP

- `RFC_002_MVP.md` e `RFC_ARQUITETURA.md` devem ser emendados para introduzir o modelo de zonas, o reframe zelador/morador (com múltiplos zeladores), a convenção de nomes e o pilar de transparência.
- `ROADMAP.md` deve encaixar **mural + painel** no MVP e **assembleia** como trilha própria.
- Onde código e `tekoa` divergirem, `tekoa` vence, corrigir a deriva, não inventar uma terceira resposta.
