# Roadmap Tainá - PWA Superapp (2025-2026)

**Status:** Em Planejamento
**Última atualização:** 2025-10-01
**Versão:** 2.0 (PWA Monolith)

---

## Visão Geral

Tainá é uma plataforma de infraestrutura digital comunitária que permite às comunidades hospedar seus próprios serviços com total soberania de dados. A versão MVP será entregue como um **Progressive Web App (PWA)** que combina três serviços essenciais em um único aplicativo:

- **Ybira** - Sistema de arquivos (base para os demais)
- **Jaci** - Galeria de fotos e vídeos
- **Guará** - Mensagens (DM e grupos)

**Meta MVP (12 meses):** Instância Tainá instalável em servidores domésticos com VPN, totalmente funcional e utilizável pela comunidade.

---

## Timeline Geral

```
2025 ──────────────────────────────────────────────────► 2026
  │
  Q1 (Jan-Mar)     Q2 (Abr-Jun)     Q3 (Jul-Set)     Q4 (Out-Dez)
  Fundação         Ybira            Jaci + Guará     Polish + Deploy
     │                │                  │                  │
     ├─ Setup        ├─ Files          ├─ Photos         ├─ Mobile UX
     ├─ Auth         ├─ Upload         ├─ Messaging      ├─ Performance
     └─ PWA Base     └─ Preview        └─ Real-time      └─ Instalação
```

---

## Fase 1: Fundação (Meses 1-2 | Jan-Fev 2025)

**Objetivo:** Estabelecer base técnica e infraestrutura PWA.

### Entregáveis

#### Setup Técnico

- Projeto Phoenix com LiveView
- Docker Compose (PostgreSQL)
- Estrutura de serviços modular (lib/taina/{auth,core,ybira,jaci,guara})
- Schemas PostgreSQL separados por serviço
- Tailwind CSS + esbuild configurados

#### PWA Básico

- Manifest.json (nome, ícones, tema)
- Service Worker básico (offline fallback)
- Layout mobile-first responsivo
- Navigation shell (estrutura base)

#### Autenticação

- Modelos User e Community
- Autenticação session-based (email/password)
- LiveViews de Login/Register
- Authorization hooks (on_mount)

#### Navegação

- Dashboard / landing page
- Service switcher (bottom bar mobile, sidebar desktop)
- Avatar e logout
- Componentes compartilhados base

### Issues Relacionadas

