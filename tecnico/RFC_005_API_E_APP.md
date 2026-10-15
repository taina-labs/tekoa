# RFC 005: API JSON v1 e o app nativo

**Status:** Proposta
**Data:** 2026-10-12
**Relação:** Deriva do `RFC_002_MVP.md` (a API é o item 1 do pós-MVP, o app nativo é o item 2) e herda as regras sociais do `RFC_003_GOVERNANCA_E_TRANSPARENCIA.md` (zonas casa/praça, zeladora sem atalho). Não altera nenhum contexto; expõe o que já existe.
**Emenda:** `RFC_006_KERNEL.md` move os *components* OpenAPI para o kernel (Ypy) e muda o id público de arquivo/pasta para o `rkey` (TID).
**Autoria:** @zoedsoupe + revisão técnica

---

## 1. Resumo executivo

O Tainá entrega hoje a interface web (LiveView) do cofre da comunidade. Esta RFC define a **API JSON v1** e o **app nativo** que ela serve, para que uma pessoa instale o Tainá pela loja de aplicativos, informe o endereço da sua comunidade e use tudo nativamente, sem instalar PWA e sem configurar rede.

Três decisões de base:

1. A API é um **canal fino sobre os contexts que já existem** (Maracá, Ybira, Jaci). Nenhuma regra de negócio nova. Todo endpoint é um controller que recebe, chama uma função de contexto e responde.
2. A autenticação do app é por **token de dispositivo** (Bearer), não por sessão de cookie. Por dispositivo, com nome, revogável pela própria pessoa ou pela zeladora.
3. O app v0.1.0 é **o Tainá inteiro**, não só fotos: sessão, pessoas, arquivos, fotos, mural, pedidos de acesso, armazenamento. É um cliente do produto, não um utilitário de backup.

Fora desta RFC (v1): tempo real, upload resumível, refresh token, Guará, federação.

---

## 2. Nomes

- **Repositório:** `taina-labs/okena` (monorepo). Okena (Tupi-Guarani: porta, entrada; ok = casa) é o código, não o produto.
- **App nas lojas:** **Tainá**. É o nome que a pessoa vê e busca; "Okena" nunca aparece na interface, nos textos das lojas nem nas mensagens da API.
- **Bundle iOS:** `io.taina.app`; **application id Android:** `io.taina.app`.
- O repositório contém `ios/` (Swift, agora), `android/` (Kotlin, depois), `api/openapi.yaml` (contrato compartilhado) e `design/` (tokens espelhados do Penpot).

Se "Okena" vazar para texto que a pessoa lê, o objetivo de não confundir se perde. A costura é: Okena é código, Tainá é o produto.

---

## 3. Decisões

### D1: A API é canal fino, sem regra de negócio nova

Controllers em `TainaWeb.API.*` parseiam, chamam uma função de contexto e renderizam. A autorização, as zonas casa/praça e o RLS ficam onde já estão. A API nunca reimplementa regra e nunca lê a sessão de cookie.

### D2: Token de dispositivo, não sessão

O irmão mobile da sessão Phoenix. Segredo com prefixo `tnd_` mais 32 caracteres; no banco só o hash SHA-256 (mesmo formato de `Ava.hash_token/1`). O segredo cru aparece uma única vez, na criação. Revogável pela dona ou pela zeladora. Sem expiração e sem teto de dispositivos em v1; a revogação é o mecanismo de ciclo de vida.

SHA-256, não bcrypt: bcrypt existe para senha humana de baixa entropia. Um segredo de dispositivo tem 32 caracteres aleatórios, e bcrypt custaria perto de 100ms de CPU por requisição na caixa da comunidade.

### D3: Bearer, sem cache de resolução

Cada requisição resolve o token no banco, dentro de uma transação de sistema, igual à sessão web (`resolve_session_ava`). Revogação e conta desativada passam a valer na requisição seguinte, na hora. Custo: três SELECTs indexados por requisição, irrelevante numa caixa que serve uma comunidade. Se um dia pesar, o cache é ETS com TTL de 30s, invalidado na revogação, e aí a revogação deixa de ser instantânea: decisão de produto, não padrão de engenharia.

