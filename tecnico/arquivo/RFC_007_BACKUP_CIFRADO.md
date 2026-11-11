# RFC 007: Backup cifrado off-site

> **Proposta histórica.** O 1.0.0 adota um recorte menor: kits aleatórios equivalentes, sem Shamir ou limiar `k-de-n`. A referência vigente é a [arquitetura atual](../ARQUITETURA.md).

**Status:** Proposta
**Data:** 2026-10-22
**Emenda:** 2026-10-31, recuperação comunitária com limiar, versionamento de chaves, consistência do snapshot e restore verificado em staging
**Relação:** Deriva do `RFC_002_MVP.md` (o backup é a Fase 4, Nhaman-lite; o "off-site cifrado" ficou parkeado no pós-MVP, D10) e do `RFC_006_KERNEL.md` (onde mora o núcleo puro, onde mora o produto). Não altera o modelo social do `RFC_003_GOVERNANCA_E_TRANSPARENCIA.md`; reforça a privacidade honesta (D7) dizendo em voz alta o que esta RFC **não** promete.
**Autoria:** @zoedsoupe + revisão técnica

---

## 1. Resumo executivo

Uma comunidade sem homelab ainda precisa de uma cópia dos dados fora da caixa: corrupção de disco, roubo, incêndio. O destino barato e confiável é a nuvem de outra pessoa (Google, Apple, um bucket S3, um amigo com NAS). Hoje isso significa entregar a `casa` inteira, em texto puro, para uma big tech: o backup atual (`Taina.Nhaman.Backup`) é um `.tar.gz` sem cifra, e a própria documentação do módulo sugere "um remote sincronizado por rclone" como destino. Quem segue esse conselho já está subindo texto puro.

Esta RFC define o **backup cifrado off-site**: a caixa cifra, o provedor guarda o que não consegue ler. Cinco decisões de base.

1. A cifra acontece **na caixa**, com chaves derivadas pelo núcleo puro. A ameaça principal é um **terceiro** (a nuvem), então a caixa pode cifrar sozinha: sem app nativo, sem `okena`, sem o aparato do E2E.
2. O backup é um **cofre de objetos sincronizado por CID**, não um tarball monolítico. O Ybira já é endereçado por conteúdo; o incremental sai do diff do conjunto de CIDs, e uma foto grande não volta a subir a cada run.
3. **A recuperação tem que sobreviver à caixa sem depender de uma única pessoa.** A raiz de recuperação é um segredo aleatório dividido com limiar `k-de-n`. O desenho padrão é `2-de-3`: uma parte pode ser perdida sem impedir o restore, e uma pessoa sozinha não reconstrói o segredo.
4. **Um backup só termina quando o manifesto assinado é publicado, e um restore só substitui o estado depois de verificar tudo em staging.** Upload concluído não é sinônimo de recuperação testada.
5. **Metadados são minimizados no formato, não apenas encobertos na copy.** Conteúdo, manifestos e banco são cifrados; todo dado remoto vive em chunks de tamanho uniforme com padding; nomes, datas, hierarquia de pastas, tamanhos individuais e quantidade exata de arquivos não aparecem para o provedor. Volume agregado e tráfego de rede continuam visíveis.

O que esta RFC **não** é: E2E. Ela não esconde a `casa` de quem tem root na caixa. A promessa da `casa` continua sendo software e confiança (RFC 003, D7); o que muda é a promessa **para a nuvem**, e essa passa a ser criptográfica, e verdadeira: o provedor não recebe as chaves e não lê o conteúdo.

Também não se promete que as partes de recuperação "só funcionam dentro do Tainá". Quem reúne `k` partes recupera material capaz de decifrar o cofre, inclusive fora da interface. O produto não oferece exportação em texto puro, mas isso não pode ser apresentado como uma fronteira criptográfica.

A frase que vende: **a cifra é o que deixa uma comunidade pobre alugar infraestrutura de gente rica sem confiar nela.**

---

## 2. Motivação

O backup do MVP resolve "guardar uma cópia". Não resolve o destino. `Taina.Nhaman.Backup.run/1` monta um `.tar.gz` único (`db.dump` + a árvore de `storage/`, incluindo `storage/keys/`) e grava em `config :taina, :backup, dir:`. Quatro problemas:

- **Não é incremental.** Todo run rearquiva o conjunto inteiro. Numa comunidade com dezenas de GB de mídia, o run noturno reenvia tudo. Num uplink de casa isso não é viável.
- **Não é cifrado.** O `.tar.gz` sai em texto puro. Se `dir` é um remote rclone, a `casa`, a `praça` e a chave privada da Tekoa vão em claro para o provedor.
- **A proposta anterior de recuperação tinha pessoa única de falha.** Uma senha da zeladoria concentra perda, esquecimento e poder de decifração na mesma pessoa. Isso não combina com a operação coletiva da comunidade.
- **Snapshot e restore precisam de atomicidade de conjunto.** Um snapshot SQLite e uma listagem posterior do disco não formam necessariamente um ponto consistente. No caminho inverso, decifrar sobre o estado vivo pode deixar banco e storage de versões diferentes depois de uma falha.

A estrutura que torna o backup incremental, endereçamento por conteúdo, também torna a cifra limpa. A recuperação comunitária e a verificação em staging resolvem as outras duas falhas sem mudar o modelo de hospedagem.

E a assimetria que importa: ao contrário do E2E, aqui a ameaça principal é um **terceiro** que a comunidade não escolheu e não governa. Não há assembleia que remova o Google. Esse é um cenário onde a promessa criptográfica é honesta e vale o preço.

---

## 3. Modelo de ameaça e limites da promessa

