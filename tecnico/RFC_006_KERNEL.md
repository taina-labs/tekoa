# RFC 006: Ypy, o kernel do Taina

**Status:** Aceita
**Data:** 2026-10-12
**Relacao:** Cria a biblioteca Ypy. Deriva da RFC_002 (deploy e instancia), da RFC_003 (modelo social, governanca, nomes) e da RFC_005 (API e app), e emenda as tres nos pontos da secao 16.
**Autoria:** @zoedsoupe + revisao tecnica

---

## 1. Resumo executivo

O Taina nasceu monolito e funciona: identidade, armazenamento, modelo social, UI e API convivem em `taina`. Mas tres produtos vao usar o mesmo nucleo: o box da comunidade (o produto principal), a plataforma hospedada (Taina Conecta) e a Universidade (RFC_004). Sem um corte, cada produto reimplementa o nucleo, e a divergencia vira retrabalho.

Esta RFC extrai o nucleo estavel para uma biblioteca propria, **Ypy** (Tupi: origem). Decisoes de base:

1. **Ypy e biblioteca, nao rede.** Ela carrega o protocolo (schema e formas), nao implementa transporte. Sem HTTP, sem processos, sem config, sem env.
2. **Um perfil de deploy, nao dois.** O box roda SQLite, uma Tekoa por caixa, sem RLS. O hospedado nao e "Postgres multi-tenant": e uma frota de caixas por Tekoa, iguais ao box, atras de um control plane proprietario. Postgres sai do produto.
3. **Identidade e `did:web` por Tekoa**, derivada do dominio da comunidade, sem PLC. Moradores sao handles (`nome@tekoa`, RFC_003 D9).
4. **Federacao e v2**, formas do ATProto sem a rede global, praca-only.
5. **Licenca MPL-2.0**, para que um produto proprietario (Conecta) possa depender do kernel.

---

## 2. Por que um kernel, e onde fica a fronteira

Um kernel so se paga quando existe mais de um consumidor. Aqui existem: o box, o Conecta e a Universidade compartilham identidade, records, zonas, papeis, mural e assembleia. O que muda entre eles e o deploy, o transporte, a UI e a superficie de produto. Extrair o que e comum e deixar o que muda e o corte desta RFC.

A fronteira e de mao unica: **um produto depende de Ypy; Ypy nao depende de produto nenhum.** Ypy nao le config de produto, nao nomeia modulo de produto e nao conhece collection id de produto.

**Dentro do Ypy (puro + schema):** identidade (namespace Maraca: handle, `did:web`, chaves, credenciais), records (namespace Ybira: envelope assinado e endereçado por conteudo, `BlobStore` plugavel), a politica social pura (zonas, papeis, a regra de leitura), o mural (cadeia append-only assinada), a assembleia, os lexicons, os components OpenAPI e os schemas Ecto com as migracoes.

**Fora do Ypy (adapter ou produto):** a escolha de persistencia (o box usa SQLite), o HTTP e a API (RFC_005 vive no box), o transporte e os fluxos de autenticacao (convite, login, sessao, reset, device token), a UI, o transporte de federacao e as superficies de produto (Jaci).

**Behaviour propria so onde o Ecto nao alcanca:** `BlobStore`, `KeyStore` e `Ops`. Nao se inventa behaviour de persistencia: o Ecto ja e esse seam. Nao se inventa behaviour por contexto: as assinaturas das funcoes do kernel sao o contrato.

---

## 3. Nome e vocabulario

A convencao de nomes e a da RFC_003 secao 5, aplicada sem excecao:

1. **Tupi-Guarani** para nome proprio de subsistema/marca: **Ypy** (o kernel), Maraca, Ybira, Jaci, Tekoa.
2. **pt-BR familiar** para o vocabulario social, uma fonte de verdade (atomo == termo do contexto == rotulo na UI): `:casa`, `:praca`, `:zelador`, `:morador`, mural, pedido, concessao, assembleia, proposta, voto.
3. **Ingles** so para o encanamento invisivel: `Did`, `Cid`, `Scope`, `Wire`, `Layout`, `Record`.

**Idioma dos docs:** README, `@moduledoc` e `@doc` em pt-BR, a lingua padrao do Taina; ingles e opcional. O codigo em si (identificadores) segue a regra acima: ingles no encanamento, pt-BR no vocabulario social.

