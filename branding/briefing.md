# Briefing de Design - Identidade Visual Tainá

## Contexto do Projeto

**Tainá** é um ecossistema de nuvem privada e auto-hospedada, construído com tecnologia brasileira para comunidades que buscam independência digital. O nome significa "estrela da manhã" em Tupi-Guarani, simbolizando o guia para um novo amanhecer tecnológico.

A documentação e colaboração pública do projeto serão centralizadas em **Tekoá**, um repositório aberto de conhecimento cujo nome remete ao espaço coletivo de uma aldeia, um ponto de encontro da comunidade global.

### Produto web e app Flutter

Tainá tem uma interface web server-rendered em Phoenix LiveView e um app Flutter para iOS e Android. As duas superfícies compartilham identidade, tokens e vocabulário, sem impor a mesma interação em plataformas diferentes.

- **Unificação do cofre** em uma experiência consistente
- **Navegação própria de cada superfície**, sem simular app nativo no navegador
- **Mobile-first** na web e clientes Flutter no telefone
- **Ciclo completo**: capturar, guardar e recuperar

O 1.0.0 reúne Maracá, Ybira, Jaci e Nhaman, acrescenta o espelho unidirecional do rolo no app e o backup cifrado fora da caixa. Guará e os demais contextos ficam para depois.

---

## Conceito Central

### Narrativa Visual

A identidade deve contar a história de uma jornada comunitária guiada pela estrela da manhã (Tainá), unindo tecnologia descentralizada e a sabedoria ancestral dos povos originários. A estética integra **formas geométricas modernas** com **elementos orgânicos inspirados na fauna, flora e arte indígena brasileira**, criando uma marca que é, ao mesmo tempo, **ancestral, tecnológica e acolhedora**.

### Valores Fundamentais

- **Autonomia**: Controle total sobre dados e infraestrutura
- **Comunidade**: Pertencimento e colaboração global
- **Brasilidade**: Cultura e identidade sem estereótipos
- **Inovação**: Soluções tecnológicas com DNA próprio
- **Acessibilidade**: Design inclusivo, simples e universal

---

## Diretrizes da Marca

### Personalidade

- **Confiável**: mas não corporativa
- **Inovadora**: sem intimidação técnica
- **Brasileira**: sem caricaturas
- **Técnica**: mas convidativa
- **Moderna**: sem se prender a tendências passageiras

### Tom de Voz Visual

- Otimista e inspirador
- Profissional, sem rigidez
- Acolhedor, diverso e humano
- Tecnológico com identidade cultural

---

## Elementos Visuais

### Logo Principal - Tainá

**Conceito**:

- Símbolo central de **estrela estilizada**
- Combinação de **geometria tecnológica** e **formas orgânicas**
- **Ascensão** como metáfora de progresso e amanhecer
- **Rede de nós conectados**: refletindo descentralização e comunidade

**Variações necessárias**:

- Logo completo (símbolo + tipografia)
- Versão minimalista (apenas símbolo)
- Versão monocromática
- Adaptação para fundos claros/escuros
- Favicon e ícones para apps

---

### Paleta de Cores

Cores inspiradas na **natureza brasileira** (aurora, floresta, terra) e no **céu noturno**, reforçando ancestralidade e modernidade:

- **Primárias**: Tons quentes de amanhecer, azul profundo do céu noturno e branco claro das estrelas.
- **Secundárias**: Verdes de floresta, azuis tropicais e tons terrosos, criando equilíbrio e identidade cultural.
- **Funcionais**: Paleta clara e acessível para estados de alerta, sucesso, erro e informação, mantendo contraste adequado.

---

### Tipografia

- **Fonte principal**: Sans-serif moderna, limpa e com excelente legibilidade em telas.
- **Fonte de apoio**: Serifas elegantes para leituras longas e uma monoespaçada robusta para trechos de código.
- **Diretrizes**: Suporte completo a português, inglês e outras línguas, reforçando o caráter internacional do projeto.

---

## Aplicações dos Serviços