| Ameaça | Defesa | Cobertura |
| --- | --- | --- |
| Provedor lendo o conteúdo do backup | AEAD na caixa, sem chaves no provedor | **sim** |
| Perda por corrupção, incêndio ou roubo da caixa | cópia off-site e kits fora da caixa | **sim** |
| Provedor alterando um objeto ou bloco | hash no manifesto, assinatura e AEAD | **sim, detectado** |
| Provedor apagando um objeto necessário | conjunto obrigatório no manifesto e preflight do restore | **sim, detectado; não impede a perda** |
| Provedor revertendo todo o cofre para um histórico válido | manifestos encadeados e âncora local no mural | **parcial** |
| Uma pessoa custodiante abrindo o cofre sozinha | limiar `k >= 2` no desenho padrão | **sim** |
| `k` ou mais custodiantes decifrando fora do Tainá | nenhuma; elas têm a capacidade de recuperação | **não** |
| Perda de uma parte de recuperação | redundância do `2-de-3` | **sim** |
| Caixa destruída e partes restantes abaixo do limiar | nenhuma; não há escrow | **não, dado irrecuperável** |
| Morador lendo a casa alheia depois do restore | regras de leitura restauradas com o banco | **sim, pela aplicação** |
| Root na caixa lendo dados ou chaves locais | cifra em repouso/E2E seriam necessários | **não** |
| Provedor reconstruindo nomes, árvore, tamanhos individuais ou número exato de arquivos | chunks uniformes com padding e manifestos cifrados | **sim, pelo formato** |
| Provedor observando volume agregado e horário/volume do tráfego | exigiria tráfego constante e decoys, fora do primeiro corte | **não** |

Quatro honestidades em voz alta:

- **A caixa continua vendo tudo.** Esta RFC não muda o "trusted host" da RFC 003. A zeladoria não ganha um atalho de leitura na aplicação, mas quem tem root na máquina continua podendo acessar o texto puro local.
- **Custódia de recuperação é confiança real.** O limiar impede uma pessoa sozinha, não um grupo que atinja o limiar. Custodiantes não viram zeladores por isso, mas precisam ser escolhidas como pessoas de confiança para a continuidade da comunidade.
- **A cifra em repouso do disco local é outro assunto.** O cofre off-site protege a cópia na nuvem. O disco da caixa segue em texto puro até uma solução como LUKS ser adotada.
- **Minimizar metadados não é tornar o tráfego invisível.** O provedor vê a quantidade de chunks armazenados, o volume total aproximado, os horários e os bytes enviados ou removidos em cada conexão. Ele não recebe o mapa que liga chunks a arquivos, pastas, pessoas ou snapshots.

---

## 4. Decisões

### D1: A cifra acontece na caixa, não no cliente

O escopo é a nuvem como adversária. A caixa detém a chave e cifra antes de subir. Diferente do E2E, aqui **não** há chave no cliente e **não** há participação do `okena`. O backup cifrado não fica preso ao app nativo.

### D2: O backup é um cofre de objetos endereçado por conteúdo

`Ypy.BlobStore.Disk` já endereça cada blob por CID. O run sincroniza o **conjunto de CIDs referenciado pelo snapshot** contra o cofre, cifrando e subindo só o que falta. O incremental é consequência da estrutura, não de um algoritmo de delta.

O conjunto não é obtido por uma listagem independente do disco depois do snapshot. Essa listagem poderia incluir objetos irrelevantes e, pior, perder objetos apagados entre as duas operações. A D14 define a retenção temporária necessária.

### D3: O nome remoto é `HMAC(chave, CID || chunk)`, nunca o CID cru

O CID, hoje, **é** o SHA-256 do texto puro. Nomear o objeto remoto com ele vaza um hash do conteúdo para a nuvem, adivinhável e correlacionável entre usuários. Cada chunk remoto recebe `HMAC(k_name, cid || chunk_index)`: estável dentro de uma versão de chave, então ainda dá para saber o que já subiu, e não ligável ao conteúdo sem a chave.

Os nomes ficam num namespace plano. Não há diretório por arquivo, prefixo comum entre chunks do mesmo CID, timestamp ou extensão que distinga conteúdo, banco, manifesto e pacote de chaves. Sem o manifesto decifrado, o provedor não consegue agrupar chunks em arquivos ou snapshots.

Dedup por nome continua apenas dentro do cofre e sob a mesma versão de chave. Nonce e naming determinísticos por chunk permitem escrita idempotente, mas não há cifra convergente global nem correlação entre Tekoas: cada cofre tem suas próprias chaves. O host deter as chaves é exatamente o que a RFC 003 D7 prevê para este recorte.

### D4: Envelope com chave-mestra e segredo de recuperação aleatório

Cada versão de dados usa uma chave-mestra `M` de 32 bytes gerada por `:crypto.strong_rand_bytes/1`. `M` deriva as subchaves de cifra e naming.

A recuperação não usa uma senha humana. O setup gera um segundo segredo `R`, também de 32 bytes, e deriva uma KEK por HKDF-SHA256 com salt e `info` próprios. A KEK embrulha por AEAD a chave-mestra, ou o keyring descrito na D13, no header do cofre.

Essa separação tem três efeitos:

- não existe senha fraca para ataque offline por GPU;
- trocar o grupo de recuperação não exige decifrar os objetos na caixa;
- uma rotação pode emitir uma nova chave de dados sem reescrever imediatamente todo o histórico.

O `header` autenticado pela KEK carrega a `tekoa_pub` dentro do payload cifrado. Alterar a chave pública, o limiar, a versão ou qualquer parâmetro do header faz o unwrap falhar. Fora do AEAD ficam somente os parâmetros mínimos para localizar o formato e derivar a KEK.

### D5: O manifesto assinado e cifrado é a fonte da verdade do restore

A nuvem é não confiável também para integridade. O manifesto lista os chunks esperados, seus hashes de texto cifrado, versões de chave, o snapshot do banco, o pacote de chaves da Tekoa e o manifesto anterior (`prev`). Ele é assinado pela chave da Tekoa e depois cifrado. Assinatura protege autenticidade; AEAD protege confidencialidade e integridade do transporte.

O restore usa a listagem do provedor apenas como dica de descoberta. A lista de chunks obrigatórios e o mapa para arquivos vêm do manifesto decifrado e verificado. Chunk ausente, hash divergente, assinatura inválida ou elo `prev` quebrado falha fechada. O provedor não lê datas, tamanhos, CIDs, caminhos ou quantidade de arquivos no manifesto.

`prev` torna adulterações detectáveis, mas não impede um provedor de apresentar um prefixo antigo inteiro e válido. A D6 fornece uma âncora enquanto a caixa antiga existe; depois da perda total da caixa, rollback completo continua sendo uma limitação explícita.

### D6: Backup da instância vira notícia no mural, nunca segredos

Transparência (RFC 003, seção 6) sobre a coisa que a zeladoria opera: todo backup concluído da instância vira uma **notícia legível no mural**, não apenas um registro operacional escondido. A comunidade vê que a cópia foi feita, quando terminou e para qual destino, sem precisar abrir o painel da zeladoria. Falhas depois de esgotadas as tentativas, setup, confirmação de custódia, rotação, perda de kit e restore também produzem notícias próprias.

