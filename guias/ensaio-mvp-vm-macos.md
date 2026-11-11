# Ensaio do MVP em VM no macOS

> Plano criado em 31/10/2026 para validar o gate do MVP sem um Raspberry Pi 5. Resultados complementares registrados em 11/11/2026.

## Objetivo e limites

Este ensaio usa uma VM Linux limpa para verificar instalação, configuração,
backup, restauração, atualização e rollback. Uma segunda configuração, com
menos recursos, serve como teste conservador de desempenho.

Uma VM não reproduz o processador, o controlador de disco, a rede ou o
comportamento térmico de um Raspberry Pi 5. Portanto:

- Os ensaios de instalação e operação podem encontrar e fechar problemas reais.
- Os benchmarks em VM e em outro hardware servem como referência, não como
  certificação do item "Benchmark no Pi 5" do [roadmap](../ROADMAP.md).
- Um ensaio conduzido por uma pessoa técnica não substitui o piloto com pessoas
  não técnicas exigido pelo gate do MVP.

## 1. Registrar o ambiente

Preencher antes de começar:

| Campo | Valor |
| --- | --- |
| Data | 31/10/2026 |
| Modelo e chip do Mac | macbok pro m4 16gb |
| Versão do macOS | 27.0.1 |
| Versão do UTM | 5.0.8 |
| Sistema e versão da VM | debian 13 trixie |
| Arquitetura da VM | arm64 |
| Versão da CLI `taina` | 0.1.0 |
| Versão inicial do Tainá | 0.6.0 |
| Versão de destino do update | 0.6.2 |

## 2. Criar a VM de referência

Usar o UTM em modo de virtualização, não emulação. Não usar emulação de
Raspberry Pi para medir tempo.

- Em Mac Apple Silicon, instalar uma imagem ARM64 de Debian ou Ubuntu.
- Em Mac Intel, instalar uma imagem amd64 de Debian ou Ubuntu.
- Usar o sistema suportado pelo [guia de instalação](instalacao.md).
- Configurar rede pelo modo padrão do UTM e confirmar que o Mac consegue abrir
  a interface da VM no navegador.

Configurações do ensaio:

| Configuração | vCPUs | RAM total | Disco | Swap |
| --- | ---: | ---: | ---: | --- |
| Referência | 2 | 4 GB | 15 GB | 2 GB |
| Restrita | 1 | 2 GB | 15 GB | Desligado |

A configuração restrita limita a VM inteira, incluindo sistema operacional,
Docker e Tainá. Ela é mais conservadora em memória e concorrência, mas não
necessariamente mais lenta que um Pi 5 em todos os aspectos.

Checklist:

- [x] Instalar o sistema Linux.
- [x] Confirmar acesso à internet e resolver nomes por DNS.
- [x] Confirmar que o Mac acessa a VM pela rede.
- [x] Anotar o endereço IP da VM.
- [x] Criar um snapshot chamado `sistema-limpo`, antes de instalar o Docker.

## 3. Ensaio de instalação limpa e cronometrada

Restaurar o snapshot `sistema-limpo`. Seguir somente os passos 1 a 6 do
[guia de instalação](instalacao.md), sem usar o `docker-compose.yml` de
desenvolvimento e sem conhecimento fora do guia.

Usar a imagem publicada, que é o caminho normal de instalação. Se for necessário
compilar a imagem a partir do código-fonte, registrar isso como desvio do
roteiro e medir a compilação separadamente.

Cronometrar cada etapa:

| Etapa | Duração | Onde travou ou exigiu interpretação |
| --- | ---: | --- |
| Instalar e verificar o Docker | 3'47" | instalar docker rootless (docker prove sh helper) |
| Instalar e verificar a CLI | 2'13" | - |
| Criar a comunidade e a pessoa zeladora | 45" | - |
| Criar e aceitar o convite do segundo morador | 2'13" | - |
| **Total, incluindo Docker e disponibilização em rede privada** | Pouco menos de 20 minutos | Sem TLS ou Tailscale neste cenário |

Critérios:

- [x] A instalação termina sem intervenção fora do guia.
- [x] `taina health` responde `saudavel`.
- [x] O assistente de primeiro boot é concluído.
- [x] O segundo morador consegue criar conta e entrar.
- [x] O tempo total fica abaixo de 30 minutos.
- [x] Depois de reiniciar a VM, a instância volta saudável e os dados continuam.
- [x] Todos os pontos de confusão foram anotados, mesmo quando foi possível
      continuar.

## 4. Ensaio de backup e restauração

Executar somente na VM descartável. A restauração é destrutiva.

Antes do backup, criar uma conta de morador, pastas, alguns arquivos e fotos
conhecidos. Depois:

- [x] Preparar o destino de backup conforme o
      [runbook de backup e restauração](runbook-backup-restore.md).
- [x] Executar `taina backup --out /mnt/backup`.
- [x] Guardar o nome completo do arquivo gerado.
- [x] Depois do backup, enviar um arquivo marcador chamado
      `depois-do-backup.txt`.
- [x] Executar `taina restore /mnt/backup/<arquivo-do-backup>`.
- [x] Confirmar `taina status` e `taina health`.
- [x] Entrar com as duas contas e conferir que usuários, pastas, arquivos e
      fotos anteriores ao backup voltaram.
- [x] Conferir que `depois-do-backup.txt` não está presente, provando que o
      estado voltou ao momento do backup.
- [x] Gerar um novo backup do estado restaurado.
- [ ] Registrar duração, erros e qualquer intervenção não documentada.

**Resultado:** backup e restauração concluídos conforme este roteiro, sem falhas
funcionais relatadas. Duração e métricas não foram registradas. O ensaio foi
conduzido por pessoa técnica e não fecha o drill com pessoa não técnica exigido
pelo 1.0.0.

