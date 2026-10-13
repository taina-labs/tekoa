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
  o `BACKUP_DIR` (`/app/backups` dentro do container). No host, esse caminho é
  o que você definir no `TAINA_BACKUP_DIR` do `.env`; sem ele, os backups caem
  num volume nomeado do Docker, que fica no mesmo disco da caixa. Para o
  backup não morrer junto com a caixa, aponte o `TAINA_BACKUP_DIR` para um
  disco USB ou um caminho sincronizado por rclone (o
  [Passo 8 do guia de instalação](instalacao.md#passo-8-ligar-o-backup)
  mostra o passo a passo, incluindo o `chown` que o disco precisa).
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
   Anote o caminho completo dele. Se você não sabe onde os backups ficam, esse é
   um detalhe que quem configurou o backup deveria ter anotado (o disco USB, o
   caminho de rede ou o serviço de sincronização remota): procure essa pessoa
   antes de seguir, em vez de adivinhar.

2. **Coloque o arquivo onde o Tainá enxerga.** Copie o `.tar.gz` para o diretório
   de backups montado no container (o `BACKUP_DIR`, por padrão `/app/backups`;
   no host, é o caminho do seu `TAINA_BACKUP_DIR`).

3. **Tire o mundo de fora, sem derrubar o app.** Durante a restauração você não
   quer ninguém escrevendo no banco. O comando de restauração roda dentro do
   container do app, então o app precisa continuar no ar: não rode `docker
   compose stop app`. A porta de entrada da caixa é o proxy. Pare só ele, que
   o app e o banco seguem de pé:

   ```sh
   docker compose stop caddy
   # o app e o banco precisam continuar de pé:
   docker compose ps
   ```

   Exceção: se você acessa a caixa pela Forma A do
   [guia de acesso remoto](acesso-remoto.md#https-sem-dor-certificado-de-verdade-sem-domínio-próprio)
   (o `tailscale serve --bg 4000`), o Tailscale serve o app direto, sem
   passar pelo Caddy. Desligue o serve também antes de seguir:

   ```sh
   tailscale serve off
   ```

4. **Tire um backup de segurança do estado atual** (recomendado), caso precise
   voltar atrás da própria restauração:

   ```sh
   docker compose exec app /app/bin/taina eval 'Taina.Nhaman.Backup.run()'
   ```

   O novo `.tar.gz` aparece no diretório de backups.

5. **Restaure.** Há dois caminhos. O recomendado usa a própria função do Tainá,
   que cuida do banco e do storage de uma vez, e roda dentro do container do
   app:

   ```sh
   docker compose exec app /app/bin/taina eval 'Taina.Nhaman.Backup.restore("/app/backups/taina-backup-AAAAMMDDTHHMMSSZ.tar.gz")'
   ```

   O `eval` roda a chamada e imprime o resultado. Se você vir exatamente
   `{:ok, :restored}`, a restauração deu certo. Qualquer outra resposta (uma
   mensagem de erro, um `{:error, ...}`) significa que falhou: não siga para o
   próximo passo, leia o erro e resolva a causa (ou peça ajuda) antes de
   continuar.

   Uma nota sobre por que `eval` e não o `remote` do release: o `remote` serve
   para se conectar a um node do Tainá que já esteja no ar. Num container
   avulso, ou com o app parado, não existe node nenhum para conectar e o
   comando falha. O `eval` roda a chamada na hora, sem depender disso.

   Enquanto o restore roda, o app pode reclamar nos logs: as tabelas estão
   sendo trocadas embaixo dele. É esperado, e ele se recompõe quando o
   restore termina. Essa função extrai o arquivo, roda o `pg_restore --clean
   --if-exists` no banco e recoloca a árvore de `storage/`. O dono das
   tabelas volta do jeito que está gravado no dump, que é o mesmo de antes
   do desastre.

   Caminho manual (só o banco), caso precise restaurar apenas o Postgres a
   partir do `db.dump`. Primeiro extraia só o `db.dump` de dentro do arquivo
   de backup. Faça de dentro do container, para funcionar tanto com volume
   nomeado quanto com disco montado (o container traz o `tar` e o cliente do
   PostgreSQL):

   ```sh
   docker compose exec app sh -c 'cd /app/backups && tar -xzf taina-backup-AAAAMMDDTHHMMSSZ.tar.gz db.dump'
   ```

   Depois rode o restore a partir do `db.dump` extraído. Note a conexão: o
   restore manual precisa do usuário administrador do banco (o
   `BACKUP_DATABASE_URL`, que já existe dentro do container), e não do
   `DATABASE_URL` que o app usa. A variável existe dentro do container, e
   não na sua shell de host, então deixe o `sh -c` expandi-la lá dentro (o
   mesmo jeito do passo de extração acima):

   ```sh
   docker compose exec app sh -c 'pg_restore --clean --if-exists --no-privileges --dbname "$BACKUP_DATABASE_URL" /app/backups/db.dump'
   ```

   Um detalhe: não acrescente `--no-owner` a esse comando. O dump gravado
   pelo backup guarda o dono dos objetos, e é ele que repõe a propriedade.
   Com `--no-owner`, tudo nasce dono do usuário da conexão de backup, o
   `taina_app` deixa de ser dono do próprio banco, e o sintoma é estranho: o
   app sobe, mas nada aparece, porque ele não é dono das tabelas que
   precisa ler.

6. **Verifique antes de reabrir.** Confira que os dados voltaram (ver a seção de
   verificação abaixo).

7. **Devolva o acesso.** O app nunca chegou a parar; o que estava parado era o
   proxy:

   ```sh
   docker compose start caddy
   ```

   Se você desligou o `tailscale serve` no passo 3, religue:

   ```sh
   tailscale serve --bg 4000
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
- **Na sua máquina de desenvolvimento (opcional):** você pode rodar a mesma
  verificação contra um banco descartável para ganhar confiança no seu
  ambiente. Uma atenção: essa tarefa é do Mix, então precisa de uma máquina
  com o código-fonte do Tainá e o Elixir instalados. Ela não roda dentro do
  container do release, que não traz o Mix. Ela é destrutiva, então nunca
  rode contra o banco de produção:

  ```sh
  MIX_ENV=dev mix taina.backup.verify
  ```

## Checklist do ensaio (drill)

O ensaio é o gate da Fase 4. Faça com uma pessoa não-técnica conduzindo, você só
observando, e anote onde ela travar (cada tropeço é uma melhoria de documentação
a fazer).

- [ ] A pessoa consegue localizar o arquivo de backup mais recente sozinha.
- [ ] A pessoa copia o arquivo para o diretório de backups do Tainá.
- [ ] A pessoa bloqueia o acesso de fora sem derrubar o app
      (`docker compose stop caddy`).
- [ ] A pessoa entende o aviso de que a restauração é destrutiva.
- [ ] A pessoa executa o restore e vê o `{:ok, :restored}`.
- [ ] A pessoa sobe o app de novo e confirma `/health` ok.
- [ ] A pessoa confere, na interface, que contas, arquivos e fotos voltaram.
- [ ] O ensaio terminou sem nenhuma intervenção técnica de fora do guia.

Se algum item falhou, conserte o procedimento (ou este runbook) e ensaie de
novo. O critério de lançamento é um ensaio com 100% de sucesso.