A notícia registra somente metadados: ids de cofre, keyset e manifesto, destino, estado e horários. O limiar e as pessoas custodiantes aparecem nas cerimônias em que isso importa. Nunca aparecem partes, kits, chaves, tokens do provedor, headers completos ou erros que contenham segredos. Tentativas automáticas intermediárias não viram uma sequência de notícias repetidas.

O hash do manifesto no mural serve como âncora de frescor local. Uma entrada de backup criada depois do snapshot não precisa estar contida no próprio snapshot; ela entra na cadeia local e será capturada por um run posterior.

Backup pessoal da `casa`, quando existir, segue a regra oposta: não vira notícia social no mural. Essa separação impede que a transparência sobre a operação da instância vire vigilância sobre a portabilidade individual.

### D7: A cripto é grátis; o Conecta vende conveniência

O plano parkeava o "off-site cifrado" como recurso pago do "Tainá Conecta" (RFC 002, D10). Mas o cadeado **é** a garantia de soberania: cobrar por ele contradiz a tese. A cifra é livre e parte do box. O Conecta vende o que é caro e chato, como destino já configurado e relay, nunca o cadeado.

### D8: OTP para as primitivas; threshold sem criptografia caseira

A caixa é um Pi. HKDF, HMAC e ChaCha20-Poly1305 usam `:crypto`. O streaming processa bloco a bloco para nunca segurar um blob grande na memória. Nada de `age`, `openssl` ou binário externo para cifrar.

Shamir Secret Sharing não faz parte da OTP. A implementação `k-de-n` deve usar uma biblioteca pequena, mantida, compatível com a BEAM e com ARM, com vetores publicados e testes independentes. A dependência precisa ser justificada no `mix.exs`, e não pode exigir NIF se houver alternativa pura adequada.

Não se implementa Shamir ou outro compartilhamento de segredo manualmente no Ypy. Se nenhuma dependência passar na revisão, o limiar fica bloqueado; o fallback `1-de-n` pode ser discutido, mas não se inventa uma construção própria para cumprir cronograma.

### D9: Recuperação comunitária `k-de-n`, com `2-de-3` como padrão

O segredo `R` é dividido em `n` partes com limiar `k`. O formato guarda `k` e `n`, mas a UI oferece poucos presets para não transformar recuperação em uma calculadora de risco.

O desenho padrão é **três custodiantes e duas partes necessárias**:

- uma parte perdida não inviabiliza a recuperação;
- uma pessoa sozinha não abre o cofre;
- duas pessoas precisam cooperar num desastre.

Para comunidades menores, o modelo de dados aceita outros presets. `1-de-2` oferece redundância, mas cada pessoa pode abrir sozinha, e a UI precisa dizer isso. `2-de-2` não é recomendado porque não tolera perda nenhuma. O caso de comunidade com uma única pessoa não ganha falsa proteção comunitária: usa um único kit e recebe o aviso explícito de ponto único de falha.

Custodiantes são moradores ativos escolhidos para a continuidade. O papel não dá permissão administrativa, não altera `casa`/`praça` e não precisa coincidir com zeladoria. A zeladoria coordena o setup, mas não deve ver todas as partes.

A caixa guarda o keyring local para o run agendado não precisar de humano. Não existe escrow, recuperação por suporte ou chave da Zeetech. Com a caixa destruída e menos de `k` partes válidas, o dado é irrecuperável.

### D10: O backup local continua; o cofre é um destino a mais

O `.tar.gz` para USB não morre. O run pode ter dois destinos: o arquivo local, humano e simples, e o cofre cifrado off-site. Nada de substituição silenciosa.

O estado da tela precisa distinguir os destinos. Um USB em dia não pode fazer a interface dizer que a cópia externa está em dia.

### D11: Vocabulário

`Replica` é o encanamento (módulo, inglês, RFC 003 D5). "Cofre externo" ou "cópia externa protegida" é o vocabulário que a pessoa lê.

"Custodiante" pode aparecer em documentação técnica; na UI prefere-se "pessoa que guarda uma parte da recuperação". "Kit de recuperação" é o material impresso ou guardado pela pessoa. "Parte" é o segredo individual. Nenhum nome Tupi-Guarani aparece na interface.

### D12: Restore pelo Tainá é o fluxo de produto, não uma prisão criptográfica

O cofre é pensado para ser restaurado por uma instalação nova do Tainá. A UI não oferece visualização ou exportação do conteúdo decifrado, e o restore recoloca as permissões originais.

Ainda assim, `k` partes recuperam material suficiente para decifrar dados fora da aplicação. Esconder esse fato seria security through obscurity. A documentação técnica e a cerimônia precisam tratar o grupo como capaz de abrir o backup inteiro, sem transformar essa capacidade em permissão social para ler a casa alheia.

### D13: Custodiantes e chaves têm versões; revogação não é retroativa

Cada keyset de recuperação e cada chave-mestra têm ids e versões. Objetos, meta e manifestos declaram qual versão usam.

Uma rotação real faz duas coisas:

1. cria uma nova chave-mestra para os dados novos;
2. cria um novo segredo `R`, novas partes e um novo keyset de recuperação.

O novo header embrulha um **keyring** com a nova mestra e as mestras antigas ainda necessárias para restaurar os snapshots retidos. Assim, o grupo atual consegue restaurar o histórico completo. O keyring local da caixa segue a mesma regra.

Somente reembrulhar a mesma mestra com partes novas **não** revoga o acesso de um grupo antigo aos dados futuros. Por isso, troca de custodiante com necessidade de revogação exige também nova chave de dados.

As consequências são explícitas:

- partes antigas não abrem objetos criados com a nova mestra;
- partes antigas continuam abrindo os snapshots e objetos que já conseguiam abrir;
- objetos antigos reutilizados por manifestos novos continuam sob a mestra antiga;
- não há como retirar acesso de quem já copiou ciphertext e partes válidas;
- reencriptar o histórico pode ser oferecido no futuro por economia de keyrings, mas não cria revogação retroativa.

A desativação de uma custodiante gera pendência de rotação, não rotação automática. Uma nova cerimônia é necessária. Se houver suspeita de comprometimento, novos runs devem ser pausados até o novo keyset estar ativo; numa simples perda sem suspeita, a política pode manter o destino atual marcado como "atenção".

### D14: Snapshot e blobs formam um ponto consistente