- [ZEETECH-9: Setup Phoenix PWA](https://linear.app/zeetech-dev/issue/ZEETECH-9)
- [ZEETECH-10: Estrutura de Serviços](https://linear.app/zeetech-dev/issue/ZEETECH-10)
- [ZEETECH-11: Modelo Community](https://linear.app/zeetech-dev/issue/ZEETECH-11)
- [ZEETECH-12: Autenticação Session-based](https://linear.app/zeetech-dev/issue/ZEETECH-12)
- [ZEETECH-15: Phoenix PubSub](https://linear.app/zeetech-dev/issue/ZEETECH-15)
- [ZEETECH-34: PWA Setup](https://linear.app/zeetech-dev/issue/ZEETECH-34)
- [ZEETECH-35: Navigation Shell](https://linear.app/zeetech-dev/issue/ZEETECH-35)
- [ZEETECH-36: Auth UI](https://linear.app/zeetech-dev/issue/ZEETECH-36)

**Status:** Aguardando início

---

## Fase 2: Ybira - Sistema de Arquivos (Meses 3-4 | Mar-Abr 2025)

**Objetivo:** Implementar serviço fundacional de arquivos usado por Jaci e Guará.

### Por que Ybira primeiro?

Ybira é a **fundação** - tanto Jaci (fotos) quanto Guará (anexos) dependem dele para armazenamento de arquivos. Sem Ybira, os outros serviços não funcionam.

### Entregáveis

#### Backend (Context + Models)

- Schema `ybira` no PostgreSQL
- Modelo `Folder` (hierárquico, self-referential)
- Modelo `File` (metadata, associations)
- API pública: `upload/2`, `get_file/1`, `delete_file/2`, `list_files/2`, etc.
- Authorization (owner + admin community)

#### Interface (LiveView)

- File Browser (navegação estilo iPad Files)
- Breadcrumb navigation (Home > Docs > Work)
- Upload com drag-drop e progress tracking
- CRUD operations (rename, move, delete, new folder)
- Preview de documentos:
  - PDF viewer (PDF.js)
  - Image viewer (jpg, png, gif, webp, heic)
  - Video/audio player básico

#### Experiência do Usuário

- Grid/list view responsivo
- Empty states com CTAs
- Loading states (skeleton loaders)
- Touch-friendly (mobile)

### Issues Relacionadas

- [ZEETECH-37: Ybira Context + Schema](https://linear.app/zeetech-dev/issue/ZEETECH-37)
- [ZEETECH-38: File Browser LiveView](https://linear.app/zeetech-dev/issue/ZEETECH-38)
- [ZEETECH-39: Upload & CRUD](https://linear.app/zeetech-dev/issue/ZEETECH-39)
- [ZEETECH-40: Document Preview](https://linear.app/zeetech-dev/issue/ZEETECH-40)

**Status:** Não iniciado
**Dependências:** Fase 1 (Auth, PWA base)

---

## Fase 3: Jaci - Galeria de Fotos (Meses 5-6 | Mai-Jun 2025)

**Objetivo:** Permitir upload, organização e visualização de fotos/vídeos.

### Entregáveis

#### Backend (Context + Models)

- Schema `jaci` no PostgreSQL
- Modelo `Photo` (vinculado a `Ybira.File`)
- Modelo `Album` (cover photo, slug)
- Many-to-many: `AlbumPhoto` (position tracking)
- EXIF extraction (data/hora, localização, câmera)
- Integração com Ybira API

#### Interface (LiveView)

- Upload de fotos/vídeos:
  - Múltiplos arquivos simultaneamente
  - Camera capture (mobile)
  - Preview antes do upload
  - Progress tracking por arquivo
- Gallery views:
  - Grid responsivo (2-6 colunas)
  - Timeline agrupada por data
  - Infinite scroll
  - Fullscreen view (reusar preview Ybira)
- Gerenciamento de álbuns:
  - Criar/editar/deletar álbuns
  - Adicionar/remover fotos
  - Set cover photo
  - URL-friendly slugs

#### Experiência do Usuário

- Thumbnails otimizados
- Multi-select com batch actions
- Empty states ("Nenhuma foto ainda")
- Touch gestures (pinch to zoom)

### Issues Relacionadas

- [ZEETECH-41: Jaci Context + Schema](https://linear.app/zeetech-dev/issue/ZEETECH-41)
- [ZEETECH-42: Photo Upload](https://linear.app/zeetech-dev/issue/ZEETECH-42)
- [ZEETECH-43: Gallery Views](https://linear.app/zeetech-dev/issue/ZEETECH-43)

**Status:** Não iniciado
**Dependências:** Fase 2 (Ybira funcionando)

---

## Fase 4: Guará - Mensagens (Meses 7-9 | Jul-Set 2025)

**Objetivo:** Messaging em tempo real (DM + grupos, SEM status).

### Entregáveis

#### Backend (Context + Models)

- Schema `guara` no PostgreSQL
- Modelo `Conversation` (direct/group)
- Modelo `Participant` (read tracking)
- Modelo `Message` (text + file attachments)
- Integração com Ybira (anexos)
- Authorization (apenas participantes)

#### Real-time (Phoenix Channels + Presence)

- Phoenix Channels: `conversation:{id}`
- Presence tracking (online/offline)
- Typing indicators
- PubSub para notificações

#### Interface (LiveView)

- Conversation List:
  - Preview da última mensagem
  - Unread count badges
  - Real-time updates
  - Criar nova conversa (DM ou grupo)
- Message Thread:
  - Timeline reversa (scroll to bottom)
  - Message bubbles (sender left/right)
  - Infinite scroll (histórico)
  - Composer (auto-resize textarea)
  - Online status indicators
- Media Attachments:
  - Imagens (inline thumbnails)
  - Vídeos (player com thumbnail)
  - Áudios (player inline)
  - Upload via Ybira

#### Experiência do Usuário

- Real-time delivery
- Read receipts (via last_read_at)
- Virtual keyboard handling
- Pull-to-refresh para histórico

### Issues Relacionadas

- [ZEETECH-44: Guará Context + Schema](https://linear.app/zeetech-dev/issue/ZEETECH-44)
- [ZEETECH-45: Conversation List](https://linear.app/zeetech-dev/issue/ZEETECH-45)
- [ZEETECH-46: Message Thread + Channels](https://linear.app/zeetech-dev/issue/ZEETECH-46)
- [ZEETECH-47: Media Attachments](https://linear.app/zeetech-dev/issue/ZEETECH-47)

**Status:** Não iniciado
**Dependências:** Fase 2 (Ybira para anexos), Fase 1 (PubSub setup)

**Importante:** NO status feature (não é WhatsApp). Apenas DM e grupos.

---

## Fase 5: Polish & Deploy (Meses 10-12 | Out-Dez 2025)

**Objetivo:** Experiência mobile-first polida + instalação plug & play.

### Entregáveis

#### UX/UI Polish

- Touch interactions refinadas (tap, swipe, long-press)
- Touch targets mínimo 44x44px
- Smooth animations e transitions
- Haptic feedback (onde suportado)
- Virtual keyboard handling perfeito

#### PWA Instalação

- Install prompt customizado
- "Add to Home Screen" flow (iOS/Android)
- App-like experience (sem browser chrome)
- Offline detection com mensagens claras

#### Performance

- Lazy loading de imagens/components
- Code splitting por rota
- Lighthouse score 90+ em todas as métricas
- Sub-100ms response times

#### Acessibilidade (A11y)

- Keyboard navigation completa
- Screen reader support (aria-labels)
- Focus indicators visíveis
- Color contrast WCAG AA

#### Deploy Infrastructure

- Docker Compose one-command setup
- VPN configuration (WireGuard)
- Reverse proxy + TLS automático
- Automated database backups
- Phoenix LiveDashboard para monitoring

#### Documentação

- Guia de instalação para admins
- Guia de uso para usuários
- Guia de contribuição para devs
- Documentação de arquitetura

### Issues Relacionadas

- [ZEETECH-48: Polish Mobile-First UX](https://linear.app/zeetech-dev/issue/ZEETECH-48)
- Novas issues de deployment (a criar)

**Status:** Não iniciado
**Dependências:** Todas as fases anteriores completas

---

## Pós-MVP (2026+)

Após MVP, prioridades baseadas em feedback da comunidade:

### Serviços Adicionais

- **Araci** - Streaming de vídeo/áudio (biblioteca de mídia)
- **Karai** - Monitoring & segurança
- **Ka'a** - Smart home / IoT
- **Nhaman** - Dashboard administrativo

### Melhorias de Plataforma

- **Clientes nativos** (extrair Jaci/Guará como apps móveis)
- **Electric SQL** - Sync offline-first
- **Federação** - Comunicação entre comunidades
- **AI local** - Organização de fotos, busca inteligente
- **Full-text search** - Mensagens, arquivos
- **Versioning** - Histórico de arquivos (Ybira)

---

## Métricas de Sucesso MVP

Ao final dos 12 meses, consideraremos o MVP bem-sucedido se:

- **Instalação:** Setup completo em menos de 30 minutos
- **Performance:** Menos de 100ms response times em 95% das operações
- **Uptime:** 99.5% uptime em instâncias comunitárias
- **UX:** Lighthouse PWA score 90+
- **Funcionalidade:** Todos os 3 serviços (Ybira, Jaci, Guará) funcionais
- **Hardware:** Roda em Raspberry Pi 5 (8GB) ou Mini PC N100
- **Comunidade:** Ao menos 3 comunidades usando em produção
- **Feedback:** Net Promoter Score (NPS) maior que 50

---

## Como Contribuir

### Para Desenvolvedores

1. Veja as [Issues abertas no Linear](https://linear.app/zeetech-dev/team/ZEETECH/all)
2. Leia o [CONTRIBUTING.md](CONTRIBUTING.md)
3. Participe das [Discussions](https://github.com/taina-org/tekoa/discussions)
4. Confira a [RFC de Arquitetura](tecnico/RFC_ARQUITETURA.md)

### Para Comunidades (Early Adopters)

- **Beta testing:** Teste instâncias durante desenvolvimento
- **Feedback:** Compartilhe casos de uso e necessidades
- **Documentação:** Ajude a melhorar guias de uso
- **Design:** Sugestões de UX/UI

### Para Todos

- Dê star no projeto
- Reporte bugs nas Issues
- Sugira features nas Discussions
- Compartilhe o projeto em suas redes

---

## Feedback e Discussões

**Quer dar feedback sobre o roadmap?**

- [GitHub Discussions - Roadmap](https://github.com/taina-org/tekoa/discussions)
- Email: zoey.spessanha@zeetech.io
- Issues: Para problemas específicos ou bugs

**Perguntas Frequentes:**

- **Por que PWA e não app nativo?** Desenvolvimento mais rápido, uma codebase, funciona em todos os dispositivos. Apps nativos virão no pós-MVP.
- **Por que Ybira primeiro?** É a fundação - Jaci e Guará dependem dele para armazenamento.
- **Posso hospedar na nuvem?** Sim! Mas o foco é self-hosting em hardware próprio.
- **Terá versão mobile?** O PWA **é** a versão mobile. Funciona como app nativo quando instalado.

---

## Licença

Este roadmap e o projeto Tainá estão sob a [GNU Affero General Public License v3.0](LICENSE).

---

**Última atualização:** 2025-10-01
**Mantenedores:** [@zoedsoupe](https://github.com/zoedsoupe) e comunidade
**Status do projeto:** Planejamento (iniciando Q1 2025)

---

_Este é um documento vivo - será atualizado conforme o desenvolvimento avança e o feedback da comunidade é incorporado._