### Contextos do cofre e do 1.0.0

| Contexto | Função | Nome | Cor Base | Status |
| --- | --- | --- | --- | --- |
| Identidade e acesso | Contas, convites, papéis e concessões | **Maracá** | Ember | Existe |
| Armazenamento | Arquivos, pastas, zonas e cotas | **Ybira** | Verde-escuro | Existe |
| Fotos e memórias | Galeria, linha do tempo e espelho do rolo | **Jaci** | Creme/Dourado | Galeria existe; espelho no 1.0.0 |
| Operação | Instalação, saúde, backup e atualização | **Nhaman** | Azul-elétrico | Local existe; backup externo no 1.0.0 |
| Clientes | App iOS e Android | **Okena** | Usa os tokens do produto | App existe; espelho no 1.0.0 |

### Visão posterior

| Contexto | Função | Nome | Status |
| --- | --- | --- | --- |
| Mensagens e assembleia | Chat e deliberação comunitária | **Guará** | Depois do 1.0.0 |
| Vídeo | Biblioteca e streaming | **Araci** | Ideia |
| Música e áudio | Playlists, podcasts e audiolivros | **Arapa** | Ideia |
| Casa inteligente | Automação e sensores | **Ka'a** | Ideia |
| Monitoração | Observabilidade da caixa | **Karai** | Ideia |

Cada contexto pode manter cor de acento própria, mas compartilha componentes e padrões visuais do produto.

---

## Diretrizes de Interface (UI)

### Princípios de Design

1. **Minimalismo Tropical**: Estética clean, com detalhes culturais sutis
2. **Hierarquia Clara**: Layouts bem estruturados e intuitivos
3. **Microinterações**: Feedback suave para criar proximidade
4. **Dark Mode Prioritário**: Interfaces que evocam o céu noturno
5. **Responsividade Total**: Design fluido, mobile-first
6. **Coesão entre Serviços**: Navegação consistente, componentes reutilizáveis

### Navegação do cofre

#### Mobile (Prioritário)

- **Bottom Navigation Bar**: poucos itens principais do cofre, definidos pelos boards atuais
- **Ícones + labels**: Sempre visíveis para clareza
- **Touch targets**: Mínimo 44x44px
- **Transições suaves**: Entre serviços sem reload
- **Gestures**: Swipe horizontal para alternar entre serviços

#### Desktop/Tablet

- **Sidebar persistente**: Expandida com ícones + nomes
- **Topbar**: Busca global, notificações, perfil
- **Keyboard shortcuts**: Navegação rápida (Cmd+1, Cmd+2...)
- **Multi-pane**: Aproveitar espaço horizontal

### Componentes Base

- **Cards** e **botões** simples e funcionais, com feedback claro
- **Ícones lineares** com consistência visual entre serviços
- **Espaçamento amplo**: criando respiro e legibilidade
- **Animações suaves**: reforçando uma experiência acolhedora
- **Navegação compartilhada**: componentes coerentes entre Documentos, Galeria, Moradores e operação
- **Upload Progress**: Component compartilhado (usado em todos os serviços)

### Padrões de Interface

- **Home/Dashboard**: Overview de todos os serviços com acesso rápido
- **Service Views**: Cada serviço ocupa tela completa quando ativo
- **Shared Components**: Preview de arquivos, modal de upload, notificações
- **Formulários**: Feedback em tempo real, validação inline
- **Modais**: Acessíveis e discretos, backdrop com blur
- **Notificações**: Minimalistas, não intrusivas, agrupadas por serviço

---

## Aplicações Práticas

### Website/Landing

- Gradientes inspirados no amanhecer
- Ilustrações de comunidade e servidores
- Demonstração visual do cofre e dos clientes
- Seção clara sobre capturar, guardar e recuperar

### Interface web e app Flutter

