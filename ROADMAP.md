# Roadmap Tainá — MVP Realista

**Status:** Ativo
**Última atualização:** 2026-06-09
**Versão:** 3.0

> Este roadmap substitui integralmente a versão anterior (v2.0, "PWA Monolith",
> 2025-2026), conforme a [RFC 002](tecnico/RFC_002_MVP.md). Leia a RFC para o
> racional completo de cada decisão.

---

## Visão

**MVP = cofre de arquivos e fotos da comunidade, com instalação indolor.**

O Tainá não tenta substituir o Nextcloud hoje; compete em **experiência de
instalação e operação** — uma comunidade não-técnica coloca um servidor no ar
em menos de 30 minutos e nunca tem medo de atualizar.

Princípios desta versão:

- **Backend-first.** Contexts, autorização, storage e instalação vêm antes de
  qualquer UI. A interface web (Phoenix LiveView, server-rendered) é
  implementada em bloco quando o backend correspondente estiver estável.
- **Uma Tekoa (comunidade) por instância, com enforcement rígido** — o sistema
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

Premissa: dev solo, tempo parcial (~10–15h/semana). Cada fase é uma **fatia
vertical com critério de saída (gate)**; não se inicia a próxima sem fechar o
gate.

### Fase 0 — Fundação e correção (3 semanas)

- [ ] Corrigir `Ybira.upload/2` (corpo vazio — implementar de verdade).
- [ ] Corrigir campos inexistentes em `Ybira.check_capacity/2`.
- [ ] Corrigir colisão `File` (stdlib) em `Ybira.get_file/1`.
- [ ] Corrigir typo `:filehash` → `:file_hash` no changeset de arquivo.
- [ ] Padronizar `public_id` com `PublicId, autogenerate: true` em todos os
      schemas.
- [ ] **Ligar o RLS de verdade:** `prepare_query/3` que levanta exceção sem
      contexto de tekoa; `FORCE ROW LEVEL SECURITY`; teste de isolamento
      cross-tekoa com role não-superuser.
- [ ] Rota `/health` (healthcheck do compose aponta para ela).
- [ ] `Ecto.Migrator` no boot do release.
- [ ] Migração de enforcement single-tekoa (índice único).
- [ ] README do repo `taina` sem menção a XRPC; docs alinhados à RFC 002.

**Gate:** `mix precommit` verde; teste de isolamento RLS passando; container
sobe saudável com migrações automáticas.

### Fase 1 — Maraca: identidade e setup (6 semanas)

- [ ] Implementar `Maraca` cumprindo `Maraca.Behaviour` (convite, confirmação,
      autenticação, autorização, permissões, solicitações de acesso).
- [ ] `Taina.Scope` + plug/on_mount montando o scope da sessão.
- [ ] Setup de primeiro boot: criar a Tekoa única + admin.
- [ ] Login/logout, rotação de sessão, rate limit de tentativas.
- [ ] Convite por link + QR code (e-mail opcional).
- [ ] Admin: listar membros, papéis, desativar conta.

**Gate:** instância nova, do `docker compose up` ao segundo membro logado, sem
tocar em terminal depois do install. Cobertura de testes nos fluxos de auth.

### Fase 2 — Ybira: arquivos (8 semanas)

- [ ] Layout de storage por tekoa/ano/mês com SHA-256 registrado.
- [ ] Upload (multi-arquivo, validação por magic bytes, cota).
- [ ] Pastas hierárquicas, renomear/mover/excluir.
- [ ] Lixeira: soft delete, restauração, purga em 30 dias (Oban).
- [ ] Download com `Range` (streaming de vídeo/áudio básico).
- [ ] Quotas visíveis e configuráveis.

**Gate (benchmark em Pi 5):** upload de foto de 4MB + thumbnail < 3s;
streaming 1080p sem stutter; 10k arquivos paginados com fluidez. Testes de
cota e de isolamento RLS sobre arquivos.

### Fase 3 — Jaci-lite: fotos (5 semanas)

- [ ] Worker Oban de thumbnails (libvips, 2 tamanhos).
- [ ] Grade de fotos responsiva com scroll infinito.
- [ ] Linha do tempo agrupada por data (EXIF quando houver).
- [ ] Visualização fullscreen.

**Gate:** galeria com 5k fotos navegável com fluidez num Pi 5.

### Fase 4 — Operabilidade e pilotos (6 semanas)

- [ ] Backup com um clique + agendado (`pg_dump` + storage), **com restore
      verificado em CI** e runbook ensaiado.
- [ ] `install.sh` real (gera secrets, sobe compose, imprime URL do setup).
- [ ] Fluxo de atualização (pull + restart + migração).
- [ ] Documentação de Tailscale para acesso remoto.
- [ ] CD: build/push de imagem multi-arch no release.
- [ ] 2–3 comunidades piloto instalando sozinhas.

**Gate (= MVP lançado):** piloto não-técnico instala em <30min só com o guia;
drill de backup/restore passa; uma comunidade usando semanalmente.

**Total estimado: ~28 semanas de trabalho; 8–9 meses calendário.**

---

## Pós-MVP (ordenado por valor)

1. **API JSON v1** (token auth, OpenAPI) — pré-requisito dos clientes.
2. **App nativo de backup de fotos** (Flutter; alternativa Tauri mobile).
3. **Tainá Conecta** — relay de acesso remoto + backup off-site (assinatura,
   modelo Nabu Casa).
4. **Guará (chat)** — Channels/Presence.
5. Imagem de appliance NixOS para Raspberry Pi.
6. **Tainá Hospedada** — instância gerenciada multi-tenant (modelo Penpot).

---

## Métricas de Sucesso

| Métrica | Alvo | Quando |
|---|---|---|
| Instalação por não-técnico | < 30 min, 3 testes observados | Gate Fase 4 |
| Drill de restore | 100% sucesso, ensaiado | Gate Fase 4 |
| Comunidade piloto ativa | ≥ 1 usando semanalmente | Lançamento +1 mês |
| Benchmark Pi 5 | gates das Fases 2–3 | Contínuo |
| Atualização sem medo | update em 1 clique sem perda de dados | Fase 4 |

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

_Documento vivo — atualizado conforme o desenvolvimento avança._
