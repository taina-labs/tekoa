# RFC 008: Espelho do rolo de câmera e estado em segundo plano

> **Proposta histórica.** O comportamento foi condensado para o app Flutter na [arquitetura atual](../ARQUITETURA.md). As referências a SwiftData e APIs específicas de Swift não são plano atual.

**Status:** Proposta
**Data:** 2026-10-22
**Relação:** Deriva do `RFC_002_MVP.md` (o app nativo de backup de fotos é o item 2 do pós-MVP; o PWA foi descartado justamente porque o iOS não dá acesso ao rolo de câmera) e consome o contrato do `RFC_005_API_E_APP.md`. Fecha a lacuna C5 e a lacuna de API 1 da `REVIEW_0.1.0.md`. Herda o modelo social do `RFC_003_GOVERNANCA_E_TRANSPARENCIA.md` sem exceção e diz em voz alta o que este RFC **não** promete. Frente oposta ao `RFC_007_BACKUP_CIFRADO.md`: aqui o backup é do aparelho para a caixa, lá é da caixa para a nuvem.
**Autoria:** @zoedsoupe + revisão técnica

---

## 1. Resumo executivo

O app nativo existe por uma razão só. O `RFC_002_MVP.md` cortou o PWA offline-first com uma frase: "iOS não tem Background Sync nem acesso ao rolo de câmera via PWA". O app nativo entrou no lugar dele, e a `REVIEW_0.1.0.md` (C5) registrou o resultado sem meias palavras: o app tem **zero** acesso à biblioteca de fotos, e a ação mais comum de um cofre de família, subir a foto do aniversário, funciona no navegador e não funciona no app.

Esta RFC define o **espelho do rolo de câmera**: o aparelho sobe sozinho o que aparece na biblioteca de Fotos, e o estado do cofre se atualiza em segundo plano. Três decisões de base.

1. O espelho é **unidirecional e para a casa**. Do aparelho para a caixa, nunca o contrário, e nunca direto para a praça. Publicar continua sendo um ato explícito e separado.
2. **"Contínuo" não existe no iOS.** O sistema não entrega execução contínua em segundo plano a um app de terceiros, e o Apple Fotos só espelha continuamente porque é o sistema. A promessa honesta é "cedo ou tarde, e mais rápido com o app aberto", e a interface diz isso.
3. O trabalho é um **diff entre duas verdades**: o índice local do aparelho e o manifesto de hashes da caixa. O resto é encanamento.

O que este RFC **não** é: cifra (nada aqui esconde a casa do zelador), espelho bidirecional, propagação de exclusão, upload resumível, e nada de E2E. A promessa da casa continua sendo software e confiança (RFC 003, D7).

---

## 2. Motivação

A `REVIEW_0.1.0.md` resume o estado com uma frase que é o problema inteiro: como backup do telefone de uma pessoa, **não, e não chega perto**. O loop que define iCloud e Google Fotos, subir o rolo sozinho, não existe em nenhuma forma.

Três razões para fechar isso agora, e não depois.

- **É o motivo declarado do app.** O `RFC_002_MVP.md` (§2) cortou o PWA com o argumento de que só um app nativo alcança o rolo de câmera. Um app que não alcança o rolo de câmera é um PWA mais caro.
- **É o gancho emocional do produto.** O `RFC_002_MVP.md` (D4) já dizia que fotos são o gancho, "o caso de uso que mais cresce". O `RFC_005_API_E_APP.md` (D3) escreveu que o app é "o Tainá inteiro, não um utilitário de backup". Esta RFC é o que faz o app ser as duas coisas.
- **A caixa já está pronta.** A lacuna de API 1 da revisão ("sem endpoint de manifesto/delta de hashes, então backup em background é impossível") **já foi fechada**: a emenda do review 0.1.0 no `RFC_005_API_E_APP.md` (§7) acrescentou `/files/manifest` com paginação por `after` e ETag fraca, e o `sha256` já volta no fio. O que falta é o cliente.

O trabalho, então, é do lado do `okena`. A caixa precisa de pouquíssimo, e o pouco está na seção 6.

---

## 3. O que o iOS permite

Esta seção é a razão de a promessa ser o que é. Ela não é um detalhe de implementação, é o contrato com a pessoa que usa.