O run precisa garantir que todo blob referenciado pelo banco esteja disponível até o upload terminar.

A sequência de consistência é:

1. impedir dois runs do mesmo cofre em paralelo;
2. criar o snapshot SQLite com `VACUUM INTO`;
3. derivar do snapshot o conjunto de CIDs necessários;
4. reter ou fixar esses blobs contra purga até o fim do run;
5. ler e subir exatamente esse conjunto;
6. liberar a retenção depois do manifesto publicado ou da falha do run.

Uploads e atualizações normais podem continuar. Objetos novos criados depois do snapshot podem ficar para o próximo run. Purgas e GC não podem remover um objeto fixado antes da publicação do manifesto.

Um run que falha deixa chunks remotos órfãos no pior caso. Ele nunca move `current` para um manifesto que aponte para um chunk cuja escrita não terminou.

### D15: Restore autentica tudo em staging antes de substituir o estado

Nenhum dado vivo é apagado no começo do restore.

O restore segue esta ordem:

1. abrir o keyset com `k` partes distintas;
2. verificar `tekoa_pub`, header e keyring;
3. escolher e verificar um manifesto;
4. calcular espaço necessário e destino de staging;
5. baixar meta, chaves e todos os chunks obrigatórios para staging;
6. verificar hashes, framing AEAD, padding, último chunk e conjunto completo;
7. validar o banco restaurado e a referência de cada CID;
8. somente então parar a aplicação e trocar banco e storage;
9. subir a aplicação e verificar saúde e permissões.

Falha antes da troca deixa a instância original intacta. Falha durante a troca exige backup do estado anterior ou operação de recuperação pelo CLI. O primeiro corte oferece restore pela UI somente no primeiro boot de uma caixa nova; restore sobre uma instância existente continua no CLI com aviso destrutivo.

### D16: O provedor vê chunks uniformes, não arquivos

O formato minimiza metadados por construção:

- blobs, snapshot SQLite, pacote de chaves e manifestos são divididos em chunks de um único tamanho;
- o último chunk de cada sequência recebe padding antes da cifra, de modo que todo chunk remoto tenha exatamente o mesmo tamanho;
- o tamanho real e o comprimento do padding ficam dentro do framing cifrado;
- chunks de origens diferentes ocupam o mesmo namespace plano e usam o mesmo framing externo;
- nomes remotos não carregam data, CID, username, caminho, extensão ou contador legível;
- manifestos, incluindo `stamp`, lista de arquivos e `prev`, são cifrados;
- `tekoa_pub`, limiar, datas e histórico de rotação ficam dentro do payload cifrado do header.

Com isso, uma listagem completa do bucket não revela a árvore de pastas, o nome ou tamanho de um arquivo, a quantidade exata de arquivos, quais chunks pertencem ao mesmo arquivo ou quais pertencem ao banco.

O que permanece visível é o resíduo operacional: número total de chunks, volume agregado aproximado, quantidade de keysets, horários de acesso, volume de cada sync e retenção ou remoção de chunks. Como os nomes são estáveis, o provedor também pode reconhecer um chunk que apareceu pela primeira vez em determinado run, mas não sabe a que conteúdo ou arquivo ele pertence.

Esconder até esse resíduo exigiria uploads de enchimento, tráfego constante e uma política de banda/custo própria. Essa extensão fica fora do primeiro corte e não pode ser prometida pela UI.

---

## 5. Formato do cofre

```text
replica/
|-- header/
|   `-- <keyset_id>.cbor          # mínimo para derivar a KEK e abrir o payload
|-- current/
|   `-- <keyset_id>.bin           # ponteiro cifrado para o manifesto mais recente
`-- chunks/
    `-- <hmac>                    # chunk cifrado, imutável e de tamanho uniforme
```

Headers e chunks endereçados são imutáveis. O único objeto sobrescrito é `current/<keyset_id>.bin`, atualizado depois que o novo manifesto e todos os seus chunks já estão remotos. Esse ponteiro é uma dica de descoberta, não uma fonte de verdade: o restore ainda decifra e verifica o manifesto e sua cadeia.

Snapshots anteriores são percorridos pelos links `prev` decifrados. Não existem diretórios remotos chamados `files`, `meta`, `keys` ou `manifest`, nem datas nos nomes.

### 5.1 Header e keyring

`header/<keyset_id>.cbor` usa DAG-CBOR. Seu envelope externo contém somente:

- `format_version`;
- `wrap`: algoritmo HKDF, salt e domínio de derivação da KEK;
- `wrapped_payload`.

O payload cifrado e autenticado contém:

- `vault_id`;
- `keyset_id`;
- `created_at`;
- `threshold`: `{k, n}`;
- `keyring`: mapa `key_version -> chave-mestra`;
- `tekoa_pub`;
- `previous_keyset`, quando houver rotação.

O envelope externo inteiro, exceto o próprio ciphertext, entra como associated data do AEAD. Trocar versão, salt ou parâmetros invalida o unwrap. Limiar, data, chave pública e histórico não ficam visíveis para o provedor.

O keyring contém apenas mestras necessárias aos snapshots ainda restauráveis. Um keyset novo precisa abrir todas as versões referenciadas pelo manifesto atual e pelos snapshots que a política de retenção preserva.

### 5.2 Kits de recuperação

Cada kit codifica:

- `format_version`;
- `vault_id`;
- `keyset_id`;
- `share_id`;
- `k` e `n`;
- fingerprint da `tekoa_pub`;
- bytes da parte de Shamir;
- checksum.

A representação humana usa grupos de caracteres com alfabeto sem ambiguidade e checksum. QR é opcional e nunca substitui a forma impressa/legível. O kit identifica a comunidade sem incluir nomes de arquivos ou conteúdo do banco.

Partes de keysets diferentes, cofres diferentes ou limiares incompatíveis são rejeitadas antes da tentativa de restore. A interface também rejeita a mesma `share_id` duas vezes.

### 5.3 Chunks uniformes e padding

Todo conteúdo remoto, seja blob, SQLite, pacote de chaves, manifesto ou ponteiro `current`, usa o mesmo tamanho de chunk e o mesmo framing externo. O v1 usa chunks de 4 MiB; mudar esse tamanho exige nova `format_version`.

O cabeçalho externo de um chunk contém somente `format_version` e o nonce necessário para abrir o AEAD. O payload cifrado contém:

