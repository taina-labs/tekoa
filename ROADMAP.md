# Roadmap Tainá, MVP Realista

**Status:** Ativo
**Última atualização:** 2026-10-05
**Versão:** 3.1
**Lançamento alvo:** `0.1.0` (primeiro lançamento público)

> Este roadmap substitui integralmente a versão anterior (v2.0, "PWA Monolith",
> 2025-2026), conforme a [RFC 002](tecnico/RFC_002_MVP.md). Leia a RFC para o
> racional completo de cada decisão.
>
> **O que mudou na v3.1 (2026-10-05):** as listas de tarefas foram marcadas
> conforme o código realmente mergesado, e a Fase 0 ganhou uma subdivisão
> (operabilidade) que é o caminho crítico do lançamento. O gate do `0.1.0`
> está isolado em [O que falta para o 0.1.0](#o-que-falta-para-o-010).

---

## Visão

**MVP = cofre de arquivos e fotos da comunidade, com instalação indolor.**

O Tainá não tenta substituir o Nextcloud hoje; compete em **experiência de
instalação e operação**, uma comunidade não-técnica coloca um servidor no ar
em menos de 30 minutos e nunca tem medo de atualizar.

Princípios desta versão:

- **Backend-first.** Contexts, autorização, storage e instalação vêm antes de
  qualquer UI. A interface web (Phoenix LiveView, server-rendered) é
  implementada em bloco quando o backend correspondente estiver estável.
- **Uma Tekoa (comunidade) por instância, com enforcement rígido**: o sistema
  recusa a criação de uma segunda comunidade (função do context + índice único
  no banco).
- **Chat (Guará) fora do MVP.** Comunidades já usam Signal/WhatsApp; arquivos
  e fotos são onde a soberania de dados importa. Guará volta no pós-MVP.
- **PWA offline-first descartado.** App nativo de backup de fotos
  (candidato principal: **Flutter**; alternativa: **Tauri mobile**) vem no
  pós-MVP, atrás de uma API JSON.
- **Stack:** Elixir 1.20 / OTP 28, Phoenix 1.8, PostgreSQL 18 com Row-Level
  Security (isolamento por comunidade), Oban para jobs, libvips para mídia.

---

## Plano de Implementação

Premissa: dev solo, tempo parcial (~10-15h/semana). Cada fase é uma **fatia
vertical com critério de saída (gate)**; não se inicia a próxima sem fechar o
gate.

Legenda: `[x]` mergesado e verificado no código, `[~]` parcial, `[ ]` pendente.

### Fase 0, Fundação e correção (3 semanas)

- [x] Corrigir `Ybira.upload/2` (corpo vazio, implementar de verdade).
- [x] Corrigir campos inexistentes em `Ybira.check_capacity/2`.
- [x] Corrigir colisão `File` (stdlib) em `Ybira.get_file/1`.
- [x] Corrigir typo `:filehash` -> `:file_hash` no changeset de arquivo.
- [x] Padronizar `public_id` com `PublicId, autogenerate: true` em todos os
      schemas.
- [x] **Ligar o RLS de verdade:** `prepare_query/3` que levanta exceção sem
      contexto de tekoa; `FORCE ROW LEVEL SECURITY`; teste de isolamento
      cross-tekoa com role não-superuser.
- [x] Rota `/health` (healthcheck do compose aponta para ela).
- [x] `Ecto.Migrator` no boot do release.
- [x] Migração de enforcement single-tekoa (índice único).
- [x] README do repo `taina` sem menção a XRPC; docs alinhadas à RFC 002.

**Gate:** `mix precommit` verde; teste de isolamento RLS passando; container
sobe saudável com migrações automáticas.

> **Status (2026-10-05): gate fechado.** `mix precommit` verde (221 testes,
> dialyzer e credo limpos). Migrações no boot e `/health` presentes.

#### Fase 0b, Operabilidade e deploy (3 semanas)

Esta subdivisão não existia na v3.0 do roadmap. Ela nasceu do que a Fase 4
exigia e do que só ficou visível quando alguém tentou rodar de verdade.

- [x] Dockerfile de produção (Elixir 1.20 / OTP 28, multi-arch).
- [x] Migrações automáticas no boot do release (`Taina.Release.migrate`).
- [x] `install.sh`: gera segredos, escreve `.env`, sobe compose, imprime a URL
      do assistente de setup. Idempotente (nunca sobrescreve um `.env`).
- [x] `update.sh`: baixa a imagem, reinicia, confere a saúde, e diz como
      voltar atrás se der errado.
- [x] Proxy reverso com HTTPS automático (Caddy; Let's Encrypt com
      `TAINA_DOMAIN`, certificado local sem ele).
- [x] Só o Caddy exposto. O app fala HTTP na rede interna do Compose.
- [x] Backup agendado com verificação de restore em CI (`mix taina.backup.verify`).
- [x] Runbook de backup e restauração documentado.
- [x] Guia de acesso remoto (Tailscale, Headscale, WireGuard, domínio).
- [x] Workflow de CD: build e push multi-arch (amd64/arm64) no release-please.
- [x] **`/app/backups` criado e com dono no Dockerfile.** Sem isso o volume de
      backup nascia root-owned e o Nhaman falhava com `:eacces`.
- [x] **Rotação de log dos containers** (10 MB x 3 arquivos). O logger padrão
      do Docker nunca rotaciona e enche o disco até o PostgreSQL cair.
- [x] **Sondagem de saúde de dentro do container do app.** A sonda anterior
      (`https://localhost/health`) nunca podia passar com `TAINA_DOMAIN`
      preenchido, porque o Caddy serve só o domínio.
- [x] **`build-image.sh`**: instalar a partir do código-fonte, para quando não
      há imagem publicada (antes do primeiro release, ou num fork).
- [ ] **Publicar a primeira imagem.** O job de docker existe e roda em release,
      mas **nenhuma release foi criada ainda** e o `ghcr.io/taina-labs/taina`
      está vazio. Sem isso o `install.sh` não tem o que baixar.
- [ ] **Vistoriar o `install.sh` numa máquina real**, do zero, cronometrado.
- [ ] **Ensaio de restore com pessoa não-técnica** (o gate do runbook).
- [ ] **Ensaio do `update.sh`**: subir, atualizar, conferir, e voltar atrás.

**Gate:** uma pessoa não-técnica instala em menos de 30 minutos seguindo só o
`guias/instalacao.md`, num drill de restore, e numa atualização.

### Fase 1, Maraca: identidade e setup (6 semanas)

- [x] Implementar `Maraca` cumprindo `Maraca.Behaviour` (convite, confirmação,
      autenticação, autorização, permissões, solicitações de acesso).
- [x] `Taina.Scope` + plug/on_mount montando o scope da sessão.
- [x] Setup de primeiro boot: criar a Tekoa única + zelador(a).
- [x] Login/logout, rotação de sessão, rate limit de tentativas.
- [x] Convite por link + QR code (e-mail **descartado** por completo, RFC 003
      D10: não há coluna de e-mail nem aparato de confirmação).
- [x] Admin: listar membros, papéis, desativar conta.
- [x] Migração `admin`->`zelador`, `member`->`morador`, múltiplos zeladores
      (RFC 003 D3).
- [x] Zonas `casa`/`praca` com a regra de leitura em todo caminho de leitura,
      incluindo thumbnail e download (RFC 003 D1/D2).
- [x] `request_access` generalizado para qualquer morador(a) + `list_my_requests`
      (RFC 003 D4).

**Gate:** instância nova, do `docker compose up` ao segundo morador logado, sem
sem tocar em terminal depois do install. Cobertura de testes nos fluxos de auth.

> **Status (2026-10-05): gate fechado no código, ainda não validado numa
> máquina real.** Todos os itens implementados e testados (221 testes), mas o
> gate fala em "do `docker compose up` ao segundo morador logado, sem tocar em
> terminal", e isso só se prova no [ensaio](#o-que-falta-para-o-010).

### Fase 2, Ybira: arquivos (8 semanas)

- [x] Layout de storage por tekoa/ano/mês com SHA-256 registrado.
- [x] Upload (multi-arquivo, validação por magic bytes, cota).
- [x] Pastas hierárquicas, renomear/mover/excluir.
- [x] Lixeira: soft delete, restauração, purga em 30 dias (Oban).
- [x] Download com `Range` (streaming de vídeo/áudio básico).
- [x] Quotas visíveis e configuráveis.

**Gate (benchmark em Pi 5):** upload de foto de 4MB + thumbnail < 3s;
streaming 1080p sem stutter; 10k arquivos paginados com fluidez. Testes de
cota e de isolamento RLS sobre arquivos.

> **Status (2026-10-05): implementação e testes fechados; benchmark de hardware
> real pendente.** O gate exige medição num Pi 5, e a suíte roda em CI num
> runner x86. É o gate mais barato de fechar de todos: é medir, não codar.

### Fase 3, Jaci-lite: fotos (5 semanas)

- [x] Worker Oban de thumbnails (2 tamanhos).
- [x] Grade de fotos responsiva com scroll infinito.
- [x] Linha do tempo agrupada por data (EXIF quando houver).
- [x] Visualização fullscreen.

**Gate:** galeria com 5k fotos navegável com fluidez num Pi 5.

> **Status (2026-10-05): implementação e testes fechados; benchmark pendente.**
> Mesma situação da Fase 2.

### Fase 4, Transparência e pilotos (6 semanas)

A v3.0 do roadmap chamava esta fase de "Operabilidade e pilotos". A operabilidade
virou a Fase 0b, porque ela é o caminho crítico do lançamento e não podia ficar
espremida no fim. O que sobra aqui é a **transparência** (RFC 003, seção 6), que
a v3.0 também não tinha listado.

- [x] **Mural:** livro de registro social append-only (`maraca.events`), com a
      regra de que uma escrita no mural nunca desfaz o ato que ela descreve.
- [x] **Painel / "Saúde da comunidade":** disco, último backup, jobs pendentes,
      moradores, versão, em pt-BR comum. Sem cor de alerta, sem gráficos.
- [x] **Verificação de update no painel:** opt-in, desligada por padrão, e
      **nunca** envia dado da tekoa para lugar nenhum.
- [ ] **Vistoriar o `install.sh` numa máquina real**, do zero, cronometrado.
- [ ] **2-3 comunidades piloto** instalando sozinhas, com observação.

**Gate (= MVP lançado):** piloto não-técnico instala em <30min só com o guia;
drill de backup/restore passa; uma comunidade usando semanalmente.

**Total estimado:** ~28 semanas de trabalho na estimativa original da v3.0. Com
a Fase 0b (operabilidade) e a transparência somadas, ~32 semanas de trabalho,
9-10 meses calendário.

---

## O que falta para o 0.1.0

O `0.1.0` é o primeiro lançamento público. O caminho crítico é curto e não tem
nada de codar novo: é **publicar e provar**.

### Bloqueadores (sem isso não existe release)

1. **Publicar a primeira imagem.** O job de docker em
   `.github/workflows/release-please.yml` só roda quando o release-please cria
   uma release, e nenhuma foi criada. Isso quer dizer que `install.sh` falha na
   linha do `docker compose pull` para qualquer pessoa que siga o guia. Publicar
   uma tag de verdade conserta.
2. **Conferir a visibilidade do pacote no ghcr.** O primeiro push de um pacote
   novo no registry do GitHub nasce privado. Se ele ficar privado, o
   `docker compose pull` falha com `denied` mesmo com a release publicada. Tem
   que ser público para o guia funcionar.
3. **Push do `main`.** Existe commit local ainda não enviado.

> O smoke test da imagem (`.github/workflows/docker-smoke.yml` no repo `taina`)
> roda em toda tag `v*` e sobe o container de verdade: migrações no boot,
> `/health` respondendo, e os dois diretórios de dados graváveis pelo usuário
> `nobody`. Ele não substitui o ensaio manual, mas garante que o roteiro de
> instalação não quebrou antes de a pessoa tentar.

### Provas de que funciona (o gate do MVP)

4. **Instalação limpa e cronometrada** numa máquina real, seguindo só o
   `guias/instalacao.md`, sem ajuda técnica externa. Anotar onde travar.
5. **Drill de backup/restore** com 100% de sucesso, feito por alguém não
   técnico. É o item mais fácil de pular e o mais caro de pular.
6. **Ensaio do `update.sh`**: subir, atualizar, conferir a saúde, e voltar para
   a versão anterior. Ninguém testou esse caminho ainda.
7. **Benchmark no Pi 5** (gates das Fases 2 e 3): foto de 4MB + thumbnail < 3s,
   1080p sem stutter, 10k arquivos fluidos, galeria com 5k fotos.

### Recomendado, não bloqueia

- Assentar o `docs/ui-prompts/` no repositório (slice 01 responsive pass, slice
  04 Nhaman cuidados). Hoje ele vive no `.git/info/exclude`, então quem chega
  pelo GitHub não vê a especificação que orientou o build da interface.
- Conferir os guias citados nas READMEs do `taina` e do `oka`: alguns apontam
  para caminhos que não existem mais.

### Ordem sugerida

1. Push do `main`, tag de release, imagem publicada, pacote público.
2. Instalar numa máquina real do zero, cronometrando, e consertar o que travar.
3. Drill de restore e ensaio do update.
4. Benchmark no Pi 5.
5. Anunciar.

O passo 5 vem depois dos outros de propósito: um anúncio de `0.1.0` que
convida a instalar é um convite para achar o primeiro bug, e a taxa de respostas
a bugs de instalação cai muito quando você já/installou você mesmo antes.

---

## Pós-MVP (ordenado por valor)

1. **API JSON v1** (token auth, OpenAPI): pré-requisito dos clientes.
2. **App nativo de backup de fotos** (Flutter; alternativa Tauri mobile).
3. **Tainá Conecta**: relay de acesso remoto + backup off-site (assinatura,
   modelo Nabu Casa).
4. **Guará (chat)**: Channels/Presence.
5. **Criptografia E2E no Ybira.** É o que permitiria dizer honestamente que o
   zelador *não consegue* ler a casa de um morador. Hoje a privacidade da casa
   é uma promessa de software e confiança, não criptográfica (RFC 003 D7).
6. **Assembleia** (governança coletiva): Sensitive viram propostas votadas. Fora
   do MVP do RFC 002, detalhado na seção 8 da RFC 003.
7. **Imagem de appliance NixOS** para Raspberry Pi.
8. **Tainá Hospedada**: instância gerenciada multi-tenant (modelo Penpot).

O produto institucional para universidades foi avaliado e **adiado** para depois
do MVP, como produto separado. Ver a
[RFC 004](tecnico/RFC_004_PRODUTO_UNIVERSIDADE.md).

---

## Métricas de Sucesso

| Métrica                    | Alvo                                  | Quando            |
| -------------------------- | ------------------------------------- | ----------------- |
| Instalação por não-técnico | < 30 min, 3 testes observados         | Gate Fase 4       |
| Drill de restore           | 100% sucesso, ensaiado                | Gate Fase 4       |
| Comunidade piloto ativa    | >= 1 usando semanalmente              | Lançamento +1 mês |
| Benchmark Pi 5             | gates das Fases 2-3                   | Contínuo          |
| Atualização sem medo       | update em 1 clique sem perda de dados | Fase 0b           |

---

## Guias do operador

- [Instalação](guias/instalacao.md): colocar no ar do zero, em menos de 30
  minutos.
- [Acesso remoto](guias/acesso-remoto.md): Tailscale, Headscale, WireGuard,
  domínio público.
- [Backup e restauração](guias/runbook-backup-restore.md): procedimento e
  ensaio (drill).
- [Hardware](guias/compras-hardware.md): o que comprar e como montar.
- [Contribuindo via GitHub](guias/contribuindo-github.md): fluxo de branches e
  pull requests.

---

## Como Contribuir

1. Leia o [Guia de Contribuição](CONTRIBUTING.md).
2. Participe das [Discussions](https://github.com/taina-labs/tekoa/discussions).
3. Confira a [RFC 002](tecnico/RFC_002_MVP.md) antes de propor mudanças de
   arquitetura.

---

## Licença

Este roadmap e o projeto Tainá estão sob a
[GNU Affero General Public License v3.0](LICENSE).

**Mantenedores:** [@zoedsoupe](https://github.com/zoedsoupe) e comunidade

_Documento vivo, atualizado conforme o desenvolvimento avança._
