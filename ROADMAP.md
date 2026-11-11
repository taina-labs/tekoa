# Roadmap do Tainá

**Atualizado em:** 2026-11-11.

O próximo marco é o `1.0.0`: uma alternativa real, usável e confiável para uma comunidade começar a substituir nuvens pagas de arquivos e fotos. O recorte é **capturar, guardar e recuperar**, não reproduzir todos os serviços de uma big tech.

## 1. O que já existe

Verificado no código de `taina` em 2026-11-10:

- Caixa com uma comunidade por instância, SQLite, interface web LiveView e regras de isolamento na aplicação.
- Contas, convites, recuperação de senha, papéis, dispositivos, pedidos e concessões de acesso.
- Arquivos e pastas em `casa` ou `praça`, upload, download, preview, cotas, lixeira e restauração.
- Galeria com miniaturas, linha do tempo e visualizador; mural e saúde da comunidade.
- API JSON v1 com OpenAPI, tokens de dispositivo, manifesto de hashes, operações em lote e ETag.
- App Flutter para iOS e Android, cobrindo as telas e os fluxos principais da API.
- Backup local `.tar.gz`, manual ou agendado, com restore verificado em CI.
- Imagens multi-arch, Compose/NixOS e instalação/atualização pelo Oka.

A [revisão 0.1.0](tecnico/arquivo/REVIEW_0.1.0.md) é histórica. C1, C2, C4 e os principais achados altos já foram corrigidos.

## 2. O que entra no 1.0.0

### Fechar o produto

- [ ] Tornar zona, filtro e publicação legíveis na galeria web e depois do upload.
- [ ] Adicionar seleção manual da galeria e captura pela câmera no app.
- [ ] Implementar o **espelho unidirecional do rolo** no app, para iOS e Android, sempre para a `casa`.
- [ ] Implementar **backup cifrado off-site**, incremental e restaurável, sem substituir o backup local.
- [ ] Usar kits de recuperação aleatórios e equivalentes. Sem Shamir ou limiar `k-de-n` neste corte.
- [ ] Provar JPEG e HEIC com geração real de miniatura no container publicado.

As regras técnicas desses dois recursos estão em [Arquitetura atual](tecnico/ARQUITETURA.md). E2E, espelho bidirecional e propagação de exclusões não entram.

### Provar em campo e nos clientes

| Prova | Situação registrada | O que ainda falta |
| --- | --- | --- |
| Instalação | Passou em VM por pessoa técnica. Docker, instalação e disponibilização em rede privada, sem TLS/Tailscale, levaram pouco menos de 20 minutos. | Observar pessoas não técnicas seguindo só o guia. |
| Recursos limitados | A VM restrita em vCPUs, memória e swap funcionou bem, inclusive em picos de uso. É evidência qualitativa, não benchmark formal. | Medir os cenários e manter os resultados no ensaio. |
| Backup local | Backup e restore passaram conforme o guia de ensaio. | Drill por pessoa não técnica. |
| Update e rollback | Não verificado. | Ensaiar atualização, integridade dos dados e volta à versão anterior. |
| App | Existem testes de Dart/Flutter. | Validar os fluxos do 1.0.0 em simulador iOS e emulador Android locais. |
| Backup externo | Não implementado. | Restaurar uma caixa nova somente com cofre e kit, depois repetir com pessoa não técnica. |
| Raspberry Pi 5 | Não comprado; nenhum resultado foi medido nesse hardware. | Foto de 4 MB com miniatura em menos de 3 s, vídeo 1080p, 10 mil arquivos e 5 mil fotos. |
| Pilotos | Não verificado. | Duas ou três comunidades; pelo menos uma usando semanalmente. |

Os resultados ficam no [ensaio de MVP](guias/ensaio-mvp-vm-macos.md). "Funcionou bem" não vira número inventado, e VM não vira certificação de Pi 5.

### Fora do 1.0.0

Busca, compartilhamento por link, backup pessoal portátil, upload resumível, universal links, offline-first, chat, assembleia, streaming, federação, Conecta e hospedagem gerenciada.

## 3. Depois do 1.0.0

Ordem de direção, sem datas ou promessa automática:

1. Busca e formas de compartilhamento orientadas pelos pilotos.
2. Backup pessoal portátil da `casa`.
3. Universal links, upload resumível, cache e background conforme necessidade real.
4. Imagem de appliance NixOS e kits de hardware.
5. Conecta como conveniência opcional de relay e destino externo.
6. Guará, com a assembleia apresentada no fluxo do chat mas mantendo propostas, votos e quórum como regras próprias.
7. Streaming e demais contextos somente quando houver comunidade e capacidade de manutenção.

## 4. Ideias, sem compromisso

- Federação da `praça` entre comunidades parceiras.
- Cópia de backup entre duas Tekoas.
- Live Photos, álbuns do telefone como pastas e substituição de fotos editadas.
- Exportação das próprias contribuições da `praça`.
- Workspace para universidades como produto separado, somente com demanda institucional real.
- Araci, Arapa, Ka'a e Karai como visão de longo prazo.

E2E não faz parte do escopo atual nem da fila do 1.0.0. O host da comunidade é confiável; a cifra do 1.0.0 protege o backup contra o provedor externo, não esconde os dados de root na caixa.