- `plaintext_length`;
- `content_class`;
- `chunk_index` e `last_chunk`;
- os bytes úteis;
- padding aleatório até o tamanho fixo.

O objeto remoto completo, incluindo cabeçalho, ciphertext e tag, tem sempre o mesmo tamanho. Na decifra, `plaintext_length`, índices, marcador final e extensão do padding são validados; qualquer inconsistência falha fechada.

Cada chunk é uma unidade AEAD independente. Associated data liga o cabeçalho externo, `vault_id`, nome remoto, `key_version`, índice e marcador de último chunk. O manifesto define a ordem e a quantidade de chunks de cada sequência, de modo que remoção, reordenação ou repetição também é detectada.

O nonce de 12 bytes é `HMAC-SHA256(k_nonce, chunk_name)` truncado, e o chunk name já liga chave, CID/conteúdo e índice. Assim, repetir a escrita do mesmo chunk produz exatamente o mesmo ciphertext, o que torna o upload idempotente. O invariante obrigatório é: uma combinação `k_nonce + chunk_name` nunca protege dois plaintexts diferentes. Uma colisão de CID ou mudança de conteúdo sob o mesmo nome é erro fatal, não motivo para reutilizar nonce.

AEAD: ChaCha20-Poly1305. Memória limitada por um pequeno número de chunks e buffers de IO, nunca pelo tamanho total do blob ou do banco.

### 5.4 Nomes remotos

Para blobs:

```text
chunk_name = base32(HMAC-SHA256(k_name, "blob" || cid || chunk_index))
```

Para componentes de um snapshot:

```text
chunk_name = base32(HMAC-SHA256(k_name, content_class || snapshot_id || chunk_index))
```

`snapshot_id` é aleatório, não um timestamp. `content_class` distingue meta, chaves, manifesto e ponteiro apenas na entrada do HMAC e no associated data; ela não aparece no caminho remoto. Não há galho compartilhado entre chunks do mesmo conteúdo.

A versão da chave de naming está no manifesto cifrado. A listagem externa mostra somente nomes opacos e uniformes sob `chunks/`.

### 5.5 Subchaves

Para cada `key_version`:

- `k_enc = HKDF-SHA256(M, info="taina-backup-v1/enc")`;
- `k_name = HKDF-SHA256(M, info="taina-backup-v1/name")`;
- `k_nonce = HKDF-SHA256(M, info="taina-backup-v1/nonce")`.

A KEK usa domínio separado, derivado de `R`, nunca de `M`. Nenhuma subchave é reutilizada para outro propósito.

### 5.6 Manifesto cifrado e assinado

O manifesto contém, no mínimo:

```text
{
  format_version,
  vault_id,
  snapshot_id,
  stamp,
  prev: {manifest_hash, chunks},
  keyset_id,
  blobs: [{cid, bytes, chunks: [{name, ciphertext_hash, key_version}]}],
  meta: {bytes, chunks: [{name, ciphertext_hash, key_version}]},
  keys: {bytes, chunks: [{name, ciphertext_hash, key_version}]},
  tekoa_pub,
  sig
}
```

O CID, tamanhos, data, mapa de chunks e todas as referências ficam somente nesse documento cifrado. `prev` permite localizar o manifesto anterior sem expor esse caminho na listagem do provedor.

A assinatura secp256k1 cobre os bytes CBOR canônicos do manifesto, como nos records. O documento assinado é então cifrado e dividido em chunks. O hash de ciphertext permite verificar cada download antes de decifrar; o AEAD continua obrigatório e não é substituído pelo hash.

`current/<keyset_id>.bin` contém, cifrado, o `snapshot_id`, os chunks do manifesto atual e seu hash. Ele só é substituído depois que esses chunks podem ser lidos e verificados no destino.

### 5.7 Meta e chaves da Tekoa

`meta` é o snapshot SQLite (`VACUUM INTO`) cifrado, preenchido e dividido em chunks como qualquer outro conteúdo. Ele muda a cada run e não participa do incremental de blobs.

`keys` contém o material privado necessário para a Tekoa recuperada continuar assinando como a mesma comunidade. O pacote segue exatamente o mesmo framing dos demais chunks.

O backup **não** precisa incluir o keyring local de backup nem credenciais do provedor. O keyring vem dos kits; as credenciais de acesso ao cofre são informadas novamente no restore. Excluir esses segredos evita autorreferência e reduz o valor do pacote de chaves.

---

## 6. Cerimônias de recuperação

### 6.1 Setup e distribuição

O setup conduz a zeladoria por destino, proteção e grupo de recuperação. Depois da escolha do grupo:

1. a caixa gera mestra, segredo `R` e partes;
2. cada custodiante acessa somente a própria parte em sessão autenticada;
3. a pessoa imprime ou guarda o kit em gerenciador de senhas;
4. a confirmação exige redigitar o código completo, não apenas marcar "guardei";
5. depois da confirmação, a parte não volta a ser exibida;
6. o keyset só fica ativo quando todas as `n` pessoas tiverem confirmado.

Enquanto a cerimônia está pendente, partes ainda não entregues ficam em registro temporário com expiração, cifradas pela mestra local. Isso evita segredos em texto puro no banco, mas não é apresentado como defesa contra root. Após confirmação ou expiração, o material temporário é apagado. Uma expiração antes da ativação invalida a cerimônia e exige gerar um novo keyset.

A caixa persiste somente metadados de custódia: `ava_id`, `share_id`, fingerprint da parte, estado, confirmação e versão. Fingerprints não são usados como segredo nem expostos publicamente.

Exigir todas as confirmações antes da ativação evita marcar o cofre como protegido quando a redundância ainda não existe. O primeiro upload pode ser preparado, mas o destino permanece em "configuração pendente" até a cerimônia terminar.

### 6.2 Verificação e drill

Confirmar que uma pessoa digitou o kit prova apenas que ela teve acesso a ele naquele momento. Não prova que o papel continua guardado.

O estado "recuperação testada" exige um drill real:

- abrir o keyset com pelo menos `k` kits;
- verificar um manifesto;
- restaurar para banco e storage descartáveis;
- conferir marcadores de banco, arquivos e chaves;
- descartar o staging sem tocar a instância viva.

Um teste de conexão com o provedor ou uma simples decifra do header não recebe esse estado. O drill completo pode exigir espaço temporário considerável e precisa informar isso antes de começar.

### 6.3 Perda de kit

