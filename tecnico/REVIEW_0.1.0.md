# Revisão 0.1.0: app nativo, cofre comunitário e prontidão para substituir iCloud/Google

**Status:** Artefato de revisão (não normativo)
**Data:** 2026-10-21
**Escopo:** Tainá `0.1.0` a `0.4.0` (repo `taina`), app nativo `okena` (branch `feat/ios-app`), deploy `oka`.
**Método:** seis revisores paralelos, cada um ancorado nas RFCs e no `AGENTS.md`. Reconstrução a partir do código, não de um app em execução nem do Penpot.

---

## 1. Veredito: substituir iCloud / Google?

**Dividido.**

- **Como backup do telefone de uma pessoa: Não, e não chega perto.** O loop que define iCloud/Google Photos (backup automático do rolo de câmera) não existe em nenhuma forma. O app nativo tem **zero** acesso à biblioteca de fotos: sem `PHPhotoLibrary`, sem `PhotosPicker`, sem câmera, sem `BGTaskScheduler`, sem background. Não lê o rolo de câmera nem manualmente. Esse era o motivo declarado de o app nativo existir (a RFC 002 descartou o PWA justamente porque o iOS não dá acesso ao rolo de câmera via PWA).
- **Como cofre comunitário (fotos + documentos compartilhados de uma comunidade não-técnica): Parcial, condicional.** É de fato o armário de arquivos e fotos da comunidade, com uma história de governança e transparência que nenhum incumbente tem. Mas faltam três itens de base (busca, compartilhamento por link, backup off-site) e o próprio gate do MVP não foi provado (as quatro provas de campo).

A afirmação honesta é mais estreita que "substituir Google/iCloud": **um cofre comunitário funcional, a um gate de MVP não-provado de ser instalável, e a uma trilha pós-MVP de app+API de tocar o problema do backup do telefone.**

---

## 2. O padrão estrutural por trás da maioria dos achados

O app nativo (`okena`) foi construído como um visualizador de leitura, à frente da experiência que o justificou. Três lacunas explicam a maior parte:

1. **Não faz a única coisa para a qual existe:** sem upload do rolo de câmera nem da câmera.
2. **O onboarding morre no caminho:** o link de convite não abre o app.
3. **O modelo de consentimento (casa/praça) é invisível onde o conteúdo vive**, então o caminho padrão compartilha nada silenciosamente.

E web e app **divergiram em dois modelos sociais**: a web tem abas casa/praça, rótulo de zona por linha, mover filtrado por dono, metadados humanizados, PubSub; o app tem lista mista, badge só na praça, mover sem filtro, `sha256` cru, sem polling.

---

## 3. Crítico (corrigir antes do piloto)

### C1. O único zelador que esquece a senha fica trancado para fora da caixa
`members_live.ex:216`, `maraca.ex:491`, `account_live.ex`, `AccountView.swift`, `guias/*`

Não há troca de senha self-service em lugar nenhum (sem `update_password` no Maracá, nada nas duas telas de Conta), o link de reset fica escondido para você mesmo (`can_reset?` exclui `member.id == scope.ava.id`) e não existe recuperação de operador: nenhuma mix task em `lib/mix/tasks/`, nenhuma seção no runbook. O backend permitiria um self-reset (`mint_reset_link` não exclui a si), mas nenhuma UI expõe isso, e ainda exigiria estar logado.

**Impacto:** a única pessoa que instalou a caixa, o único zelador, perde a comunidade sem caminho documentado de volta.

**Correção:** (a) uma task `taina senha` documentada no runbook (o operador já tem shell); (b) "Trocar minha senha" em Conta nos dois clientes.

### C2. Restaurar um arquivo de uma pasta apagada o deixa invisível para sempre
`ybira.ex:257`, `ypy/lib/ypy/ybira/file.ex:110`, `trash_live.ex:900`

Apagar uma pasta faz soft delete da subárvore inteira (`soft_delete_tree/2`). Restaurar um arquivo só limpa `apagado_em`; `folder_id` continua apontando para a pasta soft-deleted. A listagem raiz filtra `is_nil(f.folder_id)` e a listagem de pasta exige uma pasta viva e legível, então o arquivo restaurado **não aparece em lugar nenhum** (e continua consumindo cota). As duas superfícies prometem o contrário ("Os arquivos voltam um a um; a pasta em si, não").

**Impacto:** o desastre diário mais provável. Apagar uma pasta, mudar de ideia, restaurar cada foto, e toda foto some com a confirmação "Arquivo restaurado.". A pessoa conclui que o cofre comeu os arquivos.