Ypy e o codigo; Taina e o produto. Se "Ypy" vazar para texto que a pessoa le, o objetivo se perde.

---

## 4. As tres camadas e o layout de modulos

Dependencia so aponta para baixo.

**Nucleo puro (zero I/O: sem processo, DB, clock ou aleatoriedade, exceto o que e passado como argumento):** `Ypy.Cbor`, `Ypy.Cid`, `Ypy.Varint`, `Ypy.Base32`, `Ypy.Tid`, `Ypy.Secp256k1`, `Ypy.Handle`, `Ypy.Did`, `Ypy.PublicId`, `Ypy.Lexicon` (+ `Validate`, `Registry`), `Ypy.Record`, `Ypy.Mural` (matematica da cadeia: seal, verify_entry, verify_chain), `Ypy.Maraca.Policy`, `Ypy.Maraca.Keys`, `Ypy.Maraca.Credentials`, `Ypy.Assembleia.Policy`, `Ypy.Scope`, `Ypy.Layout`.

**Shell de persistencia (Ecto e o seam; toda funcao recebe `repo` explicito):** schemas `Ypy.Maraca.{Tekoa,Ava,Concessao,Pedido,TekoaKey}`, `Ypy.Mural.Entry`, `Ypy.Ybira.{File,Folder,Record,Blob}`, `Ypy.Assembleia.{Proposta,Voto}`; os `queries.ex` de cada namespace (com `tekoa_id` explicito em todo `WHERE`); os `store.ex` (repo como primeiro argumento; `Ypy.Mural.Store.append/4` faz o encadeamento com retry).

**Contextos compostos (sem transporte):** `Ypy.Maraca`, `Ypy.Ybira`, `Ypy.Mural`, `Ypy.Assembleia`.

**Contratos de efeito (as unicas behaviours) e impls de referencia:** `Ypy.BlobStore` (+ `Ypy.BlobStore.Disk`), `Ypy.KeyStore` (+ `Ypy.KeyStore.Files`), `Ypy.Ops` (contrato de driver de host, forma copiada do Pesque).

**Camada de schema/fio:** `Ypy.Wire` (+ shapes por recurso e `Ypy.Wire.Errors`), `api/openapi.components.yaml`, `priv/lexicons/*.json`, `priv/ypy/migrations/*.exs`, `lib/mix/tasks/ypy.install.ex`.

**Supervisao: nenhuma.** O kernel nao inicia processo, nao tem `Application`, nao le env, nao abre arquivo no boot. O unico estado global e o registro de lexicons em `:persistent_term`, escrito uma vez pelo produto no boot e somente lido depois.

---

## 5. As tres behaviours

```elixir
defmodule Ypy.BlobStore do
  @callback put(t(), binary()) :: {:ok, Ypy.Cid.t()} | {:error, term()}
  @callback fetch(t(), Ypy.Cid.t()) :: {:ok, binary()} | {:error, :not_found}
  @callback stat(t(), Ypy.Cid.t()) :: {:ok, %{size: pos_integer()}} | {:error, :not_found}
  @callback open(t(), Ypy.Cid.t()) :: {:ok, Path.t()} | {:error, :not_found | :unsupported}
  @callback delete(t(), Ypy.Cid.t()) :: :ok | {:error, :not_found}
end

defmodule Ypy.KeyStore do
  @callback generate(t()) :: {:ok, Ypy.Maraca.Keys.keypair()} | {:error, term()}
  @callback sign(t(), tekoa_id :: integer(), binary()) :: {:ok, binary()} | {:error, :not_found | term()}
  @callback fetch_public(t(), tekoa_id :: integer()) :: {:ok, binary()} | {:error, :not_found}
  @callback rotate(t(), tekoa_id :: integer(), Ypy.Maraca.Keys.keypair()) :: :ok | {:error, term()}
end

defmodule Ypy.Ops do
  # Contrato de driver de host. Impls vivem no produto (box: compose; appliance:
  # systemd/nix). Exit codes: 0 ok, 1 falha de dominio, 2 uso; driver soma 3 (driver/env).
  @callback data_dir() :: {:ok, String.t() | {:volume, String.t()}} | {:error, term()}
  @callback run_once(RunOnce.t()) :: :ok | {:error, term()}
  @callback up() :: :ok | {:error, term()}
  @callback down() :: :ok | {:error, term()}
  @callback restart() :: :ok | {:error, term()}
  @callback pull(nil | String.t()) :: :ok | {:error, term()}
  @callback logs(keyword()) :: :ok | {:error, term()}
  @callback health() :: {:ok, :healthy | :degraded} | {:error, term()}
  @callback interactive?() :: boolean()
  @callback owns_lifecycle?() :: boolean()
end
```