| Primitiva | O que entrega de verdade |
| --- | --- |
| Execução em primeiro plano | Sincroniza qualquer duração. É o cavalo de batalha. |
| `URLSessionConfiguration.background` | Termina um upload **já iniciado** com o app suspenso, e relança o app por `handleEventsForBackgroundURLSession`. Não enumera nem hasheia. |
| `BGProcessingTask` | O sistema agenda. Minutos, oportunista, de preferência na tomada. Ninguém controla quando. |
| `BGAppRefreshTask` | Uns 30 segundos, para o estado. Também agendado pelo sistema. |
| `beginBackgroundTask` | Uns 30 segundos de folga depois de ir para o fundo, para terminar um pensamento. |
| `PHPhotoLibraryChangeObserver` | Avisa de mudança na biblioteca, enquanto o app roda. Não acorda o app. |

A consequência, em uma frase: **o espelho converge, não é contínuo.** O que o app garante é que nenhuma foto fica de fora para sempre, e que a fila anda sozinha depois que o app foi aberto. O que ele não garante, porque o sistema não deixa, é que a foto tirada agora esteja na caixa em trinta segundos com o app fechado.

A interface diz isso com todas as letras, no tom da casa: "Sua biblioteca chega aqui aos poucos, mais rápido com o Tainá aberto. Fotos que você acabou de tirar podem demorar." Prometer paridade com o Apple Fotos seria mentir para uma comunidade não-técnica sobre uma coisa que ela percebe no primeiro dia.

---

## 4. Decisões

### D1: O espelho é unidirecional e para a casa

Do aparelho para a caixa, nunca o contrário. E todo upload nasce na **casa**. O espelho nunca publica na praça: a RFC 003 §2 faz da legibilidade da zona o próprio mecanismo de consentimento, e subir sozinho para a praça seria consentir no lugar da pessoa. O toque de publicar continua existindo, separado, no visualizador (o conserto da lacuna C7 da revisão).

### D2: A promessa é "cedo ou tarde", e a interface diz isso

O iOS não dá execução contínua em segundo plano. O app enumera e hasheia em primeiro plano, enfileira, e a fila de fundo drena. A tela de estado descreve o que está acontecendo em pt-BR, sem prometer o que o sistema não entrega (seção 9).

### D3: O índice local é a verdade do cliente; o manifesto é a verdade da caixa

O aparelho guarda um ledger do que já subiu; a caixa responde com `/files/manifest`, o conjunto de hashes legíveis. O trabalho é o diff. Nenhum dos dois é autoridade única: o ledger sobrevive a reinstalação e a fila, o manifesto sobrevive a outro aparelho e à reinstalação do app.

### D4: O trabalho de CPU é em primeiro plano; o de rede é da fila de fundo

Enumerar a biblioteca, ler bytes e calcular `sha256` são trabalho de primeiro plano, e ficam lá. O upload é da fila de fundo (`URLSessionConfiguration.background`), que sobrevive à suspensão. Essa divisão é o que faz o espelho convergir sem manter o app aberto.

### D5: O ledger existe para não re-hashear, não só para não re-subir

O detalhe que decide o custo. Para pular o que já subiu é preciso comparar o `sha256`, e calcular `sha256` exige ler o arquivo inteiro. O ledger guarda o hash por `localIdentifier` mais a `modificationDate`, então só assets novos ou editados são hasheados de novo. A primeira passada lê a biblioteca toda, uma vez; depois o custo é proporcional às fotos novas. Sem essa coluna, toda sincronização relê a biblioteca inteira.

### D6: Dedup por `sha256`, com consulta por hash na API

O cliente evita re-subir o que o manifesto já lista. Mas a resposta de um upload pode se perder (rede caiu entre o commit na caixa e a resposta), e aí o cliente não sabe se subiu. A mitigação é uma **consulta por hash**: `GET /api/v1/files/manifest?sha256=<hex>` responde se aquele hash já existe no conjunto legível da pessoa (seção 6). É uma adição pequena ao endpoint que já existe, sem regra de negócio nova.

### D7: Exclusão nunca propaga

