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

- `db.dump`: um dump do PostgreSQL no formato custom (gerado pelo `pg_dump
--format=custom`), com todo o banco da comunidade (contas, pastas, metadados
  dos arquivos, permissões).
- `storage/`: a árvore inteira de arquivos do Ybira (os arquivos e fotos de
  verdade que as pessoas enviaram).

Restaurar esse par (banco + storage) recoloca a comunidade exatamente no estado
em que estava quando o backup foi feito.

## Como os backups são feitos

O backup já vem implementado no Tainá (subsistema Nhaman). Você não precisa
escrever nada, só ligar e apontar para um destino seguro.

- **Agendado:** um job do Oban roda todo dia às 04:00 UTC e chama
  `Taina.Nhaman.Backup.run/1`. Ele só funciona se o backup estiver habilitado.
- **Habilitar:** no `.env`, defina `BACKUP_ENABLED=true`. O destino do arquivo é
  o `BACKUP_DIR` (por padrão `/app/backups` dentro do container, mapeado para um
  volume). Em produção, monte ali um disco USB ou um caminho sincronizado por
  rclone, para o backup não morrer junto com a caixa.
- **Sob demanda:** dentro do release você pode chamar `Taina.Nhaman.Backup.run/0`
  para gerar um backup na hora (por exemplo, antes de uma atualização grande).

Boa prática: não confie só no disco da própria caixa. Um backup que mora no
mesmo aparelho que ele protege some junto se o aparelho for roubado, queimar ou
o disco falhar. Mantenha cópias fora da caixa (disco USB que você guarda em
outro lugar, ou sincronização para um destino remoto).

## Aviso: restauração é destrutiva

A restauração usa `pg_restore --clean --if-exists`, que **apaga e recria** os
objetos do banco antes de recolocar os dados do backup. Tudo que estiver no
banco atual e não estiver no backup é perdido.

Por isso, restaure só quando tiver certeza, e de preferência tire um backup do
estado atual antes (mesmo que seja um estado ruim, ele pode conter algo que você
ainda vai querer). Restauração não é "desfazer": é "voltar exatamente para
aquele ponto, descartando o resto".

## Procedimento de restauração

O cenário típico: o disco falhou, a caixa foi trocada, ou uma operação deu
errado e você precisa voltar para o último backup bom.

1. **Localize o arquivo de backup.** Encontre o `taina-backup-*.tar.gz` mais
   recente que você confia (no disco USB, no destino remoto, ou em `BACKUP_DIR`).
   Anote o caminho completo dele.

2. **Coloque o arquivo onde o Tainá enxerga.** Copie o `.tar.gz` para o diretório
   de backups montado no container (o `BACKUP_DIR`, por padrão o volume mapeado
   em `/app/backups`).

3. **Pare a aplicação, mantenha o banco no ar.** Você quer ninguém escrevendo no
   banco durante a restauração, mas o Postgres precisa estar de pé para receber
   os dados:

   ```sh
   docker compose stop app
   # confira que o banco continua rodando:
   docker compose ps db
   ```

4. **Tire um backup de segurança do estado atual** (recomendado), caso precise
   voltar atrás da própria restauração.

5. **Restaure.** Há dois caminhos. O recomendado usa a própria função do Tainá,
   que cuida do banco e do storage de uma vez. Suba um shell do release:

   ```sh
   docker compose run --rm app /app/bin/taina remote
   ```

   E no prompt do Elixir:

   ```elixir
   Taina.Nhaman.Backup.restore("/app/backups/taina-backup-AAAAMMDDTHHMMSSZ.tar.gz")
   ```

   Espere o retorno `{:ok, :restored}`. Essa função extrai o arquivo, roda o
   `pg_restore --clean --if-exists` no banco e recoloca a árvore de `storage/`.

   Caminho manual (só o banco), caso precise restaurar apenas o Postgres a partir
   do `db.dump` extraído:

   ```sh
   pg_restore --clean --if-exists --no-owner --no-privileges \
     --dbname "$DATABASE_URL" db.dump
   ```

6. **Verifique antes de reabrir.** Confira que os dados voltaram (ver a seção de
   verificação abaixo).

7. **Suba a aplicação de novo.**

   ```sh
   docker compose start app
   ```

   As migrações de banco rodam sozinhas no boot do release, então um backup um
   pouco mais antigo que o esquema atual é migrado para a frente
   automaticamente.

8. **Confirme a saúde.** Acesse `/health` (deve responder status ok) e entre na
   interface para conferir que contas, arquivos e fotos estão lá.

## Verificação automática

O Tainá tem uma verificação de restauração que roda no CI a cada mudança, então
você não precisa torcer para o backup funcionar: ele é exercitado de verdade.

- **No CI:** o job de backup-restore instala o cliente `postgresql-client-18`,
  cria um banco, e roda `mix taina.backup.verify`. Essa tarefa escreve um
  marcador único no banco e no storage, faz um backup, **destrói** os dados, faz
  o restore, e confirma que o marcador voltou idêntico nos dois lugares. Se o
  ciclo de backup/restore quebrar, o CI fica vermelho.
- **Na sua caixa (opcional):** você pode rodar a mesma verificação contra um
  banco descartável para ganhar confiança no seu ambiente. Ela é destrutiva,
  então nunca rode contra o banco de produção:

  ```sh
  MIX_ENV=dev mix taina.backup.verify
  ```

## Checklist do ensaio (drill)

O ensaio é o gate da Fase 4. Faça com uma pessoa não-técnica conduzindo, você só
observando, e anote onde ela travar (cada tropeço é uma melhoria de documentação
a fazer).

- [ ] A pessoa consegue localizar o arquivo de backup mais recente sozinha.
- [ ] A pessoa copia o arquivo para o diretório de backups do Tainá.
- [ ] A pessoa para o app mantendo o banco no ar (`docker compose stop app`).
- [ ] A pessoa entende o aviso de que a restauração é destrutiva.
- [ ] A pessoa executa o restore e vê o `{:ok, :restored}`.
- [ ] A pessoa sobe o app de novo e confirma `/health` ok.
- [ ] A pessoa confere, na interface, que contas, arquivos e fotos voltaram.
- [ ] O ensaio terminou sem nenhuma intervenção técnica de fora do guia.

Se algum item falhou, conserte o procedimento (ou este runbook) e ensaie de
novo. O critério de lançamento é um ensaio com 100% de sucesso.