Com a caixa viva, perda de todos os kits não perde os dados: a caixa ainda tem o keyring e pode conduzir uma nova cerimônia. A comunidade deve rotacionar o grupo assim que perceber a perda.

Com a caixa destruída:

- `k` ou mais partes válidas permitem recuperar;
- menos de `k` partes tornam o cofre irrecuperável;
- suporte, Zeetech e provedor não possuem caminho alternativo.

Se a parte perdida pode ter sido encontrada por outra pessoa, trata-se como comprometimento e não apenas como esquecimento.

### 6.4 Troca ou saída de custodiante

A zeladoria inicia uma nova cerimônia com o grupo atualizado. Enquanto ela está pendente, a Saúde mostra "grupo de recuperação precisa ser atualizado". Depois da ativação:

- novos dados usam uma nova mestra;
- o novo keyring preserva as mestras necessárias ao histórico retido;
- os novos kits substituem os antigos como caminho recomendado de recuperação;
- os kits antigos continuam limitados aos dados que já conseguiam abrir.

A UI não diz "acesso revogado" sem qualificar: "novos backups usam um novo grupo; cópias antigas continuam protegidas pelas regras antigas".

### 6.5 Restore numa caixa nova

O restore entra no setup de primeiro boot, protegido pelo mesmo limite local ou token de bootstrap da instalação. Não pode existir um endpoint público sem autenticação para aceitar partes de recuperação.

Fluxo:

1. instalar o Tainá e escolher "Recuperar uma comunidade";
2. informar destino e credenciais do cofre;
3. testar acesso sem gravar nada;
4. selecionar o keyset quando houver mais de um;
5. receber `k` partes distintas, uma pessoa por vez;
6. abrir o keyring somente em memória e validar o header;
7. listar snapshots válidos com data e tamanho;
8. escolher o ponto de restauração;
9. verificar espaço e restaurar tudo em staging;
10. trocar o estado vazio da caixa e iniciar a comunidade;
11. apagar as partes da memória de trabalho e nunca registrá-las.

A BEAM não oferece garantia simples de zeroização de toda cópia de binário. A implementação deve reduzir cópias e tempo de vida dos segredos, mas a documentação não promete apagamento físico da memória.

Contas, permissões e nomes voltam do snapshot. Não se cria uma nova zeladoria com poder sobre o backup recuperado.

---

## 7. Run, sync e restore

### 7.1 Run

`Taina.Nhaman.Backup.run/1` ganha um destino cifrado. A sequência completa:

1. obter a trava do cofre e criar o registro do run;
2. fazer snapshot do banco;
3. derivar e fixar os CIDs referenciados;
4. ler o último manifesto válido e calcular o diff de chunks;
5. cifrar, preencher e subir apenas os chunks ausentes;
6. cifrar e subir o snapshot e o pacote de chaves da Tekoa;
7. montar, assinar, cifrar e publicar os chunks do manifesto;
8. verificar a leitura remota e só então atualizar o ponteiro `current`, que é o commit do run;
9. marcar o run como concluído e anexar a notícia no mural;
10. liberar os blobs fixados e apagar arquivos temporários.

A escrita de um chunk precisa ser idempotente. Em filesystem, usa-se arquivo temporário e rename atômico. Em S3, a conclusão do `put` do chunk imutável é o ponto de publicação. Manifestos versionados nunca são sobrescritos; apenas `current` é substituído, por escrita temporária e troca atômica quando o adaptador permitir.

Uma falha antes do passo 8 deixa o snapshot anterior como atual. Um retry pode reutilizar chunks remotos cujos hashes e tamanhos estejam corretos. Um run sem mudanças pode publicar apenas nova meta/manifesto quando necessário, sem reenviar chunks de blobs.

### 7.2 Estado e observabilidade

`last_backup/0` passa a responder por destino. A UI não faz chamada de rede síncrona no mount: ela lê runs e verificações persistidos. Jobs separados atualizam alcance do provedor, último sucesso, última falha, bytes enviados e estado do drill.

Logs estruturados usam ids de run, cofre, keyset e manifesto, contadores e razões de erro. Não incluem chaves, partes, credenciais, conteúdo de kits ou URLs assinadas com segredos.

### 7.3 Restore

O restore não confia no relógio ou no nome do objeto para escolher o snapshot. `stamp` é informação assinada, mas o provedor pode esconder manifestos recentes. Quando houver dúvida, a UI permite escolher entre os snapshots válidos encontrados e mostra suas datas.

Depois da verificação em staging, o banco é aberto para checagem de integridade e as referências de blobs são comparadas com o conjunto restaurado. Só então acontece a troca.

`mix taina.backup.verify` ganha a variante "restaura do cofre para banco e storage descartáveis". Ela cobre kits, unwrap do keyring, assinatura, download, decifra e validação final. O drill "caixa morreu" não usa nenhum arquivo da instalação anterior além do executável do Tainá.

---

## 8. Fronteira: kernel e produto

### 8.1 No kernel, Ypy puro

`Ypy.Backup` contém:

- tipos e validações de keyset, kit, header, manifesto e keyring;
- HKDF, naming por HMAC e embrulho do envelope;
- máquina de estados pura para cifra/decifra AEAD em blocos;
- integração com a implementação revisada de Shamir;
- encoding e decoding dos kits;
- CBOR canônico e verificação estrutural do manifesto.

O núcleo não lê config, não escolhe provedor, não abre Repo e não conhece LiveView.

`Ypy.Replica` é a quarta behaviour, no espírito de `BlobStore` e `KeyStore`. O contrato cobre `put`, `get`, `stat`/`list` e `delete`, com caminhos ou streams para que blobs grandes não precisem virar binários inteiros. O núcleo conhece objetos, não buckets nem OAuth.

### 8.2 No produto, Tainá e Nhaman

`Taina.Nhaman.Backup` orquestra snapshot, pins, sync, restore e staging. `Taina.Nhaman.Replica.Disk` fecha o vertical local. `Taina.Nhaman.Replica.S3` é o primeiro adaptador remoto. Uma pasta sincronizada por rclone é uma forma de operar `.Disk`, não uma cifra separada.

O Oban agenda runs e drills, com exclusividade por cofre. A config escolhe destinos. A UI conduz setup, kits, estados e recuperação.

O produto persiste, no mínimo:

- destinos e configuração não secreta;
- runs e resultados de verificação;
- keysets e versões;
- custodiantes, fingerprints e confirmações;
- último drill completo.