**Correção:** ao restaurar, se a pasta-pai estiver soft-deleted, restaurar para a raiz (ou restaurar a cadeia de ancestrais). E corrigir a cópia. Melhor ainda: tornar a pasta restaurável como unidade, que é o que "lixeira" significa em todo gerenciador de arquivos que esse público já usou.

### C3. HEIC (padrão do iPhone) não gera miniatura, silenciosamente, e o libvips pode não estar na imagem
`ybira/workers/rendition.ex:69`, `mime_detector.ex:69`, `Dockerfile:79`

`image/heic`/`image/heif` são detectados e permitidos, então o job é enfileirado. Se o libvips da caixa não tiver libheif (o caso comum), `Media.analyze/1` e `Media.thumbnail/3` devolvem `{:error, _}`, o worker loga e devolve `:ok` (best-effort, correto para não travar o upload), e a foto fica **sem miniatura para sempre**. `GET /files/:id/thumbnail/sm` devolve 404 sem que o app distinga "ainda processando" de "nunca vai existir". A grade mostra quadrados em branco. A regeneração é uma mix task manual, não exposta em lugar nenhum.

Pior: o estágio de runner do `Dockerfile` instala só `libstdc++6 openssl libncurses6 locales ca-certificates curl`. O pipeline de miniatura depende inteiramente do que o artefato pré-compilado do `vix` carrega. Se HEIF/JPEG não estiver suportado no container publicado, **toda miniatura de toda comunidade falha em silêncio**.

**Correção:** verificar dentro do container publicado que `Image.open/1` funciona e que HEIF está entre os formatos; gravar a falha (`rendition_error`) em vez de só logar; e devolver um código distinto (`rendition_pending` vs `rendition_unavailable`) em vez de 404 cru.

### C4. O link de convite não abre o app
`invite_live.ex:47`, `member_controller.ex:39`, `DeepLink.swift:22`, `Info.plist`

A tela de convite da web monta `url(~p"/convite/#{token}")`, um link `https`, e o QR codifica esse mesmo link. O único produtor de `taina://convite/<token>?host=` é o endpoint de zelador da API, que nenhuma UI chama. `DeepLink.parse` só aceita o scheme `taina`, e não há associated-domains, então `https://<box>/convite/...` nunca abre o app. `InviteAcceptView` é código morto em produção.

**Impacto:** o onboarding é por convite ("chegar como pertencimento"), e hoje a vizinha convidada: recebe o link, abre no Safari, cria a conta na web, e fica sem caminho para o app. Para entrar, precisa instalar o app, ler `http://taina.yby/convite/abc` e extrair o host à mão, e digitar.

**Correção:** no curto prazo, um campo "Cole aqui o link do convite" no Setup/Login do app, mais um segundo QR na tela de convite da web codificando `taina://convite/<token>?host=<host>`. Depois, universal links com AASA servido pela caixa.

### C5. O app não consegue enviar uma foto do rolo de câmera
`FilesView.swift:62`

`fileImporter(allowedContentTypes: [.data])` abre o picker de documentos (iCloud/Files), não a biblioteca de fotos, e um arquivo por vez. Não há `PhotosPicker`, câmera, nem permissão de biblioteca em nenhum lugar do `ios/`. A **web** faz melhor: input de câmera com `capture="environment"` e 10 arquivos por vez.

**Impacto:** a ação mais comum de um cofre de família ("aqui está a foto do aniversário") é impossível no app, enquanto funciona no navegador. O app é estritamente menos capaz que o PWA que veio substituir.

**Correção:** `PhotosPicker` multi-seleção + captura de câmera, com fila de upload e progresso/cancelar. Backup automático em background é corte legítimo; escolher da galeria manualmente não é.

### C6. Casa/praça invisível em Fotos
`gallery_live.ex:590`, `GalleryView.swift:257`, `files_live.ex:29`

Uploads nascem em casa, e a única tela onde as fotos vivem não diz nada sobre isso: o modal de Detalhes não tem zona, os tiles não têm badge (`.photo-grid__tag` existe no CSS e não é usado em lugar nenhum), Fotos não tem filtro de zona (Arquivos tem), e no app não há `ZonaBadge`. Publicar só existe em `/arquivos/:id`, em outra seção.

**Impacto:** a pessoa fotografa a festa, sobe 20 fotos, vê em Fotos, conta para a comunidade, e ninguém vê. Pior que um bug: a RFC 003 §2 faz da legibilidade da zona o próprio mecanismo de consentimento, e esta é a única tela que o esconde. O caminho padrão (não mudar nada, subir, olhar) produz uma falha silenciosa da promessa central.