### D4: Multipart primeiro

Um arquivo por requisição, reusando `Ybira.upload/3` (recebe um caminho temporário; o `Plug.Upload` já fornece um). Upload resumível fica para quando links instáveis derrubarem vídeos grandes.

### D5: Sem tempo real em v1

Sem Channels, sem SSE. O cliente faz polling. Channels sobre o mesmo token de dispositivo é o caminho de v2, quando o mural e os pedidos de acesso precisarem empurrar.

### D6: Versão e formato

`/api/v1`. Erros num envelope único `{"error": {"code", "message", "details?"}}`. `code` é o átomo do contexto em string, contrato estável do cliente; `message` é pt-BR, no tom dos erros da web (com saída, sem culpa).

---

## 4. Modelo do token de dispositivo

Tabela `maraca.device_tokens`, no schema `maraca`, com RLS como todas as tabelas de Tekoa.

Colunas: `name` (rótulo, 1 a 80), `public_id`, `token_hash`, `last_used_at`, `revoked_at`, `ava_id`, `tekoa_id`, timestamps. Índices únicos em `public_id` e `token_hash`, índices em `ava_id` e `tekoa_id`.

Callbacks novos em `Taina.Maraca.Behaviour`:

- `create_device_token(scope, name)` retorna `{:ok, %DeviceToken{}} | {:error, %Ecto.Changeset{}}`
- `resolve_device_token(token)` retorna `{:ok, %{scope: Scope, device: DeviceToken}} | {:error, :not_authenticated}`
- `list_devices(scope)` retorna `{:ok, [DeviceToken]}`
- `revoke_device(scope, device_public_id)` retorna `{:ok, DeviceToken} | {:error, :not_found | :unauthorized}`

Detalhes:

- `last_used_at` só é escrito quando passar de 5 minutos, sem escrita no caminho quente.
- `resolve_device_token/1` revalida a conta a cada chamada (conta desativada não resolve, mesmo com o token intacto), igual à sessão web. Conta reativada volta a resolver, com os tokens antigos.
- Revogar: dona do token ou zeladora. Idempotente.
- Sem evento no mural: entrar no próprio celular é higiene de conta, como trocar a senha.

---

## 5. Autenticação, scope e RLS

`TainaWeb.API.Auth` (módulo plano, plugs de função, espelha `TainaWeb.Auth` sem tocá-lo):

- `fetch_api_scope/2`: lê `Authorization: Bearer <token>`, resolve em `Taina.Scope`, assina `:current_scope` e `:api_device`. Não barra ninguém.
- `require_api_auth/2`: sem scope, responde 401 no envelope.
- `require_api_zelador/2`: sem zelador, responde 403 (a API é contrato documentado, diverge do 404 do painel web de propósito: o cliente precisa distinguir proibido de inexistente).

A API nunca lê a sessão de cookie, então não há CSRF a modelar. Cada chamada de contexto abre o próprio `Repo.with_tekoa` a partir do scope. Sem transação de requisição inteira, sem mecanismo de RLS novo. O rate limit de login (`hammer`) é reusado em `POST /session`; adivinhar Bearer não tem rate limit, porque o espaço do segredo é grande demais para isso valer algo.

---

## 6. Erros

Todos os átomos que os contexts podem devolver, mapeados para status e código. Os não óbvios:

- `:storage_quota_exceeded` é 409, não 413 nem 507: a comunidade encheu, não é corpo grande nem falha de servidor.
- `:account_deactivated` é 403, não 401: 401 convida o cliente a repedir credenciais em laço. O `code` carrega o significado, e o cliente não deve tentar de novo.
- Conflito de estado (409) e entrada malformada (422) é a linha divisória.
- `%Ecto.Changeset{}` vira 422 `validation_failed` com `details` (mensagens pt-BR).
- Átomo não mapeado vira 500 `internal_error`; o motivo vai para o log, nunca para o corpo.