## 5. Ensaio de update e rollback

Para testar uma atualização de verdade, partir de uma versão anterior publicada.
Se a VM já estiver na versão mais recente, `taina update` sozinho prova apenas o
comportamento sem mudança de versão, não o caminho completo de atualização.

- [ ] Registrar as tags inicial e de destino.
- [ ] Instalar a versão anterior usando o mecanismo `TAINA_IMAGE` documentado
      pelo Oka, ou restaurar um snapshot que já contenha essa versão.
- [ ] Criar usuários e arquivos antes de atualizar.
- [ ] Executar `taina update --tag <tag-de-destino>`.
- [ ] Confirmar que a nova versão está ativa e `taina health` responde
      `saudavel`.
- [ ] Conferir que usuários e arquivos continuam acessíveis.
- [ ] Guardar a cópia do banco criada em `rollback/`.
- [ ] Exercitar o rollback usando o procedimento efetivamente fornecido pela
      CLI, sem improvisar um downgrade do banco.
- [ ] Confirmar a versão anterior, a saúde do serviço e a integridade dos dados.
- [ ] Se a CLI não fornecer um procedimento executável de rollback, registrar
      esse item como pendência de operação ou documentação.

## 6. Benchmark na VM de referência

No diretório do Compose, manter `docker stats "$(docker compose ps -q taina)"`
aberto durante os testes e anotar os picos. Acessar a interface pelo navegador
do Mac, não por um navegador dentro da VM.

Preparar os seguintes dados:

- Uma foto real de aproximadamente 4 MB.
- Um vídeo 1080p reproduzível pelo navegador.
- 10 mil arquivos distribuídos em pastas, com paginação.
- 5 mil fotos para a galeria.

Executar:

- [ ] Medir o envio da foto de 4 MB até a thumbnail ficar disponível.
- [ ] Repetir o upload e a geração de thumbnail pelo menos três vezes.
- [ ] Reproduzir o vídeo 1080p e registrar pausas ou atrasos de buffer.
- [ ] Navegar pelas páginas dos 10 mil arquivos e registrar atrasos perceptíveis.
- [ ] Navegar pela galeria com 5 mil fotos, incluindo rolagem e visualização em
      tela cheia.
- [ ] Fazer upload enquanto navega pela interface.
- [ ] Fazer um backup enquanto a instância está em uso.
- [ ] Confirmar `taina health` depois de cada cenário.

Resultados:

| Medição | Resultado | Pico de RAM | Observações |
| --- | ---: | ---: | --- |
| Foto de 4 MB até thumbnail, tentativa 1 | | | |
| Foto de 4 MB até thumbnail, tentativa 2 | | | |
| Foto de 4 MB até thumbnail, tentativa 3 | | | |
| Streaming 1080p | | | |
| Navegação com 10 mil arquivos | | | |
| Galeria com 5 mil fotos | | | |
| Backup com instância em uso | | | |

## 7. Repetir o benchmark na configuração restrita

- [ ] Parar a VM e criar um snapshot ou clone com os dados de teste já prontos.
- [ ] Alterar a VM para 1 vCPU e 2 GB de RAM, como na tabela de configurações.
- [ ] Desligar o swap na VM com `sudo swapoff -a` e confirmar com `free -h`.
- [ ] Repetir todos os cenários e preencher uma segunda tabela de resultados.
- [ ] Registrar erros, timeouts, jobs acumulados e picos de memória.
- [ ] Não tratar aprovação nessa VM como aprovação no Pi 5.

Se um cenário falhar aqui, investigar como possível gargalo real. Se todos
passarem, registrar o resultado como evidência conservadora de funcionamento
nesses limites, sem inferir um tempo específico para o Pi 5.

**Resultado qualitativo registrado em 11/11/2026:** a VM restrita em vCPUs,
memória e swap manteve bom desempenho, inclusive nos picos de uso observados.
Não foram registradas medições suficientes para preencher a tabela de benchmark
ou certificar os limites quantitativos do roadmap. O Raspberry Pi 5 ainda não
foi comprado e continua sendo uma validação separada.

## 8. Complemento opcional no homelab

O homelab Intel i3 com 16 GB de RAM e NVMe pode fornecer uma segunda referência
em hardware real. Não usar a instância pessoal para os testes destrutivos de
restauração ou rollback.

- [ ] Usar uma VM separada ou um projeto Docker separado, com volumes e portas
      próprios.
- [ ] Repetir upload, thumbnail, backup e navegação com `docker stats` aberto.
- [ ] Registrar o resultado como benchmark em Intel i3 com NVMe.
- [ ] Não alterar nem restaurar os volumes da instância pessoal.

## 9. Fechamento do ensaio

Ao terminar, registrar separadamente:

- [x] Instalação técnica em VM: aprovada; pouco menos de 20 minutos incluindo
      Docker e disponibilização em rede privada, sem TLS/Tailscale.
- [x] Backup e restauração: aprovados conforme o roteiro, por pessoa técnica.
- [ ] Update e rollback: pendente.
- [ ] Benchmark formal de referência em VM: resultados quantitativos pendentes.
- [x] Avaliação qualitativa em VM restrita: bom desempenho também nos picos
      observados, sem certificação dos limites quantitativos.
- [ ] Piloto com pessoa não técnica: pendente.
- [ ] Benchmark no Raspberry Pi 5: pendente; hardware ainda não comprado.

Para fechar o item literal do roadmap sem comprar um Pi 5, preparar os dados e
este roteiro com antecedência e fazer somente o benchmark final num Pi 5
emprestado. Os ensaios de instalação, restore e update não precisam ser refeitos
nesse aparelho, salvo se algum problema específico de ARM aparecer.