**Correção:** badge no tile (a classe já existe), zona nos Detalhes, ação "Publicar na praça" no visualizador nas duas superfícies, filtro casa/praça em Fotos espelhando Arquivos.

### C7. O passo de publicar na praça está escondido depois do upload
`upload_live.ex:237`, `FilesModel.swift:52`

Todo upload cai em casa. A web avisa no dropzone e no empty state; o iOS não diz nada. Depois do upload, as linhas "Concluído" não oferecem nada: sem "publicar na praça", sem lembrete. Publicar existe só no menu de três pontos de um item específico, numa aba específica.

**Impacto:** a RFC 003 §2 diz que colocar um arquivo na praça **é** o ato de consentimento. Do jeito que está, o caminho padrão é: todos sobem privado, ninguém descobre o publicar, e a memória coletiva (o ponto inteiro do Ybira) fica vazia enquanto o disco enche de silos privados. Para a pessoa menos técnica, "publicar" é uma palavra que ela nunca encontrou.

**Correção:** consentimento em um toque legível no momento da intenção. Na tela de upload, após cada arquivo, uma ação na linha: **"Publicar na praça"**. No iOS, idem no estado de concluído, e rotular o destino do upload ("Vai para a sua casa"). Manter o padrão casa (não relitigar a D1), mas parar de esconder a reversão.

---

## 4. Alto (quebra de uso diário)

- **Navegar para uma pasta a partir da aba Praça cai na aba Casa vazia** ("Nada seu nesta pasta" numa pasta cheia). `browse_path/4` carrega `?sort&view&zona`, mas os links não passam a query. Um helper, cinco call sites: `files_live.ex:568`.
- **"A pasta nasce na praça" é falso:** `create_folder/2` descarta `zona` e cria em casa; a aba Praça então não mostra nada. Confirmação falsa + objeto que some: `files_live.ex:190`.
- **Arquivos trunca silenciosamente em 50** (`nextCursor` ignorado, sem contagem, sem load-more): `FilesModel.swift:198`.
- **O visualizador de fotos do app só olha** (sem compartilhar, baixar, detalhes, excluir, anterior/próximo) e **vídeos são um `play.fill` morto**: `GalleryView.swift:306`, `:372`.
- **Linhas do Início são inertes mas parecem clicáveis** (recentes, mural, cartão de pedidos): `HomeView.swift:93`. A primeira tela ensina que "tocar não faz nada".
- **Setup ensina o endereço errado:** assume `https://`, mas a caixa do piloto responde `http` puro; toda falha colapsa em "verifique a conexão": `InstanceAddress.swift:29`.
- **A cota mente em três lugares:** apagar não libera nada por 30 dias (a cópia diz "apague algo"), pode ser definida abaixo do uso (renderiza `-75161927680 B`), e o detalhamento exclui lixeira/casa alheia: `ybira.ex:408`, `FilesModel.swift:131`, `storage_live.ex:165`.
- **O cursor keyset da linha do tempo compara duas codificações ISO8601** (`taken_at` sem `Z`, `inserted_at` com `Z`): rajadas no mesmo segundo podem pular fotos. E a linha do tempo **não usa índice** (`ORDER BY COALESCE(json_extract(...))`): cada página é um scan completo: `jaci.ex:101`.
- **Consentimento é de mão única no telefone:** sem `GET /grants`, sem revogar, então um acesso aprovado nunca pode ser visto nem retomado. A RFC 003 diz que conceder deve ser reversível: `RequestsView.swift:66`.
- **A zeladora não consegue fazer o trabalho dela pelo app:** sem convite, mudança de papel, link de reset, cota ou disparo de backup. E Moradores não diz onde as ações vivem: `MembersModel.swift:5`.
- **Sem polling, sem `scenePhase`:** o app nunca atualiza, nem no foreground. A RFC 005 D5 prometeu polling: `SessionModel.swift`.
- **`folder` é sempre `null`** em `/photos`, `/recent`, `/trash` (falta `preload(:folder)`), correção de uma linha: `file_controller.ex:70`.

---

## 5. Médio / polimento (rápidos, alto impacto emocional)

