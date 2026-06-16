# RFC 002, Tainá MVP: Arquitetura Revisada e Plano de Implementação

**Status:** Proposta
**Data:** 2026-06-09
**Substitui:** `ROADMAP.md` (integralmente) e as seções de entrega/cliente da `RFC_ARQUITETURA.md` (RFC 001)
**Autoria:** @zoedsoupe + revisão técnica

---

## 1. Resumo Executivo

O Tainá é um servidor de nuvem comunitária auto-hospedado, plug & play, para
comunidades, inclusive não-técnicas. Ele não tenta substituir o Nextcloud hoje;
ele compete em **experiência de instalação e operação** e, no longo prazo, em
escopo de funcionalidades.

Esta RFC corrige três erros da RFC 001 e do roadmap anterior:

1. **"PWA offline-first + LiveView" é uma contradição.** LiveView é
   server-driven e exige conexão; PWA offline-first exige estado no cliente e
   sincronização. O MVP usa LiveView onde faz sentido (UI web do servidor,
   admin, setup) e reserva apps cliente (PWA/nativo) para o pós-MVP, atrás de
   uma API JSON.
2. **O roadmap anterior era irrealista.** Previa Fase 1 e 2 concluídas em
   abril/2025; em junho/2026 o código ainda está pré-Fase 1. Esta RFC
   re-baseia o plano na velocidade real (projeto solo, tempo parcial) com
   fases verticais e critérios de saída explícitos.
3. **O produto do MVP é a instalação, não a feature.** O diferencial
   competitivo contra Nextcloud não é ter mais recursos, é uma comunidade
   não-técnica colocar um servidor no ar em menos de 30 minutos e nunca ter
   medo de atualizar.

**MVP = cofre de arquivos e fotos da comunidade, com instalação indolor.**
Chat (Guará) sai do MVP. Federação, XRPC, IA, apps nativos: pós-MVP.

---

## 2. O que o MVP é (e o que não é)

### É

- Um servidor único (Raspberry Pi 5 / mini PC N100) por comunidade.
- Maraca: contas, convites por link/QR code, papéis (admin/membro), sessões.
- Ybira: upload, navegação, download, lixeira, cotas de armazenamento.
- Jaci-lite: visão de fotos (grade + linha do tempo) sobre os arquivos do
  Ybira, com thumbnails.
- Nhaman-lite: assistente de primeiro boot, gestão de usuários, uso de disco,
  backup com um clique, botão de atualização.
- Interface 100% web server-rendered (Phoenix LiveView), mobile-first.
- Instalação: `install.sh` + Docker Compose (fase 1 de deploy); imagem de SD
  card estilo Home Assistant OS (fase 2 de deploy).

### Não é (decisões explícitas de corte)