| Motivo | HTTP | code |
| --- | --- | --- |
| `:not_authenticated`, `:invalid_credentials`, `:invalid_token` | 401 | o próprio átomo |
| `:account_deactivated`, `:forbidden`, `:unauthorized`, `:not_zelador`, `:not_owner`, `:not_authorized` | 403 | o próprio átomo |
| `:not_found`, `:no_folder` | 404 | o próprio átomo |
| `:last_zelador`, `:not_pending`, `:invalid_status`, `:already_has_access`, `:already_readable`, `:already_bootstrapped`, `:not_bootstrapped`, `:circular_reference`, `:storage_quota_exceeded` | 409 | o próprio átomo |
| `:cannot_request_own`, `:cross_tekoa_owner`, `:invalid_action`, `:mime_not_allowed`, `:missing_file` | 422 | o próprio átomo |
| `%Ecto.Changeset{}` | 422 | `validation_failed` |
| `:rate_limited` | 429 | `rate_limited` |
| qualquer outro | 500 | `internal_error` |

---

## 7. Paginação

Envelope uniforme, um só decoder no cliente:

- Listas por keyset (fotos, timeline, lixeira, conteúdo por cursor): `{"items": [...], "next_cursor": "<opaco>" | null}`. O cursor é o `next_cursor` do Ybira/Jaci, repassado como `after_cursor`.
- Listas por offset (conteúdo de pasta, mural): mesmo envelope; `next_cursor` carrega um offset decimal em string.
- Conteúdo de pasta: `{"folders": [...], "items": [...], "next_cursor": ...}` (as pastas vêm completas, só os arquivos paginam).
- `next_cursor` é sempre string ou null.

`limit` (padrão 50, teto 100), `cursor` e `sort` (`name|date|size` mais `asc|desc`) são parseados lenientes na borda.

---

## 8. Upload, download e miniatura

Upload: `POST /api/v1/files`, multipart, um arquivo. Reusa `Ybira.upload/3`, que já faz cota, MIME por magic bytes e limpeza em erro. O `Plug.Upload` apaga o temporário quando o processo da requisição morre. O `sha256` volta no JSON do arquivo para o cliente de backup pular o que já subiu.

> Achado, não introduzido: o upload da web resolve o destino com `Ybira.get_folder/2` (regra de leitura) e o `upload/3` aceita `folder_id` cru, então hoje um morador pode subir para uma pasta da praça de outra pessoa. A API mantém paridade. Se não for intencional, o conserto é no Ybira (resolver a pasta possuída na borda do upload), não na API.

Download e miniatura: extrair a lógica de bytes do `TainaWeb.FileController` para `TainaWeb.FileServing` (módulo plano) e as duas frentes chamam. A web mantém auth por sessão e redirect; a API mantém Bearer e erro JSON. A regra das duas zonas vale porque as duas frentes autorizam por `Ybira.get_file/2` antes de servir byte; o `FileServing` nunca abre arquivo sozinho. Os testes de download e miniatura da web continuam passando sem edição.

O parser multipart ganha teto de 2 GiB (paridade com o upload da web via LiveView); urlencoded e json ficam no padrão de 8 MB.

---

## 9. Endpoints (superfície completa do v1)

Papéis: `pub` (sem auth), `mor` (qualquer morador), `zel` (zeladora). A fatia 1 é a vertical mínima.

### Instância e sessão

| Método | Caminho | Papel | Contexto | Fatia |
| --- | --- | --- | --- | --- |
| GET | /api/v1/instance | pub | `bootstrapped?/0`, `get_tekoa/0` | 1 |
| POST | /api/v1/session | pub | `authenticate/3` mais `create_device_token/2` | 1 |
| DELETE | /api/v1/session | mor | `revoke_device/2` | 1 |
| GET | /api/v1/me | mor | assigns | 1 |
| GET | /api/v1/devices | mor | `list_devices/1` | 2 |
| DELETE | /api/v1/devices/:id | mor (dono) / zel | `revoke_device/2` | 2 |

### Pessoas