A verificacao pura nunca passa por behaviour: `Ypy.Maraca.Keys.verify/3` recebe dado de chave publica, entao verificar uma cadeia (exportacao, federacao) e codigo do nucleo.

**Kernel entrega os verbos que operam sobre estado do kernel; o produto entrega os verbos que operam sobre o deploy.** No kernel: `migrate`, `backup`/`restore` (`VACUUM INTO` + tar sobre `Ypy.Layout`), e os verbos de identidade do Maraca. No produto: `setup`, `update`, o ciclo de vida (`up`/`down`/`restart`/`pull`/`logs`/`health`) e o `doctor` de federacao. Os host drivers (compose/systemd/nix) sao do CLI, nao da lib.

---

## 6. A camada de schema: lexicons, record e OpenAPI

**As formas de record sao lexicons** (JSON no formato ATProto), guardados em `priv/lexicons/`, com os nucleos `ypy.file.v0` e `ypy.folder.v0`. Produtos estendem dropando os proprios lexicons no proprio `priv/lexicons/` e passando os dois diretorios para `Ypy.Lexicon.Registry.load/1` no boot. O kernel nunca aprende collection id de produto. A validacao e `Ypy.Lexicon.Validate`, pura e total, com caminho por erro.

**O envelope** (`Ypy.Record`) e um mapa com `$type`, `rkey` (TID), `actor` (handle), `created_at` (microssegundos, inteiro, para determinismo do CBOR) e o payload validado, **aninhado sob a chave `payload`** (para que um campo do payload chamado `rkey`/`actor` nao colida com metadado do envelope). `encode/1` produz bytes DAG-CBOR canonicos e o CIDv1 (chaves ordenadas por tamanho depois bytewise, floats 64-bit, link tag 42). A chave da Tekoa assina os bytes; a assinatura mora na entrada da cadeia, nao no envelope, porque envelopes sao endereçados e copiados, e assinaturas sao fatos ordenados pela cadeia. Na cadeia, o `prev` e o CID da entrada **completa** anterior (incluindo a assinatura), o que sela tambem a ordem. Para o record de arquivo, o payload carrega `name`, `folder`, `mime_type`, `size_bytes` e `blob` (CID), de modo que a cadeia seja auto-descritiva e o walk de export/verify reconstrua o arquivo; `zona` fica na coluna tipada local, porque casa nao federa.

**Portabilidade de uma Tekoa** e `Ypy.Layout` (db + blobs/ + keys/) mais o caminho da cadeia: todo estado commitado, dado ou social, e uma entrada selada; todo envelope e alcancavel a partir da sua entrada; todo blob, do seu CID. Exportar e verificar sem processo nenhum e sem rede.

**OpenAPI:** o kernel entrega `api/openapi.components.yaml` so com components (schemas, enums de zona/papel/status, o envelope de erro `{"error": ...}`, o envelope de paginacao `{"items", "next_cursor"}`, formatos de cid/rkey). Os paths sao do box (RFC_005 secao 9). Um teste de conformidade do kernel serializa uma amostra por lexicon e afirma que todo campo do lexicon aparece como propriedade de component com o tipo mapeado; e o guarda contra deriva silenciosa entre o mundo CBOR (dados) e o mundo HTTP JSON (fio).

---

## 7. Persistencia

**Uma biblioteca nao pode ser dona de um Repo.** O kernel nao define `Ypy.Repo`, nao le config global, nao usa `Repo.with_tekoa` nem o dicionario de processo. Em vez disso: schemas, changesets e query builders sao modulos sem repo; **toda funcao de store recebe `repo` como primeiro argumento**; as migracoes sao copiadas para o produto por `mix ypy.install` e rodadas pelo `ecto.migrate` do produto. Explicito, sem global, e o nucleo puro nunca ve um repo.

**Tabelas flat, sem schema Postgres** (SQLite nao tem schema): `maraca_tekoas`, `maraca_avas`, `maraca_concessoes`, `maraca_pedidos`, `maraca_tekoa_keys`, `maraca_mural`, `ybira_files`, `ybira_folders`, `ybira_records`, `ybira_blobs`, `assembleia_propostas`, `assembleia_votos`. Toda tabela com escopo de Tekoa carrega `tekoa_id`, e toda query do kernel filtra `tekoa_id` explicitamente.