Keyrings locais e credenciais do provedor ficam sob `storage/keys/` com modo `0600`, fora de logs e fora do pacote de chaves da Tekoa. Não se afirma que isso protege contra root.

---

## 9. Fluxo de UI e recortes de Penpot

A tela "Saúde da comunidade" continua sendo um resumo. O detalhe da cópia externa fica em uma tela própria, sem transformar o painel em console administrativo.

### 9.1 Permissões

- todo morador vê o estado geral da cópia e quem cuida da máquina;
- somente zelador configura destino, dispara run e inicia rotação;
- cada custodiante vê somente a própria cerimônia e o próprio kit pendente;
- nenhum papel vê todas as partes depois da ativação;
- o primeiro boot de recuperação usa o fluxo local/bootstrap, não uma sessão de morador.

### 9.2 Boards necessários

1. **Saúde da comunidade:** cópia em dia, primeira cópia em andamento, atrasada, destino inacessível, configuração pendente e recuperação testada.
2. **Configuração:** destino, explicação da proteção, escolha do grupo e revisão.
3. **Cerimônia coletiva:** kits pendentes/confirmados para a zeladoria, sem segredos; visão individual para cada custodiante.
4. **Kit individual:** imprimir/guardar, confirmar, expiração e perda.
5. **Primeira sincronização:** progresso, retry, rede interrompida e conclusão.
6. **Recuperação no primeiro boot:** destino, entrada das partes, erro de parte, escolha do snapshot, staging e conclusão.
7. **Rotação:** saída/perda de custodiante, novo grupo, aviso de histórico antigo e conclusão.
8. **Estados de erro:** cofre adulterado, objetos ausentes, espaço insuficiente, partes insuficientes e keyset incompatível.

Os fluxos críticos precisam de mobile e desktop: kit individual e recuperação em caixa nova. A implementação LiveView começa depois desses boards estarem verificados no Penpot.

### 9.3 Limites da interface

Não existe botão "baixar arquivos descriptografados". Não existe restore destrutivo de um clique numa instância viva. "Backup concluído" e "recuperação testada" são estados separados. A copy pode dizer que o provedor não lê conteúdo, nomes, pastas ou tamanhos individuais. Ela não diz que o provedor "não sabe nada": volume aproximado e horários de sincronização continuam visíveis.

---

## 10. Sequência de build

1. **Núcleo criptográfico:** HKDF, AEAD em chunks uniformes, padding, envelope, keyring, kits e threshold. Portão: golden tests, properties e vetores, sem Repo e sem disco.
2. **Formato e `Replica.Disk`:** headers versionados, namespace plano, sync incremental por chunk, manifesto cifrado e `current` como commit e retomada. Portão: ciclo put/list/restore com crash injection e inspeção da listagem sem chaves.
3. **Consistência do run:** snapshot, conjunto referenciado e pins contra purga. Portão: purge concorrente não gera manifesto incompleto.
4. **Restore em staging:** keyring, verificação integral e troca somente no final. Portão: restaurar para banco e storage vazios usando somente cofre e kits.
5. **Operação:** Oban, histórico, mural, logs, CLI e variante cifrada de `mix taina.backup.verify`. Portão: drill "caixa morreu" no CI e em VM descartável.
6. **Penpot e LiveViews:** setup, cerimônia, estados, recuperação e rotação. Portão: pessoa não técnica conclui setup e drill sem comandos manuais.
7. **Provedores e retenção:** S3 compatível, smoke test real, guia rclone e política segura de retenção. Portão: retry real de rede e restore a partir do provedor.

Os passos 1 a 5 não dependem de provedor e formam um vertical fechado sobre `Replica.Disk`. Rotação de keyset precisa entrar antes de comunidades reais dependerem do recurso, ainda que a UI possa vir depois do primeiro vertical.

---

## 11. Testes

### 11.1 Criptografia e kits

- round-trip de chunk, blob inteiro e keyring;
- borda de chunk, blob vazio e blob grande com memória limitada;
- todo chunk remoto tem exatamente o mesmo tamanho, inclusive meta, manifesto e último chunk;
- padding e `plaintext_length` inválidos são rejeitados;
- adulteração, remoção, repetição, reordenação e truncamento de sequência são detectados;
- retry do mesmo chunk reproduz o mesmo ciphertext, e nonce nunca é reutilizado para plaintexts diferentes;
- domínio de associated data validado;
- qualquer combinação de `k` partes abre; `k-1` não abre;
- parte repetida, malformada, de outro cofre ou de outro keyset é rejeitada;
- alteração de qualquer campo do header quebra o unwrap;
- fingerprints e checksums detectam erro de transcrição sem vazar a parte.

### 11.2 Sync e consistência

- diff sobe apenas os chunks que faltam;
- run sem mudança não reenvia chunks de blobs;
- nome por HMAC é estável dentro da versão;
- crash entre cada etapa deixa somente o `current` anterior como commit;
- retry reutiliza chunks válidos e não publica duplicatas de manifesto;
- purga concorrente não remove blob fixado;
- upload posterior ao snapshot não entra retroativamente no conjunto;
- dois runs do mesmo cofre não movem `current` concorrentemente.

### 11.3 Manifesto e restore

- assinatura inválida é rejeitada;
- manifesto não pode ser lido antes da decifra;
- `prev` quebrado é detectado;
- chunk ausente ou divergente falha antes da troca;
- banco inválido ou referência sem blob falha antes da troca;
- restore a partir de cofre e kits, sem keyring local antigo;
- restore preserva contas, permissões de `casa`/`praça` e chave da Tekoa;
- staging é descartado depois de sucesso e de falha;
- espaço insuficiente não toca a instância original.

### 11.4 Minimização de metadados

- uma listagem crua do destino não contém nomes de arquivos, usernames, CIDs, caminhos ou timestamps;
- blobs, meta, chaves e manifestos são indistinguíveis pelo framing e tamanho externos;
- chunks do mesmo arquivo não compartilham prefixo ou diretório;
- tamanho individual e número exato de arquivos não podem ser derivados da listagem;
- header externo não expõe `tekoa_pub`, limiar, data ou histórico de rotação;
- testes de golden view documentam o resíduo ainda visível: quantidade de chunks, volume e horários das operações.

### 11.5 Rotação

