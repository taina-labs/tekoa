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
   recente que você confia (no disco USB, no destino remoto, ou em `BACKUP_DIR`).
   Anote o caminho completo dele. Se você não sabe onde os backups ficam, esse é
   um detalhe que quem configurou o backup deveria ter anotado (o disco USB, o
   caminho de rede ou o serviço de sincronização remota): procure essa pessoa
   antes de seguir, em vez de adivinhar.

2. **Coloque o arquivo onde o Tainá enxerga.** Copie o `.tar.gz` para o diretório
   de backups montado no container (o `BACKUP_DIR`, por padrão `/app/backups`;
   no host, é o caminho do seu `TAINA_BACKUP_DIR`).

3. **Tire um backup de segurança do estado atual** (recomendado), caso precise
   voltar atrás da própria restauração. O app está de pé aqui, então a chamada
   roda direto nele:

   ```sh
   docker compose exec app /app/bin/taina eval 'Taina.Nhaman.Backup.run()'
   ```

   O novo `.tar.gz` aparece no diretório de backups.

4. **Pare o app e o proxy.** A restauração substitui o arquivo do banco, e o
   app o mantém aberto enquanto está de pé: nada pode estar lendo ou escrevendo
   nele na hora da troca. Pare os dois:

   ```sh
   docker compose stop caddy app
   # confirme que pararam:
   docker compose ps
   ```

   Exceção: se você acessa a caixa pela Forma A do
   [guia de acesso remoto](acesso-remoto.md#https-sem-dor-certificado-de-verdade-sem-domínio-próprio)
   (o `tailscale serve --bg 4000`), o Tailscale serve o app direto, sem
   passar pelo Caddy. Desligue o serve também antes de seguir:

   ```sh
   tailscale serve off
   ```

5. **Restaure.** Com o app parado, rode a função de restauração do Tainá num
   container avulso da mesma imagem (ele monta os mesmos volumes, e nada mais
   está usando o banco):

   ```sh
   docker compose run --rm app /app/bin/taina eval 'Taina.Nhaman.Backup.restore("/app/backups/taina-backup-AAAAMMDDTHHMMSSZ.tar.gz")'
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

   Como o app está parado, não há nada reclamando nos logs: a troca acontece
   com o banco quieto. Essa função extrai o arquivo, substitui o arquivo do
   banco pelo snapshot e recoloca a árvore de `storage/`.

   Por que o app fica parado e a restauração roda num container avulso: o banco
   é um arquivo, e o app o mantém aberto enquanto está de pé. Trocar o arquivo
   embaixo de um app no ar perderia escrita e corromperia o banco. Com o app
   parado e um container avulso rodando o `eval`, a troca é segura.

   Caminho manual (só o banco), caso precise restaurar apenas o arquivo do
   banco a partir do `db.dump`. Com o app parado, extraia o `db.dump` e copie-o
   por cima, tudo dentro de um container avulso (funciona tanto com volume
   nomeado quanto com disco montado):

   ```sh
   docker compose run --rm app sh -c 'cd /app/backups && tar -xzf taina-backup-AAAAMMDDTHHMMSSZ.tar.gz db.dump && cp -f db.dump /app/data/taina.db'
   ```

6. **Verifique antes de reabrir.** Confira que os dados voltaram (ver a seção de
   verificação abaixo).

7. **Devolva o acesso.** Suba o app e o proxy de novo:

   ```sh
   docker compose up -d
   ```

   Se você desligou o `tailscale serve` no passo 4, religue:

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

- **No CI:** o job de backup-restore cria um banco SQLite de desenvolvimento e
  roda `mix taina.backup.verify`. Essa tarefa escreve um
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
- [ ] A pessoa para o app e o proxy (`docker compose stop caddy app`) antes de
      restaurar.
- [ ] A pessoa entende o aviso de que a restauração é destrutiva.
- [ ] A pessoa executa o restore e vê o `{:ok, :restored}`.
- [ ] A pessoa sobe o app de novo e confirma `/health` ok.
- [ ] A pessoa confere, na interface, que contas, arquivos e fotos voltaram.
- [ ] O ensaio terminou sem nenhuma intervenção técnica de fora do guia.

Se algum item falhou, conserte o procedimento (ou este runbook) e ensaie de
novo. O critério de lançamento é um ensaio com 100% de sucesso.