| Método | Caminho | Papel | Contexto | Fatia |
| --- | --- | --- | --- | --- |
| GET | /api/v1/members | mor | `list_members/1` | 2 |
| POST | /api/v1/members | zel | `invite_user/3` | 3 |
| PATCH | /api/v1/members/:id | zel | `update_member_role/3` | 3 |
| DELETE | /api/v1/members/:id | zel | `deactivate_member/2` | 3 |
| POST | /api/v1/members/:id/reactivate | zel | `reactivate_member/2` | 3 |
| DELETE | /api/v1/members/:id/invite | zel | `revoke_invite/2` | 3 |
| POST | /api/v1/members/:id/reset-link | zel | `mint_reset_link/2` | 3 |

### Convite e senha (públicos)

O aceite do convite e a redefinição de senha chegam por deep link
(`taina://convite/:token`, `taina://redefinir/:token`). O link carrega também o
endereço da caixa (`?host=`), porque quem chega por um convite ainda não tem
comunidade salva no aparelho. Os quatro são públicos: o token é a credencial.
Os dois POST devolvem uma **sessão** (token de dispositivo), então a pessoa já
entra ao terminar; a confirmação de senha é conferida no cliente, que envia um
único `password`.

| Método | Caminho | Papel | Contexto | Fatia |
| --- | --- | --- | --- | --- |
| GET | /api/v1/invites/:token | pub | `invite_status/1` | 3 |
| POST | /api/v1/invites/:token/accept | pub | `accept_invite/2` mais `create_device_token/2` | 3 |
| GET | /api/v1/password-resets/:token | pub | `reset_status/1` | 3 |
| POST | /api/v1/password-resets/:token | pub | `reset_password/3` mais `create_device_token/2` | 3 |

### Pastas e arquivos

| Método | Caminho | Papel | Contexto | Fatia |
| --- | --- | --- | --- | --- |
| GET | /api/v1/folders | mor | `list_folders/1` | 1 |
| POST | /api/v1/folders | mor | `create_folder/2` | 1 |
| GET | /api/v1/contents | mor | `list_folder_contents/3` | 1 |
| GET | /api/v1/folders/:id | mor | `get_folder/2` | 2 |
| PATCH | /api/v1/folders/:id | mor (dono) | `rename_folder` / `publicar_folder` / `tirar_folder_da_praca` | 2 |
| POST | /api/v1/folders/:id/move | mor (dono) | `move_folder/3` | 2 |
| DELETE | /api/v1/folders/:id | mor (dono) | `delete_folder/2` | 2 |
| POST | /api/v1/files | mor | `get_folder/2` mais `upload/3` | 1 |
| GET | /api/v1/files/:id/download | mor (leitura) | `get_file/2` mais `FileServing.serve/2` | 1 |
| GET | /api/v1/files/:id/thumbnail/:size | mor (leitura) | `get_file/2` mais `FileServing.thumbnail/3` | 1 |
| GET | /api/v1/files/:id | mor (leitura) | `get_file/2` | 2 |
| PATCH | /api/v1/files/:id | mor (dono) | `rename_file` / `publicar_file` / `tirar_file_da_praca` | 2 |
| POST | /api/v1/files/:id/move | mor (dono) | `move_file/3` | 2 |
| DELETE | /api/v1/files/:id | mor (dono) | `delete_file/2` | 2 |
| GET | /api/v1/recent | mor | `list_recent/2` | 3 |

### Lixeira e fotos

| Método | Caminho | Papel | Contexto | Fatia |
| --- | --- | --- | --- | --- |
| GET | /api/v1/trash | mor (própria) | `list_trash/2` | 2 |
| POST | /api/v1/trash/:file_id/restore | mor (dono) | `restore_file/2` | 2 |
| GET | /api/v1/photos | mor | `list_photos/2` | 2 |
| GET | /api/v1/photos/timeline | mor | `timeline/2` | 3 |

### Pedidos de acesso, mural, armazenamento, saúde