- **Tema escuro**: tema único, sem variante clara no produto atual
- **Acentos por contexto**: cada serviço com sua cor de destaque
- **Navegação por plataforma**: bottom nav quando fizer sentido no telefone e sidebar no desktop
- **Upload compartilhado**: estados de progresso, pausa, erro e conclusão coerentes
- **Preview e galeria**: componentes reutilizados para arquivos e fotos
- **Espelho no app**: estado e configuração ficam no Flutter, sem criar uma superfície duplicada na web

### Onboarding e clientes

- **Convite como chegada**: link, QR e deep link conduzem a pessoa à comunidade
- **Instalação guiada da caixa**: primeiro boot compreensível para quem não é técnico
- **Ícones e abertura**: assets próprios de iOS e Android derivados dos tokens
- **Validação local**: simulador iOS e emulador Android são portões do 1.0.0

### Documentação (Tekoá)

- Leitura confortável com tipografia hierárquica
- Busca eficiente e destaque para blocos de código
- Seção dedicada a design patterns do cofre
- Guias de contribuição UI/UX

---

## Referências de navegação

A referência não é reproduzir um superapp. O cofre precisa de poucas ações por tela, descoberta progressiva e nenhuma gamificação. Material Design e Apple HIG orientam padrões de plataforma; os boards do Penpot continuam sendo a fonte visual do Tainá.

## Referências e Inspirações

- **Arte indígena**: Geometria, simbologia, conexão ancestral
- **Natureza tropical**: Cores, texturas e formas orgânicas
- **Arquitetura moderna**: Linha limpa, curvas, harmonia espacial
- **Design de software**: Projetos open source de identidade forte
- **Cyberpunk tropical**: Estética de futuro com raízes culturais
- **Ecossistemas coesos**: Navegação fluida e integração de serviços
- **Design systems brasileiros**: Estética local com padrões internacionais

**Evitar:** Clichês (futebol, carnaval) e estereótipos, tribalismo literal, cores saturadas demais, complexidade visual desnecessária, apropriação cultural, densidade oriental excessiva sem contexto.

---

## Considerações Técnicas

### Formatos e Assets

- **Logos**: SVG (vetorial), PNG em múltiplas resoluções
- **Ícones**: SVG inline para performance, sprite sheets para coleções
- **Fontes**: WOFF2 otimizado, fallbacks web-safe
- **Imagens**: WebP com fallback JPEG, lazy loading

### Requisitos dos clientes

- **Tokens compartilhados**: Penpot, CSS da caixa e espelho do Okena sem deriva
- **Ícones e splash screens**: assets próprios para iOS e Android
- **Sem offline-first**: polling, cache ou background somente dentro do escopo atual
- **Sem PWA obrigatório**: a web é LiveView e os clientes móveis são Flutter

### Acessibilidade (A11y)

- **Padrões WCAG AAA** em todo o produto
- **Ícones com labels**: Sempre acompanhados de texto (visible ou sr-only)
- **Cores secundadas por símbolos**: Não depender apenas de cor
- **Keyboard navigation**: Tab order lógico, shortcuts documentados
- **Screen reader support**: ARIA labels, semantic HTML
- **Touch targets**: Mínimo 44x44px em mobile
- **Contrast ratios**: 7:1 para textos, 3:1 para UI elements

### Performance

- **Web enxuta**: assets e imagens otimizados, carregamento sob demanda
- **App verificado**: analyzer, testes e builds locais em iOS e Android
- **Mídia com streaming**: uploads e downloads não carregam arquivos grandes inteiros na memória
- **Benchmark separado**: resultados em VM não são apresentados como medições de Pi 5

### Responsive Breakpoints

```
- Mobile small: 320px (base)
- Mobile: 375px
- Mobile large: 425px
- Tablet: 768px
- Desktop: 1024px
- Desktop large: 1440px
- 4K: 2560px
```

### Design Tokens (Recomendado)

Usar sistema de tokens para manter consistência entre serviços:

- Spacing scale (4px base)
- Typography scale (modular)
- Color palette (com variantes)
- Border radius (consistente)
- Shadow depths (elevation)
- Animation timings (easing functions)
