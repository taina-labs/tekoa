# Arquitetura atual

**Revisado em:** 2026-11-11.

Este documento guarda as decisões que orientam o código e o corte técnico do `1.0.0`. Detalhes de endpoints e formas de dados vivem no código e no OpenAPI. As RFCs em [arquivo](arquivo/INDICE.md) preservam o histórico, não são contrato atual.

## Produto e fronteiras

- **Uma comunidade por caixa.** Sem troca de comunidade ou feed entre instâncias.
- **Casa e praça.** Novos conteúdos nascem na `casa`. A dona e concessões explícitas leem a casa; toda moradora lê a `praça`. Publicar é explícito e reversível. Zona de pasta não cascateia.
- **Zeladoria sem atalho.** `zelador` cuida da máquina; `morador` usa o cofre. Pode haver vários zeladores, mas nenhum ganha leitura automática da casa alheia.
- **Identidade sem e-mail.** Login por `username`, `display_name` opcional, convites por link/QR e recuperação por token ou CLI de operador.
- **Transparência local.** Mural, saúde, logs e telemetria ficam na caixa. Nenhum dado vai a SaaS de observabilidade.
- **Host confiável.** A casa local é protegida por software e confiança, não por criptografia. Quem tem root lê os dados. E2E não está no escopo.
- **Vocabulário.** Tupi-Guarani para nomes internos de subsistemas; pt-BR para conceitos sociais no banco, código e interface; inglês para encanamento invisível.

## Código e clientes

- Monolito modular Elixir 1.20/OTP 28, Phoenix/LiveView, SQLite/WAL, Oban e libvips.
- Ypy é uma fronteira interna em `lib/ypy`, com núcleo puro e efeitos nas bordas. A antiga dependência externa do Hex foi revertida.
- Não há PostgreSQL nem RLS. Isolamento e autorização são da aplicação e têm testes próprios.
- A API v1 usa token Bearer por dispositivo, revogável, sem cookie e sem tempo real. O OpenAPI no repositório `taina` é o contrato.
- O app Okena é Flutter para iOS e Android. Regras de sincronização ficam em núcleo testável; Photos, background, disco e rede ficam nos adaptadores de plataforma.
- O backup local `.tar.gz` continua existindo. O cofre externo é um destino adicional, não uma substituição silenciosa.

## Espelho do rolo no 1.0.0

### Comportamento

- Seleção manual da galeria e captura pela câmera usam a mesma fila de upload e servem de base para o espelho.
- Envio unidirecional do aparelho para uma pasta "Rolo de câmera" na `casa`.
- Fotos e vídeos; edição gera novo conteúdo. Live Photos podem entrar inicialmente só como foto, se isso estiver visível para a pessoa.
- Nunca publica na `praça`, nunca baixa a casa de volta e nunca propaga exclusões.
- Wi-Fi por padrão; dados móveis são opt-in. Cota ou rede inadequada pausa a fila com motivo legível.
- A promessa é convergência, não execução contínua. A interface explica que o sistema agenda o trabalho em background.
- Configuração e estado ficam somente no app. Não é necessária uma tela de espelho na web.

### Funcionamento

1. O aparelho mantém um registro local por identificador e data de modificação do asset, hash e estado do upload.
2. O manifesto da caixa informa os hashes já conhecidos, inclusive os que não devem ser reenviados da lixeira.
3. Só assets novos ou modificados são lidos e hasheados. A segunda passada não refaz trabalho sem necessidade.
4. Consulta por hash resolve resposta perdida depois de um upload commitado, evitando duplicata.
5. Uploads já iniciados continuam pelos mecanismos de background de cada plataforma. Relançar o app reconcilia fila e resultado.
6. O envio usa streaming, respeita cota e mantém progresso, retry e cancelamento compreensíveis.

Consulta por hash e presença da lixeira no manifesto são as únicas extensões de API esperadas além do comportamento atual. Nenhuma regra social muda.

### Portão

- Passar em simulador iOS e emulador Android locais.
- Subir asset novo e não repetir hash nem upload na segunda passada.
- Não duplicar depois de resposta perdida, reinício ou retry.
- Não reenviar item excluído ou presente na lixeira.
- Pausar corretamente por Wi-Fi, permissão e cota.
- Gerar miniatura de HEIC no container publicado.

## Backup cifrado off-site no 1.0.0

### Promessa e limite

A caixa cifra antes de enviar. O provedor não recebe conteúdo, nomes, árvore de pastas ou manifesto legível; volume agregado e horários continuam visíveis. Isso protege a cópia externa, não cifra o disco local nem impede root de ler a caixa.

O primeiro corte oferece um destino remoto guiado e testado. Outros destinos podem usar o mesmo contrato depois, sem transformar configuração avançada em requisito para todas as comunidades.

### Recuperação com kits equivalentes

- Cada cofre tem segredo aleatório próprio; não depende de senha humana.
- A comunidade recebe kits legíveis/imprimíveis e pode guardar cópias com pessoas diferentes.
- **Todo kit equivalente abre o cofre.** Não há limiar `k-de-n` neste corte.
- Quem tem um kit e acesso ao cofre consegue decifrar os dados. Essa confiança precisa ser explicada na cerimônia.
- Se a caixa e todos os kits forem perdidos, não existe recuperação por suporte, Zeetech ou provedor.
- Kits e segredos nunca aparecem em logs, telemetria ou mural.

### Consistência e restore

1. O run impede execuções concorrentes do mesmo cofre e cria snapshot consistente do SQLite.
2. O conjunto de blobs referenciados fica protegido contra purga até o fim do run.
3. Blobs são sincronizados incrementalmente por conteúdo; banco e manifesto são versionados a cada snapshot.
4. O manifesto cifrado e autenticado é a fonte do restore. Ausência ou adulteração de objetos falha fechada.
5. Um ponteiro atual só é publicado depois que todos os objetos necessários estão no destino.
6. O restore baixa, verifica e abre banco e blobs em staging. Nenhum estado vivo é substituído antes dessa validação.
7. O produto oferece um caminho guiado de configuração e um drill "a caixa morreu" para instalação nova.

Rotação de kits, retenção e política de custo precisam ser resolvidos antes de uso prolongado, sem introduzir Shamir ou recuperação por suporte.

### Portão

- Restaurar contas, permissões, arquivos e chaves numa caixa nova usando somente cofre e kit.
- Detectar objeto ausente, adulterado ou manifesto inválido antes de tocar o estado vivo.
- Retomar run interrompido sem reenviar objetos válidos.
- Provar um destino remoto real e concluir o drill com pessoa não técnica.

## Decisões que evitam expansão acidental

- Sem E2E, espelho bidirecional, propagação de exclusões ou offline-first.
- Sem Shamir/limiar, escrow de chave ou recuperação por suporte.
- Sem chat, assembleia, streaming ou federação no 1.0.0.
- Quando a assembleia existir, sua experiência pode viver no Guará; propostas, votos e quórum continuam sendo regras de domínio próprias, não mensagens soltas.