Apagar no Apple Fotos **não** apaga na caixa. A caixa é o arquivo da comunidade; apagar sozinho contra um commons partilhado é um ato irreversível que ninguém pediu. O espelho converge por acréscimo. O preço honesto: o espelho desvia do rolo ao longo do tempo, e a interface não pode sugerir que os dois são idênticos.

O espelho da regra, que precisa do lado da caixa: um arquivo na lixeira **não** pode voltar a subir sozinho. Se o manifesto não falar do que está na lixeira, a próxima passada re-sobe o que a pessoa acabou de jogar fora, e a lixeira vira uma promessa quebrada. O manifesto precisa carregar o hash do que está na lixeira, ou o cliente precisa poder perguntar por ele (seção 6).

### D8: Edição vira arquivo novo, e a caixa não ganha "substituir" agora

Editar uma foto muda os bytes, então é outro `sha256` e outro arquivo. A caixa é endereçada por conteúdo, então o blob não duplica, mas o registro `File` sim: o mesmo nome, dois rkeys. Não há `replace` na API, e esta RFC não inventa um. Aceitar a duplicata é a escolha simples e honesta; o endpoint de substituição é assunto de outra RFC, com o gatilho em `Questões em aberto`.

### D9: A cota é um freio, não um erro

Um rolo inteiro estoura a cota da comunidade. A caixa já responde 409 `storage_quota_exceeded` e expõe `/storage/quota`. O espelho **pausa** e mostra o motivo em pt-BR, sem descartar nada em silêncio e sem apagar nada. Encher a caixa é decisão da comunidade, não do app; o app só para de empurrar e conta o que falta.

### D10: Wi-Fi por padrão

O padrão é só Wi-Fi. Subir um rolo inteiro na rede móvel de alguém é gastar o dinheiro dessa pessoa sem perguntar. Celular é opt-in explícito, e a tela diz o tamanho do que falta antes de ligar.

### D11: Tudo cai numa pasta só, na casa

O espelho cria (uma vez) uma pasta própria na casa, "Rolo de câmera", e tudo que ele subiu vai para lá. Assim o que veio do espelho se separa do que foi subido à mão, e desligar o espelho não mistura as duas coisas. Álbuns do Apple Fotos **não** viram pastas (não-objetivo, com gatilho).

### D12: O estado em segundo plano é o mesmo cofre local

"O app recarrega o estado em segundo plano" e "o espelho sabe o que já subiu" são o mesmo problema: um lugar no disco que sobrevive ao relançamento. Hoje o livro de ETags é em memória (o `ConditionalStore` "a relaunch reads the collections again"), então toda atualização de fundo seria uma releitura inteira. Persistir o livro de ETags e um cache de leitura das coleções torna a atualização de fundo quase de graça (a caixa responde 304 sem corpo), e é o que permite as telas abrirem do disco antes da rede.

### D13: O núcleo é puro; o app é dono do SwiftData e do Photos

O `OkenaCore` define o contrato e a política (o ledger como protocolo, o plano como função pura), sem saber o que é SwiftData nem o que é `PHPhotoLibrary`. O app é dono do container SwiftData e do adaptador de Fotos. É a mesma costura do `RFC_005_API_E_APP.md`: contrato no núcleo, casca no app.

### D14: Vocabulário

"Espelho" é a palavra do produto (pt-BR, familiar), distinta de "backup". O backup off-site da `RFC_007_BACKUP_CIFRADO.md` é a caixa para a nuvem; o espelho é o aparelho para a caixa. Na tela: "Espelhar o rolo de câmera", "Enviando", "Pausado", "Só no Wi-Fi". "Ledger" e "manifesto" são encanamento e não aparecem para quem usa.

---

## 5. O espelho: o fluxo

### O pipeline

