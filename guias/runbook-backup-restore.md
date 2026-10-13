# Runbook de Backup e Restauração do Tainá

> Documento escrito em 17/06/2026, atualizado conforme o deploy evolui.

Backup que não foi testado não é backup. Este runbook tem duas partes: o
procedimento de restauração (o que fazer quando precisar) e o ensaio (o
"drill"), que é uma exigência de lançamento do Tainá (RFC 002, gate da Fase 4):
uma pessoa não-técnica precisa conseguir restaurar seguindo este guia, num
ensaio com 100% de sucesso, antes de a comunidade depender disso para valer.

Leia o runbook inteiro uma vez com calma antes de precisar dele de verdade.

## Índice

1. [O que o backup contém](#o-que-o-backup-contém)
2. [Como os backups são feitos](#como-os-backups-são-feitos)
3. [Aviso: restauração é destrutiva](#aviso-restauração-é-destrutiva)
4. [Procedimento de restauração](#procedimento-de-restauração)
5. [Verificação automática](#verificação-automática)
6. [Checklist do ensaio (drill)](#checklist-do-ensaio-drill)

---

## O que o backup contém

Cada backup é um único arquivo `.tar.gz` com nome no formato
`taina-backup-AAAAMMDDTHHMMSSZ.tar.gz` (data e hora em UTC). Dentro dele há:

- `db.dump`: um snapshot do banco SQLite (gerado pelo `VACUUM INTO`), um
  arquivo de banco completo com todo o banco da comunidade (contas, pastas,
  metadados dos arquivos, permissões).
- `storage/`: a árvore inteira de arquivos do Ybira (os arquivos e fotos de
  verdade que as pessoas enviaram).

Restaurar esse par (banco + storage) recoloca a comunidade exatamente no estado
em que estava quando o backup foi feito.

## Como os backups são feitos

O backup já vem implementado no Tainá (subsistema Nhaman). Você não precisa
escrever nada, só ligar e apontar para um destino seguro.

- **Sob demanda:** a CLI gera um na hora.

  ```sh
  taina backup --out /mnt/backup
  ```

  Sem `--out`, o destino é o `BACKUP_DIR` do Tainá (por padrão `/app/backups`,
  que no host é o volume `taina_backups` ou o caminho do seu `TAINA_BACKUP_DIR`).
- **Agendado:** um job do Oban roda todo dia às 04:00 UTC e chama
  `Taina.Nhaman.Backup.run/1`. Ele só funciona se o backup estiver habilitado.
- **Habilitar:** no `taina.conf`, defina `TAINA_BACKUP_ENABLED=true` e rode
  `taina update`. O destino do arquivo é o `TAINA_BACKUP_DIR`; aponte-o para um
  disco USB ou um caminho sincronizado por rclone (o
  [Passo 8 do guia de instalação](instalacao.md#passo-8-ligar-o-backup) mostra
  o passo a passo, incluindo o `chown` que o disco precisa).

Boa prática: não confie só no disco da própria caixa. Um backup que mora no
mesmo aparelho que ele protege some junto se o aparelho for roubado, queimar ou
o disco falhar. Mantenha cópias fora da caixa (disco USB que você guarda em
outro lugar, ou sincronização para um destino remoto).

## Aviso: restauração é destrutiva

A restauração **substitui o arquivo do banco** pelo snapshot do backup e troca
a árvore de `storage/` inteira. Tudo que estiver no banco atual e não estiver no
backup é perdido.

Por isso, restaure só quando tiver certeza, e de preferência tire um backup do
estado atual antes (mesmo que seja um estado ruim, ele pode conter algo que você
ainda vai querer). Restauração não é "desfazer": é "voltar exatamente para
aquele ponto, descartando o resto".

## Procedimento de restauração

O cenário típico: o disco falhou, a caixa foi trocada, ou uma operação deu
errado e você precisa voltar para o último backup bom.

1. **Localize o arquivo de backup.** Encontre o `taina-backup-*.tar.gz` mais
   recente que você confia (no disco USB, no destino remoto, ou no
   `TAINA_BACKUP_DIR`). Anote o caminho completo dele. Se você não sabe onde os
   backups ficam, esse é um detalhe que quem configurou o backup deveria ter
   anotado (o disco USB, o caminho de rede ou o serviço de sincronização
   remota): procure essa pessoa antes de seguir, em vez de adivinhar.

2. **Restaure.** Um comando só:

   ```sh
   taina restore /mnt/backup/taina-backup-AAAAMMDDTHHMMSSZ.tar.gz
   ```

   O `restore` para o Tainá, roda a restauração num container avulso sobre os
   mesmos dados (o arquivo do banco é substituído, então nada pode estar aberto)
   e sobe o Tainá de novo. Ele imprime o que fez a cada passo.

   O arquivo precisa estar num caminho que a caixa enxerga. Se ele está fora
   (num pendrive que você acabou de plugar, por exemplo), copie-o para o
   `TAINA_BACKUP_DIR` antes.

   Por que o app para: o banco é um arquivo, e o Tainá o mantém aberto
   enquanto está de pé. Trocar o arquivo embaixo de um app no ar perderia
   escrita e corromperia o banco. A CLI cuida disso: para, restaura e sobe.

3. **Confirme a saúde.** A CLI já confere, mas se quiser ver a olho:

   ```sh
   taina status
   taina health
   ```

   `taina health` deve responder `saudavel`. Entre na interface para conferir
   que contas, arquivos e fotos voltaram.

4. **Gere um backup novo** do estado restaurado, para não ficar sem cópia:

   ```sh
   taina backup --out /mnt/backup
   ```

   As migrações de banco rodam sozinhas no boot do release, então um backup um
   pouco mais antigo que o esquema atual é migrado para a frente
   automaticamente.

## Verificação automática

O Tainá tem uma verificação de restauração que roda no CI a cada mudança, então
você não precisa torcer para o backup funcionar: ele é exercitado de verdade.

- **No CI:** o job de backup-restore cria um banco SQLite de desenvolvimento e
  roda `mix taina.backup.verify`. Essa tarefa escreve um marcador único no banco
  e no storage, faz um backup, **destrói** os dados, faz o restore, e confirma
  que o marcador voltou idêntico nos dois lugares. Se o ciclo de backup/restore
  quebrar, o CI fica vermelho.
- **Na sua máquina de desenvolvimento (opcional):** você pode rodar a mesma
  verificação contra um banco descartável para ganhar confiança no seu
  ambiente. Uma atenção: essa tarefa é do Mix, então precisa de uma máquina com
  o código-fonte do Tainá e o Elixir instalados. Ela não roda dentro do
  container do release, que não traz o Mix. Ela é destrutiva, então nunca rode
  contra o banco de produção:

  ```sh
  MIX_ENV=dev mix taina.backup.verify
  ```

## Checklist do ensaio (drill)

O ensaio é o gate da Fase 4. Faça com uma pessoa não-técnica conduzindo, você só
observando, e anote onde ela travar (cada tropeço é uma melhoria de documentação
a fazer).

- [ ] A pessoa consegue localizar o arquivo de backup mais recente sozinha.
- [ ] A pessoa entende o aviso de que a restauração é destrutiva.
- [ ] A pessoa executa `taina restore <arquivo>` e vê o Tainá voltar.
- [ ] A pessoa confirma `taina health` ok.
- [ ] A pessoa confere, na interface, que contas, arquivos e fotos voltaram.
- [ ] O ensaio terminou sem nenhuma intervenção técnica de fora do guia.

Se algum item falhou, conserte o procedimento (ou este runbook) e ensaie de
novo. O critério de lançamento é um ensaio com 100% de sucesso.