**Regras de migracao do kernel:** sem `CREATE SCHEMA`, sem RLS, sem prefixo de schema, sem indice Postgres-only, sem operador jsonb em query, `:map` para payload, `:utc_datetime` para tempo, `:id` surrogate mais public id string.

**O produto acrescenta** as proprias migracoes, sem tocar a forma das tabelas do kernel (adicionar indice e permitido): o box acrescenta `maraca_device_tokens` (RFC_005 secao 4), renditions, tabelas `jaci_*`, `oban_*` e a trava de uma-Tekoa-por-box (que e politica do box, nao invariante do kernel: o kernel permite N Tekoa, o Conecta precisa de N).

---

## 8. A cadeia unica

O mural e os commits de record sao **uma so cadeia por Tekoa**: cada entrada carrega `seq`, `prev` (CID da anterior), payload, e uma assinatura da chave da Tekoa. Ato social (convite, publicar-na-praca, concessao, mudanca de papel) e put de arquivo sao entradas da mesma cadeia.

Isso torna literal o que a RFC_003 secao 2 ja dizia: **colocar algo na praca E o ato de consentimento**, agora um fato assinado, encadeado e portavel. O mural legivel e uma projecao da cadeia (filtro por tipo de entrada), e a portabilidade de record e a outra projecao. Resolve a duplicacao entre `maraca.events` e qualquer log futuro de commit.

Serializacao de escrita e do banco: SQLite serializa escritor por natureza; o ponto de serializacao no Postgres (se um dia existir) e o indice unico em `(tekoa_id, seq)` com retry limitado em `Ypy.Mural.Store.append` (3 tentativas, depois `{:error, :chain_conflict}`). O append da cadeia acontece **na mesma transacao** da escrita do dado, entao nao existe buraco possivel entre o dado e a cadeia; o retry de conflito reexecuta a transacao inteira. Nao ha processo por repo.

---

## 9. O que fica no kernel e o que fica no produto

**No kernel:** os substantivos e invariantes do Maraca (colunas de ava pendente, predicados de expiracao de convite/reset, hash e verificacao de senha como funcoes puras), os verbos sociais do Ybira (upload, get_file, list, lixeira, publicar/tirar-da-praca, move/rename, cada um anexando uma entrada na cadeia), o mural e a assembleia.

**No produto:** os fluxos de autenticacao e UX (`invite_user`, `authenticate` com bcrypt e rate limit, sessao, entrega de reset, device token da RFC_005 secao 4). A superficie `Taina.Maraca` (40 callbacks) nao vira kernel inteira: isso recriaria o box dentro da biblioteca. As behaviours `Maraca.Behaviour` e `Ybira.Behaviour` do box se aposentam; as assinaturas do kernel mais os testes sob sandbox as substituem.

---

## 10. Como os produtos consomem (mao unica)

Regra: produtos dependem de `{:ypy, "~> 0.1"}`; o Ypy nao importa nada de produto, nao le config de produto e nao nomeia modulo de produto.

**Box Taina** (primeiro consumidor): `Taina.Repo` passa para `ecto_sqlite3` (WAL, busy_timeout). `mix ypy.install` traz as migracoes do kernel. Os contexts viram composicao fina: `Taina.Ybira.upload/3` chama `Ypy.Ybira.upload(Taina.Repo, Taina.Blobs, scope, tmp, opts)`. Mantem toda a UI LiveView, sessoes/convites/reset, rate limit, workers Oban, o painel, o driver compose de `Ypy.Ops` e `TainaWeb.FileServing`. Serve o documento DID em `/.well-known/did.json`, montado puro por `Ypy.Did.document/2`.

**Taina Conecta** (hospedado, proprietario): app Phoenix proprio; **uma caixa por Tekoa** (mesmo SQLite, mesmo kernel), orquestrada por um control plane que consome artefatos publicados (imagem, verbos, `/health`) e nao importa codigo do Taina. Nunca re-serve a API do box; faz relay, tunel e backup off-site (RFC_002 D10). A co-localizacao de varias Tekoas nao reintroduz Postgres nem RLS: a isolacao e o container e o volume.