1. **Autorização.** `PHPhotoLibrary.requestAuthorization` para leitura. O acesso pode ser total ou **limitado** (a pessoa escolhe um subconjunto); os dois casos valem, e o limitado pode crescer pelo `presentLimitedLibraryPicker`.
2. **Enumerar.** `PHAsset` da biblioteca, filtrado (não-objetivo: o que fica de fora). Cada asset vira um `LibraryAsset` (valor, `Sendable`): `localIdentifier`, `modificationDate`, tamanho, nome, mime, `taken_at`.
3. **Diff.** `SyncPlan` (função pura) cruza os assets com o ledger e com o conjunto de hashes do manifesto e devolve: `toHash`, `toUpload`, `toSkip`, `unchanged`. Nada de IO aqui.
4. **Hashear.** Só o que o plano mandou: materializar os bytes (de `PHAssetResource`, com `isNetworkAccessAllowed` conforme D10) e calcular `sha256`.
5. **Planejar de novo.** Com o hash em mãos, o que já está no manifesto vira `skip` e vai para o ledger. O que não está vira `upload`.
6. **Enfileirar e subir.** Cada upload vira uma tarefa da `URLSession` de fundo; o `taskIdentifier` volta para o ledger, para reconciliar depois de um relançamento.
7. **Registrar.** No sucesso, o ledger guarda `rkey`, `sha256`, tamanho e data. No fracasso, o item volta à fila com recuo exponencial, e a tela conta o que falta.

### O ledger

Uma linha por asset. Campos: `assetID` (o `localIdentifier`, único), `modificationDate` (o que o plano compara), `sha256`, `size`, `filename`, `mime`, `takenAt`, `state` (`new | hashed | queued | uploading | uploaded | skipped | failed`), `rkey`, `backgroundTaskID`, `attempts`, `lastError`, timestamps.

### O plano, em forma

```
plan(assets, ledger, knownHashes) ->
  for each asset, em ordem:
    record = ledger[asset.id]

    sem record                          -> toHash
    record.modificationDate == asset.modificationDate
                                        -> unchanged   (nada a fazer, o mais barato)
    record.sha256 in knownHashes        -> skip        (já está na caixa; só atualiza o ledger)
    record.sha256 presente, fora de knownHashes
                                        -> upload(record.sha256)   (já hasheado, falta subir)
    resto                               -> toHash      (a foto mudou, precisa ler os bytes)

  para cada hash calculado no passo anterior:
    hash in knownHashes                 -> skip
    senão                               -> upload
```

Toda a regra difícil vive aqui, testável com structs, sem tocar disco nem rede. É o mesmo lugar do `GalleryModel` para as telas: a decisão é pura, a casca só executa.

---

## 6. O que a caixa precisa mudar

Pouco, e nada de regra de negócio nova.

1. **HEIC (bloqueador, já em conserto).** O rolo de câmera do iPhone é HEIC por padrão. A `REVIEW_0.1.0.md` (C3) achou que HEIC não gera miniatura silenciosamente quando a imagem não tem `libheif`, e que a `Dockerfile` publicada não instala as bibliotecas que o NIF de imagem carrega. Um espelho que sobe milhares de HEIC para uma grade de quadrados em branco não serve para nada. O conserto já está em curso na integração do review 0.1.0 (`libglib2.0-0 libheif1 libjpeg62-turbo` na imagem). **O espelho não entra antes disso.**
2. **Consulta por hash no manifesto.** `GET /api/v1/files/manifest?sha256=<hex>`: filtro opcional que devolve a entrada daquele hash, ou vazio. É o que torna o upload seguro contra resposta perdida (D6), sem inventar idempotência em `POST /files` nem mexer na semântica de arquivo por pasta.
3. **A lixeira no manifesto.** O manifesto é "fora da lixeira". Se um arquivo vai para a lixeira, o hash dele sai do conjunto e o cliente re-sobe. O manifesto passa a carregar o hash do que está na lixeira da pessoa (um flag `trashed`, ou uma coleção à parte), para o espelho saber que o arquivo existe e que a pessoa o jogou fora (D7). Sem isso, a lixeira ressuscita.
4. **Teto de 2 GiB e resumível.** O `RFC_005_API_E_APP.md` (§8) tem teto multipart de 2 GiB e adia upload resumível com o gatilho "quando links instáveis derrubarem vídeos grandes". O rolo de câmera tem vídeos, então o gatilho se aproxima. Nesta RFC o resumível continua fora: a `URLSession` de fundo já re-tenta, e um vídeo que falha no fim recomeça do zero. Anotar o custo honesto e reabrir com o primeiro vídeo grande que falhar (Questões em aberto).