- **Ícone de cadeado em casa** (`ZonaBadge`) contradiz a RFC 003 §7 ("promessa de software, não cadeado matemático") numa conexão não-criptografada: `Components.swift:196`.
- **`sha256`, MIME cru e o TID do aparelho** mostrados a pessoas não-técnicas (a web humaniza os três): `FileDetailView.swift:151`, `AccountView.swift:187`.
- **Botões vermelhos de perigo para "Tornar zelador(a)" e "Sair da comunidade"** (promoção lê como destrutivo; logout lê como sair do grupo): `members_live.ex:406`, `account_live.ex:100`.
- **O "Motivo" do pedido de acesso é fabricado** pelo servidor ("Pedido de acesso pela tela do arquivo") e mostrado como se um humano tivesse escrito: `file_preview_live.ex:69`.
- **O nome do dono é buscado, enviado no fio, e jogado fora** ("peça acesso e a pessoa decide" em vez de "Pedir acesso a Dona Maria?"): `file_preview_live.ex:289`, `FileDetailView.swift:137`.
- **Nomes de arquivo acentuados são corrompidos no download/share** (o cliente lê só `filename=`, ignora `filename*=UTF-8''`): `LiveTainaAPI.swift:648`.
- **A folha de mover oferece destinos que a caixa vai recusar**, com erro mal atribuído: `MoveSheet.swift:4`.
- **Spinner circular genérico em 25 lugares** onde o `DESIGN.md` manda skeleton: `Components.swift:299`.
- **Um tema claro é entregue que o `DESIGN.md` diz explicitamente que não existe**; **sem ícone de app**; **`NSAllowsArbitraryLoads: true` global**; **Keychain `AfterFirstUnlock`** (o token viaja num backup do iCloud para um telefone novo): `Theme.swift:154`, `project.yml:39`, `project.yml:59`, `Keychain.swift:23`.
- **Corrida de GC na purga da lixeira** (dedup + purga pode apagar um blob que um upload concorrente acabou de commitar): `ybira.ex:534`.
- **Um upload de 2 GiB é lido inteiramente na memória** no processo do LiveView (pode derrubar um Pi): `ypy/lib/ypy/ybira.ex:92`, `upload_live.ex:89`.

---

## 6. Lacunas de API para um app nativo photo-first

1. **Sem endpoint de manifesto/delta de hashes**, então backup em background é impossível mesmo quando o app tiver acesso a Fotos (`sha256` está no fio, mas nada devolve o conjunto de hashes existente).
2. **Sem operações em lote** (um arquivo por POST; excluir/mover em lote).
3. **Sem `taken_at`/dimensões/estado de miniatura no fio** (o cliente não ordena/agrupa localmente nem distingue pendente de falho).
4. **Sem `GET /grants` / revogar.**
5. **Sem requisições condicionais** (sem ETag/If-None-Match).

---

## 7. O que é genuinamente bom (não mexer)

A regra de leitura de zona aplicada em SQL em todo caminho (incluindo a miniatura), a distinção 403-vs-404, o enquadramento honesto de não-cripto onde importa, a voz da cópia em pt-BR (a melhor parte do app), o mural append-only com tabela de frases compartilhada, a separação Functional Core / Imperative Shell, `SessionModel.screen/1` como ponto único de 401, e a disciplina de Fakes/previews.

---

## 8. Deriva de documentação encontrada no caminho

- **`taina/SECURITY.md` ainda afirma Row Level Security do PostgreSQL** (removido pela RFC 006). É uma afirmação de segurança, então importa.
- `CHANGELOG.md` para em `0.3.0` enquanto as tags vão até `v0.4.0`.
- **RFC 005 §8** descreve o bug "subir para a pasta de praça alheia" como aberto; o kernel agora fecha (`owner_guard`).

---

## 9. Método (o fan-out)

Seis revisores em paralelo, todos ancorados nas RFCs e no `AGENTS.md`:

1. Experiência Maracá (identidade, onboarding, governança), web + app.
2. Experiência Ybira (arquivos/drive), web + app.
3. Experiência Jaci (fotos + pipeline), web + app.
4. App nativo `okena` inteiro, paridade Apple Photos, consistência com a web.
5. Implementação dos contexts Maracá/Ybira/Jaci + costura de API.
6. Prontidão para substituir iCloud/Google.

Os três revisores por context encontraram repetidamente os mesmos problemas transversais por ângulos diferentes (rolo de câmera, convite, casa/praça), que é exatamente o sinal desejado. Uma adição pagaria numa próxima rodada: um revisor de fidelidade ao design system com acesso ao MCP do Penpot (o revisor do app sinalizou que não pôde julgar hierarquia renderizada nem contraste, e o design comprometido vive no Penpot).