**Taina University** (RFC_004, adiada): mesmo padrao; acrescenta os proprios lexicons e tabelas.

---

## 11. Seguranca: RLS fora do kernel

As policies de RLS no repo atual sao estritamente `tekoa_id`-scoped: **nunca defenderam casa/praca nem dono.** Zona e dono sempre foram app-layer (`Maraca.authorize?` mais filtro de zona, RFC_003 D2). Logo, tirar o RLS nao custa privacidade intra-Tekoa em nenhum perfil, e a regra de leitura ja vive onde deve.

Com uma Tekoa por caixa, leitura cruzada entre Tekoa nao existe por construcao, e com hospedado por containers a isolacao e o container/volume. O RLS fica estacionado (tag no git mais uma pagina de doc), nao apagado. O que o box perde e a segunda camada de enforcement do escopo; compensa mantendo a invariante de escopo na app (uma funcao de leitura testada uma vez, RFC_003 D2) e uma suite que varre "morador B nao le nenhuma casa de A" em toda superficie.

---

## 12. Identidade: did:web, TID e nomes

- **`did:web` por Tekoa**, derivado do dominio da comunidade. `did:plc` rejeitado: mintar escreve no `plc.directory`, ledger publico estrangeiro, o que e "telefonar para casa, publicamente, no nascimento" e viola a RFC_003 D6. Sem PLC.

**O documento DID e DID Core puro, sem o perfil ATProto.** O identificador e `did:web:<dominio>`, com `verificationMethod` multikey. Em particular: o fragmento da chave de assinatura **nao** usa a convencao `#atproto` do ATProto, nao ha service `#atproto_pds` nem `alsoKnownAs` com `at://`. Esses elementos anunciariam um PDS que nao servimos e fariam um resolvedor que conhece ATProto tratar o dominio como PDS e falhar. O fragmento e da propria Tekoa: `#tekoa`.
- **`rkey` (TID) e o id publico de arquivo**, substituindo o `nanoid public_id`. Um id so, ordenavel e unico. Ciente: o TID e ordenavel por tempo, entao o id revela quando o item foi criado (aceitavel na praca).
- **Nomes sociais aplicam a RFC_003 D5 por completo:** `Concessao`, `Proposta`, `Voto`, e os atomos `:casa`/`:praca`/`:zelador`/`:morador`, iguais no banco, no contexto e na UI.

---

## 13. Decisoes

- **D1:** Ypy e biblioteca; sem HTTP, processos, config ou env; carrega o protocolo (schema e formas).
- **D2:** Tres camadas (nucleo puro, shell de persistencia, contextos compostos) com dependencia so para baixo.
- **D3:** So tres behaviours (`BlobStore`, `KeyStore`, `Ops`); Ecto e o seam de persistencia; sem behaviour por contexto.
- **D4:** `repo` explicito como argumento; sem `Ypy.Repo`, sem `with_tekoa`, sem RLS no kernel.
- **D5:** Tabelas flat sem schema Postgres; migracoes portaveis copiadas por `mix ypy.install` e rodadas pelo produto.
- **D6:** Uma cadeia assinada por Tekoa unindo mural e commits de record; o append e **atomico** com a escrita (mesma transacao); serializacao no banco (unique `(tekoa_id, seq)` + retry que reexecuta a transacao inteira).
- **D7:** Lexicons ATProto-shaped em `priv/lexicons`; components OpenAPI no kernel; teste de conformidade lexicon-componente; paths no box.
- **D8:** Identidade `did:web` por Tekoa; moradores sao handles; sem PLC.
- **D9:** RFC_003 D5 aplicado por completo (Concessao/Proposta/Voto) e **pt-BR sempre no dominio**: os valores tambem em pt-BR (`:aberta/:aprovada/:rejeitada/:expirada`, `:remover_morador/:nomear_zelador/:destituir_zelador/:mudar_cota`, `:sim/:nao`); `rkey`/TID como id publico.
- **D10:** Fluxos de auth no produto; o kernel tem substantivos e invariantes; `Ypy.Maraca.Credentials` (bcrypt) entra no kernel.
- **D11:** Trava de uma-Tekoa-por-box e politica do box, nao invariante do kernel (o Conecta precisa de N).
- **D12:** Licenca MPL-2.0 (produto proprietario pode depender do kernel).
- **D13:** Docs (README, `@moduledoc`, `@doc`) em pt-BR, a lingua padrao do Taina; ingles opcional. Codigo em ingles, com o vocabulario social em pt-BR.
- **D14:** Expiracao de convite 2 dias; link de reset 30 minutos. Constantes de produto, no kernel.