| Cortado do MVP                                         | Por quê                                                                                                                                                                         | Quando volta                                |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| **Guará (chat)**                                       | Comunidades já têm Signal/WhatsApp; custo de troca altíssimo, valor marginal baixo. Arquivos/fotos é onde a soberania de dados importa de verdade.                              | Pós-MVP, após API v1                        |
| **PWA offline-first**                                  | iOS não tem Background Sync nem acesso ao rolo de câmera via PWA; armazenamento evictável. "Backup de fotos via PWA" não existe tecnicamente, o app do Immich é Flutter nativo. | Pós-MVP como app nativo                     |
| **API pública (REST/XRPC)**                            | Sem cliente para consumi-la, é custo morto. Os contexts já nascem com contrato de API (`Maraca.Behaviour`), então a camada HTTP futura é fina.                                  | Pós-MVP, junto com o app de backup de fotos |
| **XRPC / AT Protocol**                                 | Nunca esteve na RFC 001 nem no código; só no README. Remover a menção.                                                                                                          | Indefinido (provavelmente nunca)            |
| **ClamAV / antivírus**                                 | Daemon pesado num Pi, atualizações de assinatura, e o modelo de ameaça é uma comunidade privada, não um serviço de upload público.                                              | Opt-in pós-MVP                              |
| **E-mail obrigatório**                                 | SMTP auto-hospedado = inferno de deliverability para admin não-técnico. Convites por link/QR code. SMTP da comunidade é opcional.                                               | Descartado (RFC_003)                        |
| **Federação / NATS / event bus**                       | Zero demanda comprovada; complexidade enorme.                                                                                                                                   | Quando 10+ comunidades pedirem              |
| **Multi-nó / clustering**                              | Um nó BEAM atende 50 usuários com folga. `dns_cluster` fica no código, inerte.                                                                                                  | Hospedagem gerenciada                       |
| **Streaming (Araci), IoT (Ka'a), monitoração (Karai)** | Stubs de README hoje. Manter como visão, não como promessa.                                                                                                                     | Backlog                                     |

---

## 3. Decisões de Arquitetura (ADRs)

### D1, Entrega via LiveView no MVP; API + clientes depois

A UI do MVP é Phoenix LiveView. Justificativa: o usuário está, por definição,
online com o servidor quando o usa (LAN ou VPN); LiveView elimina a camada de
API, serialização e estado de cliente; um único codebase para um dev solo.

LiveView **onde faz sentido**: navegação de arquivos, galeria, admin, setup.
Quando o app nativo de backup de fotos chegar (pós-MVP), ele consome uma API
JSON nova, e os contexts já têm contratos prontos (`Maraca.Behaviour` etc.),
então a camada HTTP é controller fino sobre função existente.

### D2, Uma Tekoa por instância; RLS mantido

O modelo de produto é "cada comunidade roda sua caixa". Logo, o MVP opera em
**modo single-tekoa, com enforcement rígido**: o setup cria exatamente uma
Tekoa e o sistema **recusa** a criação de uma segunda, garantido em dois
níveis: na função de criação do context Maraca e por índice único no banco
(`CREATE UNIQUE INDEX ... ON maraca.tekoas ((true))`). Invariantes mais
simples em todo o resto do código. O índice é removível no futuro modo de
hospedagem gerenciada multi-tenant.

O RLS (Row-Level Security) do PostgreSQL **permanece e é ligado de verdade**
(hoje as policies existem mas nenhuma query seta `app.current_tekoa_id`, ver
seção 5, Fase 0). Razões para manter mesmo com tenant único:

1. Defesa em profundidade: bug de aplicação não vaza dados entre tekoas.
2. É a fundação da futura **hospedagem gerenciada paga** (modelo Penpot):
   múltiplas comunidades em infra compartilhada, isolamento garantido pelo
   banco.

### D3, Padrão Scope (Phoenix 1.8) + RLS automático no Repo

Todo context recebe um `Taina.Scope` como primeiro argumento e o Repo injeta o
contexto RLS de forma impossível de esquecer:

```elixir
defmodule Taina.Scope do
  @enforce_keys [:ava, :tekoa]
  defstruct [:ava, :tekoa]
end

# Contexts: sempre scope-first
def list_files(%Scope{} = scope, folder_id) do
  Repo.with_tekoa(scope.tekoa.public_id, fn ->
    Repo.all(from f in Ybira.File, where: f.folder_id == ^folder_id)
  end)
end
```

E, como rede de segurança, `prepare_query/3` no Repo **levanta exceção** se uma
query rodar fora de contexto de tekoa sem a flag explícita
`skip_tekoa_id: true` (migrações e operações de sistema). A RFC 001 já
especificava esse fail-safe; ele nunca foi implementado. Agora é obrigatório
e bloqueia a Fase 0.

### D4, Fotos ficam no MVP (Jaci-lite), como camada sobre o Ybira

Fotos são o gancho emocional do produto (o caso de uso que mais cresce em
self-hosting, vide Immich). Custo marginal baixo depois do Ybira pronto:
filtro por `mime_type image/*`, thumbnails, grade e linha do tempo. Sem
álbuns complexos, sem EXIF profundo, sem upload automático de celular (isso é
o app nativo do pós-MVP).

### D5, PostgreSQL mantido (contra SQLite)

SQLite + Litestream seria operacionalmente mais simples para uma caixa única
(modelo PocketBase). Mas: o RLS não existe no SQLite, o investimento em
policies já foi feito, e o caminho de hospedagem gerenciada exige Postgres.
Decisão: Postgres 16, tunado para Pi (shared_buffers baixo, ver compose).

### D6, Convites por link, e-mail opcional

`invite_user` gera URL com token (+ QR code renderizado na tela do admin).
Admin manda o link pelo canal que a comunidade já usa. Swoosh entra como
adapter opcional se a comunidade tiver SMTP próprio.

> Atualização (RFC_003, seção 4): o e-mail deixou de ser "opcional" e foi
> **descartado**. A identidade passou a ser o `username` (login por nome) e a
> recuperação é mediada pelo zelador via link de redefinição. Esta D6 vale só
> pela parte de convites por link/QR; para identidade, ver RFC_003.

### D7, Deploy é produto: Compose primeiro, imagem de appliance depois

- **Deploy v1 (MVP):** `install.sh` + Docker Compose + Caddy (TLS automático)
  - serviço de backup. Migrações rodam no boot do release
    (`Ecto.Migrator`), nunca como passo manual.
- **Deploy v2 (pós-MVP imediato):** imagem de SD card NixOS (via
  `nixos-generators`) para Pi 5, ligar, acessar `http://taina.local`,
  seguir o wizard. O `flake.nix` existente já aponta nessa direção.
- **Acesso remoto:** Tailscale documentado como caminho recomendado (resolve
  NAT, zero config). WireGuard puro fica como alternativa avançada.

### D8, Processamento de mídia: libvips, sem binários externos frágeis

Thumbnails e leitura básica de EXIF via `image`/`vix` (bindings libvips):
rápido, pouca memória, perfeito para ARM. **Sem** ImageMagick, **sem**
exiftool, **sem** heif-convert no MVP (HEIC: aceitar upload, thumbnail
best-effort; conversão completa fica para depois). Detecção de MIME por magic
bytes (biblioteca pura Elixir), nunca por extensão.

### D9, Jobs de fundo: Oban

Thumbnails, limpeza de lixeira (30 dias), backups agendados. Oban usa o
próprio Postgres (sem Redis), sobrevive a queda de energia do Pi, dá retry e
agendamento de graça.

### D10, Sustentabilidade: OSS sempre; receita estilo Nabu Casa, não só Penpot

Ordem de implementação de receita (todas mantêm o core AGPL):

1. **Agora:** Open Collective / GitHub Sponsors (doações).
2. **Pós-MVP, "Tainá Conecta" (modelo Nabu Casa / Home Assistant Cloud):**
   assinatura mensal para (a) relay de acesso remoto sem configurar rede e
   (b) backup criptografado off-site. Os dados continuam na caixa da
   comunidade, receita perfeitamente alinhada com a tese de soberania, e o
   modelo comprovadamente sustenta OSS desta categoria.
3. **Depois, "Tainá Hospedada" (modelo Penpot):** instância gerenciada
   multi-tenant para comunidades sem hardware. É aqui que o RLS multi-tekoa
   paga o investimento.
4. **Oportunista:** kits de hardware pré-gravados (Pi 5 + SD + case) vendidos
   no Brasil.

---

## 4. Stack consolidada

| Camada          | Escolha                                       | Observação                                                                                                                                              |
| --------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Runtime         | Elixir 1.20 / OTP 28, nó único                | `mix release`, migrações no boot                                                                                                                        |
| Web             | Phoenix 1.8 + LiveView                        | UI adiada: implementação é **backend-first**; deps de frontend (`phoenix_live_view`, `tailwind`, `esbuild`, etc.) só entram quando a fase de UI começar |
| Banco           | PostgreSQL 18 + RLS                           | schemas `maraca`/`ybira`/`guara`                                                                                                                        |
| Jobs            | Oban                                          | Postgres-backed                                                                                                                                         |
| Mídia           | `image` (libvips/vix)                         | thumbnails, EXIF básico                                                                                                                                 |
| Upload          | LiveView uploads (chunked, progresso)         | TUS/resumable fica para API pós-MVP                                                                                                                     |
| Download        | `Plug.Conn.send_file/5` com suporte a `Range` | streaming de vídeo básico via offset/length                                                                                                             |
| Auth            | Sessões em cookie criptografado, bcrypt       | rate limit de login (`hammer`), rotação de sessão                                                                                                       |
| E-mail          | Descartado (RFC_003)                          | identidade por username; convites por link/QR                                                                                                           |
| Proxy/TLS       | Caddy                                         | HTTPS automático, headers de segurança                                                                                                                  |
| i18n            | Gettext, pt-BR primeiro                       |                                                                                                                                                         |
| Observabilidade | LiveDashboard (admin-only) + `/health`        |                                                                                                                                                         |
| Deploy          | Docker Compose -> imagem NixOS (Pi)           | Tailscale para acesso remoto                                                                                                                            |

---

## 5. Plano de Implementação

Premissa de capacidade: dev solo, tempo parcial (~10-15h/semana). Cada fase é
uma **fatia vertical com critério de saída**; não se inicia a próxima sem
fechar o gate. Estimativas já incluem folga.

**Ordem de execução: backend-first.** Os contexts, a autorização, o storage e
a instalação vêm antes de qualquer UI. Os itens de interface (wizard, file
browser, galeria) das Fases 1-3 são implementados em bloco quando o backend
correspondente estiver estável, os gates de backend valem por contexto +
testes, não por tela.

### Fase 0, Fundação e correção (3 semanas)

O código atual tem bugs que bloqueiam qualquer coisa em cima:

- [ ] `Ybira.upload/2`: corpo do `with` vazio, implementar de verdade
      (gravar arquivo no layout de storage, inserir registro, retornar
      `{:ok, file}`).
- [ ] `Ybira.check_capacity/2`: corrigir campos inexistentes
      `used_storage`/`storage_quota` -> `storage_used_bytes`/`storage_quota_bytes`.
- [ ] `Ybira.get_file/1`: `Repo.get_by(File, ...)` resolve para o `File` da
      stdlib, usar `Ybira.File`.
- [ ] `Ava` changeset: typo `:filehash` -> `:file_hash`.
- [ ] `public_id` inconsistente: Ybira/Guara usam `:string` sem autogenerate,
      padronizar todos os schemas em `Taina.Repo.PublicId, autogenerate: true`.
- [ ] **Ligar o RLS de verdade:** `prepare_query/3` que levanta exceção sem
      contexto de tekoa; teste de isolamento cross-tekoa (o da RFC 001).
- [ ] Rota `/health` (o healthcheck do compose já aponta para ela e hoje
      falha).
- [ ] `Ecto.Migrator` no boot do release.
- [ ] Migração de enforcement single-tekoa (índice único, ver D2).
- [ ] Deps somente quando a fase exigir (backend-first; nenhuma dep de
      frontend agora).
- [ ] README do repo `taina`: **remover a menção a XRPC**; alinhar com esta
      RFC.

**Gate:** `mix precommit` verde; teste de isolamento RLS passando; container
sobe saudável com migrações automáticas.

### Fase 1, Maraca vertical: identidade e setup (6 semanas)

- [ ] Implementar `Maraca` cumprindo `Maraca.Behaviour` (hoje é só contrato):
      `invite_user`, `confirm_email` (via link), `authenticate`, `authorize?`,
      `grant/revoke_permission`, `request/approve/deny_access`.
- [ ] `Taina.Scope` + plug/on_mount que monta o scope da sessão.
- [ ] Setup wizard (primeiro boot): criar Tekoa única + admin, escolher local
      de storage, definir nome da comunidade.
- [ ] Login/logout, rotação de sessão, rate limit de tentativas.
- [ ] Convite por link + QR code; aceite de convite cria conta.
- [ ] Admin: listar membros, papéis, desativar conta.
- [ ] Testes: fluxo completo de convite->aceite->login; authorize matrix.

**Gate:** instância nova, do `docker compose up` ao segundo membro logado,
sem tocar em terminal depois do install. Cobertura de testes nos fluxos de
auth.

### Fase 2, Ybira vertical: arquivos (8 semanas)

- [ ] Layout de storage: `/var/taina/storage/{tekoa}/files/{ano}/{mes}/{public_id}{ext}`,
      SHA-256 registrado (dedup futura).
- [ ] Upload via LiveView uploads (multi-arquivo, progresso, drag & drop),
      validação por magic bytes, enforcement de cota.
- [ ] Navegador de arquivos: pastas hierárquicas, breadcrumb, grid/lista,
      renomear/mover/excluir.
- [ ] Lixeira: soft delete, restauração, purga automática em 30 dias (Oban).
- [ ] Download com `Range` (vídeo/áudio tocam no browser); preview de imagem
      e PDF.
- [ ] Quotas visíveis (admin define por tekoa; barra de uso).
- [ ] **Benchmark em hardware real (gate):** no Pi 5, upload de foto de
      4MB + thumbnail < 3s; streaming de vídeo 1080p sem stutter; 10k
      arquivos listados com paginação fluida.

**Gate:** benchmark acima + suíte de testes do context + teste de cota e de
isolamento RLS sobre arquivos.

### Fase 3, Jaci-lite: fotos (5 semanas)

- [ ] Worker Oban de thumbnail (libvips, 2 tamanhos) acionado pós-upload.
- [ ] Grade de fotos responsiva (2-6 colunas) com scroll infinito.
- [ ] Linha do tempo agrupada por data (EXIF quando houver, senão mtime).
- [ ] Visualização fullscreen com swipe.
- [ ] Upload de fotos pelo browser mobile (input capture), manual, não
      background.

**Gate:** galeria com 5k fotos navegável com fluidez num Pi 5.

### Fase 4, Operabilidade e pilotos (6 semanas)

- [ ] Backup com um clique + agendado (Oban): `pg_dump` + tar do storage para
      destino configurável (disco USB ou remote rclone). **Com verificação:**
      restore testado automaticamente em CI.
- [ ] Runbook de restauração documentado e ensaiado ("drill" com pessoa
      não-técnica).
- [ ] `install.sh` real (gera secrets, sobe compose, imprime URL do wizard).
- [ ] Botão/fluxo de atualização (pull de imagem + restart + migração).
- [ ] Documentação de Tailscale para acesso remoto.
- [ ] CD: build e push de imagem multi-arch (amd64/arm64) no release-please
      tag, hoje o CI não publica artefato nenhum.
- [ ] **2-3 comunidades piloto** instalando sozinhas, com observação.

**Gate (= MVP lançado):** piloto não-técnico instala em <30min só com o guia;
drill de backup/restore passa; uma comunidade usando semanalmente.

**Total estimado: ~28 semanas (~7 meses) de trabalho; 8-9 meses calendário
com folga.**

### Pós-MVP (ordenado por valor)

1. **API JSON v1** (token auth, OpenAPI): pré-requisito dos clientes.
2. **App nativo de backup de fotos** (upload automático do rolo de câmera,
   o recurso que PWA não consegue entregar). Candidato principal: **Flutter**
   (caminho comprovado pelo Immich); alternativa em avaliação: **Tauri
   mobile**.
3. **Tainá Conecta** (relay + backup off-site pago, ver D10).
4. **Guará (chat)**: Channels/Presence; agora sim o socket LiveView/Channel
   destampa.
5. Imagem de appliance NixOS para Pi (deploy v2).
6. Tainá Hospedada (multi-tenant gerenciado).

---

## 6. Métricas de sucesso revisadas

As métricas anteriores (99,5% uptime, NPS > 50, 3 comunidades em produção)
eram fantasia para um projeto pré-código. Novas:

| Métrica                    | Alvo                                  | Quando            |
| -------------------------- | ------------------------------------- | ----------------- |
| Instalação por não-técnico | < 30 min, 3 testes observados         | Gate Fase 4       |
| Drill de restore           | 100% sucesso, ensaiado                | Gate Fase 4       |
| Comunidade piloto ativa    | >= 1 usando semanalmente              | Lançamento +1 mês |
| Benchmark Pi 5             | gates das Fases 2-3                   | Contínuo          |
| Atualização sem medo       | update em 1 clique sem perda de dados | Fase 4            |

---

## 7. Riscos

| Risco                                                 | Mitigação                                                                                        |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Velocidade solo (roadmap anterior derrapou ~14 meses) | Fases verticais com gates; cortar escopo, nunca esticar prazo                                    |
| Performance RLS + libvips no Pi 5 desconhecida        | Benchmark é gate de fase, não afterthought                                                       |
| LiveView -> API depois custar caro                    | Contexts scope-first com contratos (`Behaviour`) desde já; API é controller fino                 |
| Comparação inevitável com Nextcloud/Immich            | Posicionamento: instalação e operação sem dor, comunidade-first, pt-BR; não paridade de features |
| Burnout / abandono                                    | Receita cedo (OC agora, Conecta pós-MVP); pilotos dão feedback que motiva                        |

---

## 8. Mudanças nos documentos existentes

1. `taina/README.md`: remover "APIs XRPC e eventos"; descrever entrega real
   (web UI LiveView; API pós-MVP).
2. `tekoa/ROADMAP.md`: substituir pelo plano da seção 5 desta RFC.
3. `RFC_ARQUITETURA.md` (RFC 001): marcar como "superada parcialmente pela
   RFC 002" no cabeçalho; o conteúdo de modelo de dados e RLS continua
   válido.
4. READMEs de serviço (`araci`, `kaa`, `karai`, `nhaman`, `arapa`): adicionar
   aviso "visão de longo prazo, não está no MVP".