- novo grupo abre o histórico retido;
- grupo antigo não abre objetos de uma nova mestra;
- grupo antigo continua abrindo somente o histórico anterior;
- manifesto com versões mistas restaura corretamente;
- keyset pendente não vira fonte de recuperação ativa;
- desativação de custodiante gera pendência visível.

### 11.6 Produto e operação

- cofre desatualizado aparece como desatualizado;
- cópia local em dia não mascara cópia externa atrasada;
- cerimônia incompleta aparece como pendência;
- drill parcial não recebe estado "recuperação testada";
- morador não acessa configuração ou kit alheio;
- logs, mural e erros não contêm segredos;
- backup concluído publica exatamente uma notícia legível no mural, com retries intermediários silenciosos;
- falha depois de esgotadas as tentativas publica uma notícia de falha;
- restore no primeiro boot não fica exposto como endpoint público.

`mix precommit` é o portão de saída. O drill com pessoa não técnica continua sendo portão de produto, não substituído pelo CI.

---

## 12. Não objetivos e riscos

### 12.1 Não objetivos

- **E2E**, esconder a `casa` de root na caixa: outro problema e outra RFC.
- **Cifra em repouso do disco local:** disciplina de sistema, como LUKS no install.
- **Backup de Tekoa para Tekoa:** alinhado à soberania, mas exige segunda caixa; gatilho é federação ou piloto com comunidades parceiras.
- **Escrow de chave:** nunca. Contradiz a recuperação soberana.
- **Impedir tecnicamente que `k` custodiantes decifrem fora do Tainá:** incompatível com entregar a elas a raiz de recuperação.
- **Esconder volume agregado e padrão temporal do tráfego:** exigiria decoys, uploads constantes e orçamento de banda próprio. O formato já esconde os metadados internos; não promete invisibilidade operacional.
- **Integração iCloud nativa universal:** não há caminho geral equivalente ao S3 para a caixa Linux.
- **GC completo no primeiro vertical:** inicialmente objetos imutáveis podem ser retidos; retenção segura entra antes de operação prolongada em provedor pago.

### 12.2 Riscos

1. **Partes abaixo do limiar com a caixa destruída significam dado perdido.** Mitigação: `2-de-3`, cerimônia obrigatória, drill e pendência visível.
2. **`k` custodiantes podem abrir todo o backup.** Mitigação de produto: escolha consciente e copy honesta; não há mitigação criptográfica sem mudar o modelo.
3. **Banda do uplink.** O primeiro seed pode levar dias. Incremental, retomada e progresso visível reduzem o custo dos runs seguintes.
4. **Rollback completo é melhor esforço.** O mural ajuda enquanto a caixa antiga existe, mas não resolve sozinho o desastre total. Uma âncora independente fica em aberto.
5. **APIs e contas de provedores mudam.** S3 compatível é o primeiro caminho; rclone é o escape operacional para outros destinos.
6. **Keyring e credenciais ficam na caixa.** Quem tem root já tem os dados, mas esses arquivos continuam sendo alvos de cuidado, permissão `0600` e exclusão de logs.
7. **Biblioteca de threshold vira supply chain crítico.** Revisão, vetores, testes e atualização controlada são obrigatórios.
8. **Restore e drill exigem espaço temporário.** Preflight precisa considerar staging e evitar encher o disco da caixa.
9. **Retenção pode aumentar custo indefinidamente.** Não apagar nada é seguro para o primeiro vertical, mas não é política final de operação.
10. **Rotação mal explicada cria falsa sensação de revogação.** UI e runbook precisam separar dados futuros de histórico já acessível.
11. **Padding aumenta armazenamento e chunks aumentam chamadas ao provedor.** O tamanho fixo precisa ser medido num Pi e num destino real; a minimização de metadados não pode tornar o backup economicamente inviável.
12. **Volume e horários continuam correlacionáveis.** Mesmo sem ler o manifesto, um provedor que observa sucessivos runs pode inferir atividade agregada da comunidade. A documentação precisa manter esse limite visível.

---

## 13. Relação com as outras RFCs

- **RFC 002 (MVP):** esta RFC antecipa e reescopa o "backup criptografado off-site" que a D10 parkeou no Conecta. A Fase 4, Nhaman-lite, cresce; o `.tar.gz` local continua.
- **RFC 003 (governança e transparência):** a D7, trusted host, fica intacta. A recuperação comunitária distribui continuidade sem criar um novo papel administrativo. Mural e logs estendem a transparência sem publicar segredos. A cifra nunca é convergente.
- **RFC 006 (kernel):** adiciona `Ypy.Backup` e a behaviour `Ypy.Replica`. A regra de mão única vale: o núcleo conhece objetos e transformações puras, não buckets, credenciais ou UI.
- **RFC 005 (API e app):** a seção 8 devolve um `sha256` para o cliente de backup pular o que já subiu. Aquilo é backup **do aparelho para a caixa**. Esta RFC é backup **da caixa para a nuvem**. São frentes opostas e independentes.
- **RFC 009 (backup pessoal):** a cópia da instância vira notícia no mural; a cópia pessoal da `casa` não. Os formatos e as recuperações são independentes: a instância usa chunks e custódia comunitária, a pessoa usa um pacote legível e kit individual.

---

## 14. Questões em aberto

- **Biblioteca de Shamir:** qual implementação passa pelos critérios de manutenção, ARM, ausência de NIF e vetores independentes?
- **Âncora de frescor:** como detectar rollback completo depois que a caixa e seu mural sumiram? Um segundo destino independente, checkpoints guardados pelas custodiantes ou outra construção precisam ser avaliados sem criar telemetria externa.
- **Presets para comunidades pequenas:** quais combinações além de `2-de-3` entram na primeira UI, e como explicar `1-de-n` sem prometer cooperação?
- **Código comum entre `.tar.gz` local e cofre:** fazer USB como `Replica.Disk` simplifica sync, mas perde o arquivo humano único. Os dois formatos podem continuar separados.
- **Drive nativo:** OAuth e manutenção do adaptador compensam frente a uma pasta sincronizada por rclone? iCloud fica fora dessa promessa.
- **Retenção:** quantos snapshots manter e como remover chunks não referenciados sem race com runs, restores e rotações?
- **Custo do padding e dos chunks:** 4 MiB é o ponto de partida do v1, mas precisa de benchmark em Pi, uplink doméstico e provedor com cobrança por requisição antes de congelar a `format_version`.
- **Drill periódico:** qual frequência é útil sem transformar transparência em notificação constante, e como reservar espaço para o restore descartável?