---

## 14. Sequencia de build

Cada passo tem um portao; o seguinte nao comeca sem ele.

1. Esqueleto: projeto lib, deps, CI. Portao: compila limpo.
2. Primitivas vindas do Pesque (cbor, cid, varint, base32, tid, secp256k1; atribuicao). Portao: property tests verdes com nenhum repo e nenhum arquivo tocado.
3. Identidade: handle, `did:web`, chaves, credenciais. Portao: golden tests.
4. Lexicon + Record: validar, encode/decode, determinismo. Portao: envelope montado duas vezes de entrada embaralhada da bytes e CID identicos.
5. Cadeia (mural): seal, verify_entry, verify_chain. Portao: cadeia de N verifica; virar um byte quebra naquela entrada.
6. Politica social: zonas, papeis, regra de leitura, expiracao. Portao: matriz de autorizacao, incluindo zelador-sem-atalho.
7. Politica da assembleia: quorum, maioria, expiracao. Portao: tabela-verdade, empate, quorum faltando.
8. Schemas, changesets, queries. Portao: changeset puro; query compila.
9. TestRepo + stores + contextos (SQLite `:memory`). Portao: upload ponta a ponta, conflito de cadeia sob corrida, ciclo da assembleia com claim unico.
10. Wire + conformidade. Portao: teste lexicon-componente verde; todo atomo de erro tem code.
11. `mix ypy.install` + docs. Portao: instala num app Phoenix de teste, migra, sobe.
12. Adocao no box (repo `taina`, separado): troca de SQLite, exportacao, migracoes de produto.

---

## 15. Nao-objetivos e riscos

**Nao-objetivos, com gatilho de retorno:** transporte de federacao e qualquer codigo de rede (gatilho: RFC de federacao pos-Guara); GC/reference counting de blob (gatilho: primeiro problema de espaco por dedup); impls S3 de BlobStore e KeyStore hospedado (gatilho: build do Conecta); DID por morador e assinatura no cliente (gatilho: cripto E2E, RFC_003 secao 7); formato de exportacao MST/CAR (gatilho: a federacao escolher); Oban, thumbnails, tabelas Jaci, device tokens, UI (produto).

**Riscos:**
1. A extracao do kernel cai no mesmo repo que tambem vai fazer a troca de persistencia do box (SQLite). Sao dois verticais grandes; sequenciar para nao colidirem.
2. Contrato de saida dos verbos: todo verbo admin tem que existir como verbo que um self-hoster roda igual, senao apodrece numa API admin que so existe pro control plane.
3. O tunel do Conecta (acesso remoto atras de NAT) e o primeiro hook que nao tem uso de self-hosting; resolver como o `ask` do TLS: env generico, qualquer relay, o nosso e uma opcao.

---

## 16. Relacao com as outras RFCs (emendas)

- **RFC_002:** D10 (Taina Hospedada) deixa de ser multi-tenant compartilhado e passa a ser frota de caixas por Tekoa iguais ao box, atras de control plane que consome artefatos; Postgres sai do produto.
- **RFC_003:** D1 e D2 sobem para o kernel (`Ypy.Maraca.Policy` e a funcao de leitura); o RLS fica estacionado; a convencao de nomes da secao 5 e aplicada por completo (D9 desta RFC).
- **RFC_005:** secao 12 deixa de valer no ponto "sem artefato OpenAPI no codigo": o kernel entrega components OpenAPI e os drift-checa; os paths continuam no box, e o id publico de arquivo passa a ser o `rkey` (TID).
- **RFC_004:** segue adiada e independente; consome o kernel quando sair do papel.

---

## 17. Questoes em aberto

- **Janela de rotacao de chave:** proposta de listar a chave atual e a anterior no documento `did:web` com aposentadoria em 30 dias, rotacao registrada como entrada da cadeia. Numero e promessa de produto.
- **Assinatura no envelope vs na cadeia:** esta RFC põe a assinatura na entrada da cadeia. Se um dia um record precisar ser verificavel sem a cadeia, reavaliar.
- **Escopo do registry de lexicons em multi-produto:** hoje o produto carrega os diretorios no boot; se um dia houver lexicons dinamicos, reavaliar.