O resto já existe: `sha256` no fio, `taken_at`, `width`/`height`, estado de miniatura, operações em lote, ETag, `/storage/quota`, 409 de cota.

---

## 7. Atualização do estado em segundo plano

O segundo pedido, e o mesmo cofre do D12.

- O `ConditionalStore` deixa de ser em memória e passa a ser um livro de ETags **em disco**, por sessão e por coleção.
- Um **cache de leitura** das coleções (arquivos, fotos, mural, pessoas) mora no mesmo armazenamento local, para as telas abrirem do disco e a rede só confirmar.
- `BGAppRefreshTask` chama `me` mais as coleções de topo com `If-None-Match`. Como a caixa responde 304 sem corpo quando nada mudou, a atualização de fundo é quase de graça.
- O `poll` de 30 segundos que o `SessionModel` já faz em primeiro plano continua; a atualização de fundo é o que substitui o poll quando o app está suspenso.
- O que a atualização de fundo pode mudar: a identidade e a cota (o `me`), a contagem para o badge, e a validade da sessão (um 401 em segundo plano deixa a sentença para a tela de login, igual ao `screen/1`).

---

## 8. Fronteira: núcleo e app

**No núcleo (`OkenaCore`, puro).**

```
Sync/LibraryAsset.swift     valor Sendable: o asset, sem Photos
Sync/SyncLedger.swift       protocolo + implementação em memória (testes)
Sync/LibrarySource.swift    protocolo: enumerar e materializar bytes
Sync/SyncPlan.swift         puro: (assets, ledger, knownHashes) -> plano
Sync/SyncEngine.swift       actor: orquestra hash -> subir -> registrar
Transport/BackgroundTransport.swift  URLSession de fundo + delegate
```

`BackgroundTransport` é a mudança mais invasiva. O `URLSessionTransport` de hoje usa `URLSession.shared`, que não pode ser uma sessão de fundo; sessões de fundo são baseadas em delegate, e delegate não compõe bem com `async/await`, então a ponte é por continuações, mais um mapa `taskIdentifier` -> `assetID` no ledger e o `handleEventsForBackgroundURLSession` no app delegate para reconciliar no relançamento. O comentário do `Transport.swift` ("the only seam that touches the network") continua verdadeiro, e é por isso que a mudança merece fase própria e testes próprios.

**No app (`Okena`).**

```
Store/SyncRecord.swift        @Model, assetID único
Store/StoreActor.swift        @ModelActor; o único dono do ModelContext
Sync/PhotosLibrarySource.swift  PHPhotoLibrary + PHAssetResourceManager
Sync/SyncCoordinator.swift    @MainActor; cena, observer, BGTaskScheduler
Capture/{CameraPicker,LibraryPicker}.swift  câmera e seleção manual
Features/Sync/                a tela de estado
```

Duas disciplinas que o SwiftData exige com `SWIFT_STRICT_CONCURRENCY: complete`: o `@Model` não é `Sendable` e o `ModelContext` não é `Sendable`, então **um** `@ModelActor` é o único dono do contexto, e nada de callback de delegate de rede toca o container direto (o callback entrega ao actor). Entre atores, trafegam valores (`LibraryAsset`), nunca instâncias de `@Model`. O `@Attribute(.unique)` do `assetID` faz upsert, e isso se prova com teste antes de confiar.

---

## 9. Interface e vocabulário

Uma tela de estado, na linguagem da casa, sem jargão.

- **Ligado/desligado** do espelho, com o que ele significa em uma frase: "As fotos e os vídeos que você faz aparecem aqui, na sua casa. Só você decide o que vai para a praça."
- **Progresso**: "Enviando 12 de 340", "Faltam 328". Nunca uma barra que não corresponde a nada.
- **Pausas, com o motivo**: "Pausado, só no Wi-Fi" e "Pausado, a comunidade encheu o armazenamento". A segunda aponta para a saúde da comunidade, não para o app.
- **A honestidade do D2**, dita uma vez, no lugar certo: as fotos chegam aos poucos.
- **Nada de praça.** O espelho só fala de casa. O toque de publicar vive no visualizador, onde o conteúdo vive (conserto da C7), e continua sendo um ato da pessoa.

---

## 10. Sequência de build