| Método | Caminho | Papel | Contexto | Fatia |
| --- | --- | --- | --- | --- |
| GET | /api/v1/files/:id/access-target | mor | `fetch_access_target/2` | 3 |
| POST | /api/v1/files/:id/access-request | mor | `fetch_access_target/2` mais `request_access/5` | 3 |
| GET | /api/v1/access-requests | mor | `list_access_requests/1` | 3 |
| GET | /api/v1/my-requests | mor | `list_my_requests/1` | 3 |
| POST | /api/v1/access-requests/:id/approve | mor (dono) | `approve_access_request/2` | 3 |
| POST | /api/v1/access-requests/:id/deny | mor (dono) | `deny_access_request/2` | 3 |
| GET | /api/v1/mural | mor | `list_events/2` | 3 |
| GET | /api/v1/storage | mor | `storage_stats` mais `storage_stats_by_kind` mais contagens | 2 |
| PATCH | /api/v1/storage/quota | zel | `update_tekoa_quota/2` | 3 |
| GET | /api/v1/saude | mor | stats mais `pending_job_counts` mais `last_backup` | 3 |
| POST | /api/v1/saude/backup | zel | `Oban.insert(Backup.new(%{}))` | 3 |

Nomes de caminho: inglês para o encanamento (`files`, `folders`, `devices`), pt-BR onde a palavra é o vocabulário do produto (`/mural`, `/saude`, zona `casa`/`praca`, papéis `zelador`/`morador`).

> **Emenda (RFC_006):** o `:id` público de arquivo e pasta passa a ser o `rkey`
> (TID), substituindo o `public_id` nanoid. Os caminhos não mudam; o segmento
> muda de valor. Ciente de que o TID é ordenável por tempo.

---

## 10. Fatia mínima (v0.1.0 do app)

Endpoints: instance, session (login por token), me, folders (listar e criar), contents, files (upload, download, miniatura). Login, navegar, subir, ver: ponta a ponta num aparelho.

Plano de arquivos no `taina`:

```
priv/repo/migrations/*_create_maraca_device_tokens.exs             novo
lib/taina/maraca/device_token.ex                                   novo
lib/taina/maraca/behaviour.ex                                      edita (+4 callbacks)
lib/taina/maraca.ex                                                edita (+4 @impl)
lib/taina_web/api/auth.ex                                          novo
lib/taina_web/api/error.ex                                         novo
lib/taina_web/api/query.ex                                         novo
lib/taina_web/api/serializer.ex                                    novo
lib/taina_web/api/{instance,session,me,folder,file}_controller.ex  novos
lib/taina_web/file_serving.ex                                      novo (extraído)
lib/taina_web/controllers/file_controller.ex                       edita (delega)
lib/taina_web/router.ex                                            edita (:api, scope /api/v1)
lib/taina_web/endpoint.ex                                          edita (multipart 2 GiB)
lib/taina_web/controllers/error_json.ex                            edita (envelope)
test/support/conn_case.ex                                          edita (authorize_device/2)
test/taina/rls_isolation_test.exs                                  edita (device_tokens)
test/taina/maraca/device_token_test.exs                            novo
test/taina_web/api/*_test.exs                                      novos
```

No `okena`:

```
api/openapi.yaml
ios/Okena/App/OkenaApp.swift
ios/Okena/Core/{TainaAPI,LiveTainaAPI,FakeTainaAPI}.swift
ios/Okena/Core/Models/
ios/Okena/Core/{Keychain,SessionStore,HostStore}.swift
ios/Okena/Features/{Setup,Login,Files,Photos,People,Mural,Account}/
android/   (depois)
design/
```

Primeiro build do app: Setup (informar o endereço, `GET /instance`), Login (`POST /session`), Arquivos (listar, subir, baixar, miniatura). O resto entra no mesmo protocolo `TainaAPI`.

---

## 11. Testes

Sem Mox e sem Bypass no v1: o padrão do repositório é testar os contexts reais sob o sandbox SQL, e não há HTTP externo nesta superfície (nada para o Bypass apontar). O que a suíte prova:

- Ciclo do token: a criação devolve o cru uma vez e o banco só tem o hash; resolve válido, inválido, revogado e desativado; `last_used_at` com a janela de 5 minutos; revogar por dona, por zeladora, idempotente, `not_found` de outra Tekoa.
- Auth: sem header 401; header lixo 401; cookie não autentica a API (sessão de teste sem Bearer responde 401); revogar entre duas requisições (200 e depois 401); conta desativada 401 na seguinte; rate limit de login 429.
- RLS com papel não-superuser: incluir `maraca.device_tokens` no teste de isolamento (sem contexto, 0 linhas; contexto certo, 1 linha; válvula visível só na transação).
- Erros: uma asserção por átomo da tabela; changeset 422 com `details`; átomo desconhecido 500 sem o motivo no corpo.
- Zonas casa/praça sobre a API: dono sobe um arquivo de casa; outro morador recebe 403 no download, na miniatura e no metadado, e o arquivo não aparece no `contents` nem nas `photos`; publicar na praça libera; a zeladora não tem atalho (403 no arquivo de casa de um morador); 404 e 403 são distintos.
- Upload: caminho feliz 201 com sha256; parte ausente 422 `missing_file`; pasta de casa alheia 403; cota estourada 409 e `storage_stats` inalterado; magic bytes 422 `mime_not_allowed`; Range 206 e 416.

`mix precommit` é o portão de saída.

---

## 12. Não-objetivos e riscos

Não-objetivos do v1, cada um com o gatilho de retorno:

- Sem tempo real (Channels/SSE quando o mural precisar empurrar).
- Sem upload resumível (quando links instáveis derrubarem vídeos grandes).
- Sem refresh token, expiração ou teto de dispositivos (incidente de aparelho perdido num piloto).
- Sem cache de token (só se o profiling mostrar a resolução dominando).
- Sem aceite de convite, reset de senha ou revogação de permissão pela API (deep link `taina://convite/:token`).

  *Emenda (app nativo):* o **aceite de convite** e a **redefinição de senha**
  entraram no v1 (`POST /api/v1/invites/:token/accept` e
  `POST /api/v1/password-resets/:token`, ambos públicos e devolvendo sessão),
  porque o app precisa postar o token sem sair para o navegador. O deep link
  passa a carregar o endereço da caixa (`?host=`). A revogação de permissão
  pela API continua fora.
- Sem artefato OpenAPI no código; a tabela de endpoints vive aqui, no `tekoa` (segundo cliente Kotlin ou consumidor público).

  *Emenda (RFC_006): invertido em parte. O kernel (Ypy) entrega os **components** OpenAPI e os drift-checa por teste de conformidade; a tabela de endpoints (paths) continua aqui, no `tekoa`.*

Riscos:

1. Latência de revogação contra custo por requisição: sem cache, toda requisição paga uma transação de sistema. Paridade com a sessão web, irrelevante numa caixa. O cache compra latência e perde o "revoga neste segundo".
2. Destino de upload (achado): qualquer morador pode subir numa pasta da praça alheia. Conserto no Ybira.
3. Cursor de offset desvia sob inserção concorrente (conteúdo de pasta, mural): inerente ao offset, igual à web; as listas por keyset não desviam.
4. Teto multipart de 2 GiB ocupa temporário em disco (N uploads concorrentes ocupam N vezes 2 GB); mitigação operacional (limites no Caddy, tmpfs no `install.sh`), anotar no runbook.
5. Tokens `tnd_` em log: o prefixo torna a captura acidental identificável; a revogação é o remédio e é instantânea.
6. Bearer-only: cookie da web não dá acesso à API, de propósito.
7. `account_deactivated` como 403: o cliente não deve repedir login; documentar no contrato do cliente.

---

## 13. Relação com as outras RFCs

- **RFC 002 (MVP):** esta RFC implementa o item 1 do pós-MVP (API JSON v1) e o pré-requisito do item 2 (app nativo). Não muda nenhuma decisão do MVP.
- **RFC 003 (governança e transparência):** as regras sociais são herdadas sem exceção. A API respeita casa/praça em todo caminho de leitura, a zeladora não tem atalho para a casa de ninguém, e a lixeira é só da dona. O mural e a saúde entram como leitura.
- **RFC 004 (universidade):** segue adiada e independente.