1. **Câmera e galeria manuais.** `PhotosPicker` multi-seleção mais captura de câmera, reusando o `upload` que já existe, com fila, progresso e cancelar. Entrega valor sozinho e fecha a C5. **Portão:** subir uma foto da galeria e uma da câmera, as duas caem na casa.
2. **O cofre local.** `@Model` mais `StoreActor`, o ledger por trás do protocolo, o livro de ETags em disco e o cache de leitura. **Portão:** as telas abrem do disco em partida fria sem rede; um 304 sobrevive ao relançamento.
3. **O núcleo do espelho.** `LibraryAsset`, `SyncPlan`, `SyncEngine`, `PhotosLibrarySource`, só em primeiro plano. **Portão:** property tests do plano; um rolo de 500 fotos espelha, e a segunda passada não re-sobe nem re-hasheia nada.
4. **A fila de fundo.** `BackgroundTransport`, o mapa de `taskIdentifier`, a reconciliação no relançamento, o `handleEventsForBackgroundURLSession`. **Portão:** matar o app no meio de um upload, reabrir, a fila termina e nada duplica.
5. **O agendamento.** `BGAppRefreshTask` e `BGProcessingTask`, `UIBackgroundModes`, os identificadores permitidos. **Portão:** a atualização de fundo roda e atualiza o badge; o processamento drena quando o sistema concede.
6. **A tela de estado.** Progresso, pausas com motivo, a cópia honesta do D2. **Portão:** uma moradora não-técnica entende o que está acontecendo e por que parou.

As fases 1 e 2 não dependem do acesso ao rolo e podem andar antes de qualquer decisão de produto em aberto. As fases 3 e 4 são o risco concentrado.

---

## 11. Testes

- **Plano (puro):** property tests sobre `SyncPlan`. Asset novo hasheia; hash conhecido vira skip; `modificationDate` igual vira unchanged; hash calculado que já está no manifesto vira skip; a segunda passada não tem trabalho.
- **Ledger:** upsert pelo `assetID` único; sobrevive ao relançamento; um item em `uploading` sem `backgroundTaskID` volta à fila.
- **Idempotência:** matar o app entre o commit da caixa e a resposta, reabrir, e a consulta por hash (D6) evita a duplicata.
- **Lixeira:** um arquivo na lixeira não volta a subir (o manifesto o conhece); a lixeira continua sendo da dona.
- **Fotos:** autorização limitada funciona; um asset só no iCloud falha com a mensagem certa e volta à fila; `PHPhotoLibraryChangeObserver` dispara a passada certa.
- **Fundo:** o relançamento reconcilia as tarefas pendentes; um 401 em segundo plano deixa a sentença para o login.
- **Zona:** nenhum caminho do espelho produz um arquivo na praça.
- **Honestidade:** a tela mostra "pausado" com o motivo certo quando a cota estoura e quando não há Wi-Fi.

`mix precommit` é o portão de saída do lado da caixa; `swift test` no `OkenaCore`, do lado do app.

---

## 12. Não-objetivos e riscos

Não-objetivos, cada um com o gatilho de retorno:

- **Espelho bidirecional** (baixar o que está na caixa para o aparelho): outro produto, outro RFC.
- **Propagação de exclusão** (apagar na caixa apaga no aparelho, ou o inverso): decidido contra no D7.
- **Substituição de arquivo** (edição sem duplicata): gatilho é o ruído de duplicatas incomodar num piloto.
- **Upload resumível**: gatilho é o primeiro vídeo grande que falhar no fim.
- **Álbuns do Apple Fotos viram pastas**: fora; a pasta é uma só (D11). Gatilho: um piloto pedir organização por álbum.
- **E2E e cifra**: fora; a promessa da casa continua de software e confiança (RFC 003, D7).
- **Android**: a `RFC_005_API_E_APP.md` prevê Kotlin depois. O contrato desta RFC (manifesto, consulta por hash) serve os dois; a implementação Kotlin fica para quando o app Android existir.
- **Tempo real**: o espelho é polling e fila, como o v1 da API (RFC 005, D5).

Riscos:

1. **A expectativa de "contínuo".** O risco dominante, e é de produto, não de engenharia. Mitigação: a cópia do D2 e da seção 9, dita antes de a pessoa tirar a primeira foto.
2. **iCloud Fotos.** Com "Otimizar Armazenamento", os originais não estão no aparelho. O `PHAssetResourceManager` baixa da iCloud antes de subir, então a banda do espelho inclui um download da iCloud. Precisa de `isNetworkAccessAllowed` e de Wi-Fi (D10), e a tela não pode confundir "enviando" com "baixando".
3. **Cota da comunidade.** Um rolo inteiro estoura a caixa. É o incidente mais provável no mundo real, e é uma decisão de commons, não do app (D9). O teto por moradora é questão em aberto.
4. **Primeira passada é cara.** Hashear a biblioteca inteira lê tudo uma vez. Aceitável, mas a tela precisa explicar por que o primeiro espelho demora.
5. **HEIC.** Bloqueador, já em conserto (seção 6, C3). Sem ele, a grade nasce em branco.
6. **A sessão de fundo e o relançamento.** O `handleEventsForBackgroundURLSession` é o lugar clássico de bug sutil; por isso a fase 4 tem portão próprio.
7. **2 GiB.** Um vídeo longo estoura o teto multipart. Anotar, e reabrir com o resumível.

---

## 13. Relação com as outras RFCs

- **RFC 002 (MVP):** esta RFC entrega o item 2 do pós-MVP ("app nativo de backup de fotos"), o recurso que motivou o app nativo. A escolha de linguagem mudou desde a RFC 002 (que cogitava Flutter): o app implementado é Swift (`RFC_005_API_E_APP.md`, §2), e esta RFC o assume.
- **RFC 003 (governança e transparência):** as regras sociais são herdadas sem exceção. O espelho só escreve na casa (D1), nunca na praça, e a legibilidade da zona continua sendo o mecanismo de consentimento. Nada aqui esconde a casa do zelador; a privacidade honesta (D7) vale, e a interface não promete cadeado.
- **RFC 005 (API e app):** esta RFC consome o contrato e pede três adições pequenas (consulta por hash, lixeira no manifesto, HEIC na imagem). A emenda do review 0.1.0 (§7) já entregou o manifesto e o ETag; o resto é extensão do mesmo endpoint, sem regra nova.
- **RFC 006 (kernel):** nada. O espelho é cliente; o núcleo da caixa não muda.
- **RFC 007 (backup cifrado):** a seção 12 daquela RFC já separa as duas frentes: o backup **do** aparelho **para** a caixa (esta RFC) e o backup **da** caixa **para** a nuvem (aquela). Sem conflito. Se um dia o aparelho também guardar uma cópia, é assunto da RFC 007, não desta.

---

## 14. Questões em aberto

- **Live Photos.** Uma Live Photo é um HEIC mais um MOV. Sobe a foto só, ou os dois? A favor dos dois: não perder o movimento. Contra: dobra os arquivos e complica a grade. Recomendação de partida: só a foto no v1, o MOV fica para quando a grade souber mostrá-lo.
- **Filtro de enumeração.** Screenshots, fotos ocultas, "Apagadas Recentemente". Recomendação de partida: excluir ocultas e apagadas, incluir screenshots, e deixar a escolha visível.
- **Teto por moradora.** Um rolo inteiro pode encher a caixa de uma pessoa só. A comunidade decide se há teto por pessoa, ou se a cota é só da Tekoa e o espelho pausa (D9). É decisão de commons.
- **HEIC no cliente como escape.** Se alguma caixa não tiver `libheif`, o cliente converte para JPEG antes de subir? Perde qualidade e metadado, e é a caixa que deve se consertar (seção 6), mas é o plano B se um piloto tiver uma caixa sem a biblioteca.
- **Substituição de arquivo (D8).** Um endpoint `replace` na caixa resolve a duplicata de edição, mas mexe na semântica de conteúdo endereçado. Fica para outra RFC, com o gatilho de duplicatas incomodarem.
- **Um app para dois papéis.** O espelho é do aparelho de uma pessoa. Uma comunidade com um só aparelho de "arquivo" (um tablet da sede) usa o mesmo mecanismo, ou pede outra coisa? Provavelmente o mesmo, mas vale observar num piloto.
