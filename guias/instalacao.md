# Guia de Instalação do Tainá

> Documento escrito em 05/10/2026, atualizado conforme o deploy evolui.

Este é o guia para colocar o Tainá no ar. Ele assume que você **não é
técnico**: nunca abriu um terminal, ou abriu mas nunca instalou nada. Se algo
aqui não couber na sua situação, o [guia de acesso remoto](acesso-remoto.md)
explica outras formas de chegar na caixa, e o
[runbook de backup](runbook-backup-restore.md) explica como não perder nada.

O critério do projeto é simples: seguir este guia do começo ao fim, sem ajuda
de ninguém, em menos de 30 minutos.

Se você só quer experimentar numa caixa local, pare depois do
[Passo 5](#passo-5-criar-a-comunidade). As partes de domínio público, acesso
remoto e disco de backup são para quando a comunidade for usar de verdade.

## Índice

1. [O que você vai precisar](#o-que-você-vai-precisar)
2. [Passo 1: instalar o Docker](#passo-1-instalar-o-docker)
3. [Passo 2: baixar o Oka](#passo-2-baixar-o-oka)
4. [Passo 3: montar a imagem do Tainá](#passo-3-montar-a-imagem-do-tainá)
5. [Passo 4: instalar](#passo-4-instalar)
6. [Passo 5: criar a comunidade](#passo-5-criar-a-comunidade)
7. [Passo 6: convidar a primeira pessoa](#passo-6-convidar-a-primeira-pessoa)
8. [Passo 7: acessar de fora da sua rede](#passo-7-acessar-de-fora-da-sua-rede)
9. [Passo 8: ligar o backup](#passo-8-ligar-o-backup)
10. [Atualizar o Tainá](#atualizar-o-tainá)
11. [Quando algo dá errado](#quando-algo-dá-errado)
12. [Quanto tempo leva](#quanto-tempo-leva)

## O que você vai precisar

- Uma máquina ligada 24 horas por dia. Um Raspberry Pi 5 serve, um mini PC
  serve, um servidor de VPS serve. O [guia de
  hardware](compras-hardware.md) lista opções com preço.
- Um terminal. No Raspberry Pi OS e no Linux ele já vem instalado. No Windows,
  instale o [Windows Terminal](https://aka.ms/terminal) ou use o
  PowerShell. No macOS, o aplicativo Terminal já existe.
- Docker, com o plugin Compose. É o [Passo 1](#passo-1-instalar-o-docker).
- `openssl`, que o `install.sh` usa para sortear o segredo da sessão. No Raspberry Pi OS
  e no Linux ele costuma já estar instalado (`sudo apt-get install -y openssl`
  no Debian e no Ubuntu, caso não esteja). No macOS vem com o sistema. No
  Windows, instalar o [Git for Windows](https://git-scm.com/download/win) traz
  o `openssl` junto.
- `git`, só se você for montar a imagem do código-fonte
  ([Passo 3](#passo-3-montar-a-imagem-do-tainá)). Quem usa a imagem pronta do
  [Passo 4](#passo-4-instalar) não precisa dele.
- Storage de verdade: um pendrive, um disco externo ou uma partição do próprio
  disco que não seja do sistema.

Um cartão SD comum é o pior lugar para guardar os arquivos de uma comunidade.
Ele é lento e as fotos da família o matam. Se a sua caixa é um Raspberry Pi, use
um SSD USB. Isso muda mais a experiência do Tainá do que qualquer configuração
deixada de fora.

Opcional, só se você quiser um endereço público com certificado de verdade: um
domínio apontando para a sua caixa, com as portas 80 e 443 liberadas. Sem
domínio, o Tainá funciona normalmente com um certificado local e o
[Passo 7](#passo-7-acessar-de-fora-da-sua-rede) mostra como acessar de fora
mesmo assim.

## Passo 1: instalar o Docker

O Tainá roda como um conjunto de programas isolados dentro do Docker. Isso
mantém o resto da sua máquina limpo e faz a atualização ser um comando só.

**No Raspberry Pi OS ou num Linux (Debian, Ubuntu)**, abra o terminal, cole este
bloco inteiro e aperte Enter:

```sh
curl -fsSL https://get.docker.com | sh
```

O script pergunta se quer adicionar o Docker ao grupo de administradores.
Responda `yes`. Para isso valer nesta sessão, saia do terminal e abra de novo.

Confira se funcionou:

```sh
docker --version
docker compose version
```

Duas linhas com número de versão, e está pronto.

Se o comando `docker compose` der "não é um comando", o plugin do Compose não
foi instalado. No Debian e no Ubuntu isso costuma ser um passo à parte:

```sh
sudo apt-get update && sudo apt-get install -y docker-compose-plugin
```

**No Windows e no macOS**, instale o Docker Desktop pelo site
[docker.com](https://www.docker.com/products/docker-desktop/) e abra o
aplicativo uma vez antes de continuar. No Windows, abra o Docker Desktop e
confirme que ele está no modo Linux.

## Passo 2: baixar o Oka

O Oka é o repositório que coloca tudo no ar. Ele não contém o Tainá: ele
organiza o banco de dados, a aplicação e o proxy que cuida do HTTPS.

```sh
git clone https://github.com/taina-labs/oka.git
cd oka
```

Sem o `git` instalado? Baixe o ZIP da
[página do Oka](https://github.com/taina-labs/oka) e descompacte. O resto do
guia funciona igual, com uma exceção: se você precisar montar a imagem na sua
própria caixa ([Passo 3](#passo-3-montar-a-imagem-do-tainá)), o `git` é
necessário de verdade, porque o script baixa o código do Tainá de um
repositório.

## Passo 3: montar a imagem do Tainá

**Este passo só é necessário se você quiser partir do código-fonte.** A forma
normal de instalar é o [Passo 4](#passo-4-instalar), que baixa a imagem pronta
do Tainá. Se o `install.sh` reclamar que a imagem não está disponível no
registro, volte para cá e rode:

```sh
./build-image.sh
```

O que isso faz: baixa o código do Tainá para dentro da pasta `oka/taina-src` e
compila a imagem na sua própria máquina. Na primeira vez leva vários minutos,
porque baixa o compilador do Elixir e as dependências todas. Depois disso leva
pouco.

Ao terminar, ele escreve um arquivo `docker-compose.override.yml`. Não se
preocupe com ele: o Docker Compose lê esse arquivo sozinho e passa a usar a
imagem que você acabou de montar. Para voltar ao comportamento normal, apague
esse arquivo.

Se você montou a imagem local, as atualizações também passam a ser locais:
rode `./build-image.sh` de novo e depois `./update.sh`. Está descrito em
[Atualizar o Tainá](#atualizar-o-tainá).

## Passo 4: instalar

```sh
./install.sh
```

O que acontece, em ordem:

1. O script confere se o Docker está presente.
2. Ele cria um arquivo `.env` com um segredo sorteado, que assina as sessões.
   Guarde esse arquivo com cuidado. Quem tem ele tem acesso à caixa.
3. Ele baixa a imagem do Tainá e sobe os dois serviços: a aplicação (com o
   banco da comunidade, um arquivo SQLite que o app mesmo cuida) e o proxy
   reverso.
4. Ele espera o aplicativo responder que está saudável, o que prova que o app
   subiu e que o banco está acessível. Isso leva cerca de um minuto na primeira
   vez.
5. Ele imprime o endereço para o passo seguinte.

Rode o script mais de uma vez sem problema: um `.env` que já existe nunca é
sobrescrito, então seus segredos e seus dados continuam onde estão.

Se aparecer `ERRO: imagem do Tainá indisponível`, volte ao
[Passo 3](#passo-3-montar-a-imagem-do-tainá).

### Usando um domínio público

Se você tem um domínio apontando para a sua caixa e as portas 80 e 443
abertas, rode com o domínio:

```sh
TAINA_DOMAIN=nuvem.suacomunidade.org ./install.sh
```

Isso faz o Caddy pedir um certificado real e gratuito (Let's Encrypt) e
redirecionar HTTP para HTTPS automaticamente.

Sem domínio, o Caddy usa um certificado local. Funciona, mas o navegador vai
avisar que a conexão não é segura na primeira vez, em cada dispositivo. Isso é
esperado. Para não ter aviso nenhum sem gastar com domínio, veja o
[Passo 7](#passo-7-acessar-de-fora-da-sua-rede).

O `.env` guarda o domínio. Para mudar depois, edite a linha `TAINA_DOMAIN=` no
arquivo `.env` e rode `./update.sh`.

## Passo 5: criar a comunidade

Abra no navegador o endereço que o `install.sh` imprimiu no final. Ele termina
em `/setup`. É algo como `https://nuvem.suacomunidade.org/setup`.

Vão aparecer três telas, em sequência:

1. **Nome da comunidade.** O nome que a sua comunidade vai chamar. Pode ser
   "Vila Esperanca", "Coletivo do Beco", "Casa da Rua 12", qualquer coisa. Cada
   instância do Tainá é uma comunidade só, e o sistema recusa criar uma
   segunda: o nome é definitivo para sempre.
2. **Quem cuida da máquina.** Um nome de usuário, um nome de exibição para as
   pessoas verem e uma senha com pelo menos 8 caracteres. Essa pessoa é a
   **zeladora** (ou o zelador). Cuida do disco, dos backups, das atualizações e
   dos convites.
3. **Armazenamento.** O Tainá mostra onde os arquivos vão ficar e quanto espaço
   livre existe. Leia, e siga em frente se estiver de acordo.

Depois do último passo você entra logado. Uma Tekoa foi criada e você é a
primeira pessoa dela.

Uma coisa importante para expectations: essa conta é **zeladora**, não
administradora. A zeladora cuida da máquina e não tem poder de ler o que está
na `casa` dos moradores. Para cada morador(a), a `casa` é privada por padrão e
só se abre com um pedido de acesso aprovado. E, para ser honesto: isso é uma
promessa de software, não um cadeado matemático. O Tainá ainda não criptografa
os arquivos.

## Passo 6: convidar a primeira pessoa

No menu, abra **Membros** e depois **Convidar**.

O Taina mostra um link de convite e um QR code. Funciona dos dois jeitos:

- **Link e QR code.** Para quem está na mesma rede, ou para quem você quer
  receber o convite pessoalmente. A pessoa abre e cria a conta dela.
- **QR code.** Bom para passar o celular para a pessoa. Ela aponta a câmera e
  o convite abre na hora.

O e-mail não existe no Tainá, e isso é uma decisão, não uma falta: a RFC 002
descartou e-mail por completo. Menos uma senha guardada num servidor de
terceiros.

Se a pessoa não conseguir usar o convite (o celular dela não está na sua rede),
veja o [Passo 7](#passo-7-acessar-de-fora-da-sua-rede) e convide de novo
depois.

## Passo 7: acessar de fora da sua rede

Dentro da sua rede local, o endereço da caixa já funciona. De fora, há quatro
caminhos, explicados no [guia de acesso remoto](acesso-remoto.md).

A escolha que a gente recomenda para a maioria das comunidades é o **Tailscale**
(nomeado `Opção 1` lá). Ele dá um endereço seguro à sua caixa, sem abrir porta
nenhuma na internet e sem certificado falso. Leva uns dez minutos:

1. Instale o Tailscale na caixa e no celular ou no computador de quem vai
   acessar.
2. Entre na mesma conta nos dois.
3. Instale no celular de cada morador(a) e faça login com a mesma conta.

Pronto. Cada pessoa passa a acessar o Tainá pelo endereço Tailscale, com HTTPS
de verdade, sem aviso do navegador.

## Passo 8: ligar o backup

O backup vem **desligado**. Uma comunidade de verdade precisa dele ligado, e
precisa de uma cópia em outro lugar, não só no mesmo disco.

O [runbook de backup e restauração](runbook-backup-restore.md) tem o
procedimento inteiro, incluindo como restaurar e como ensaiar. O resumo:

1. **Monte um disco externo** na caixa. O ponto de montagem precisa existir
   quando o Docker sobe. Por exemplo, um disco em `/mnt/backup`.
2. **Dê o disco ao Tainá.** O container roda como um usuário sem privilégios
   (`nobody`, uid 65534), então a pasta precisa pertencer a ele, senão o
   backup falha na hora de escrever:

   ```sh
   sudo chown 65534:65534 /mnt/backup
   ```

3. **Aponte o backup para ele.** No `.env`, descomente a linha do caminho de
   backup e coloque o seu disco:

   ```
   TAINA_BACKUP_DIR=/mnt/backup
   ```

   Sem essa linha, o backup mora num volume nomeado (`backups_data`) dentro do
   Docker: fica no mesmo disco do sistema, e some com o aparelho. Um caminho
   de disco é o seu disco, e você pode trocar, levar embora ou guardar em
   outro cômodo. O mesmo vale para os arquivos de verdade: o
   `TAINA_STORAGE_DIR` aponta a pasta de uploads para outro lugar (veja
   ["Onde ficam meus arquivos"](#onde-ficam-meus-arquivos) mais abaixo).
4. **Ligue o backup.** No `.env`, mude a linha:

   ```
   BACKUP_ENABLED=true
   ```

5. **Suba de novo:**

   ```sh
   ./update.sh
   ```

   A partir daí um backup é feito todo dia às 04:00 UTC.

6. **Gere um agora e ensaie restaurar.** Um backup que nunca foi testado não é
   backup. O runbook tem o passo a passo do ensaio.

Sobre a pasta de backup: uma cópia que mora no mesmo aparelho que ela protege
desaparece junto se o aparelho for roubado, queimar ou o disco falhar. Tenha
pelo menos uma cópia em outro lugar. Um pendrive que você guarda guardado vale
mais do que uma pasta no mesmo disco.

## Atualizar o Tainá

```sh
./update.sh
```

Ele baixa a versão nova, reinicia e confere a saúde. As migrações de banco
rodam sozinhas no boot, então atualizar é seguro e você não precisa rodar
nenhum comando no banco.

Antes de mexer em qualquer coisa, o `update.sh` faz um backup do banco de
dados por conta própria e guarda na pasta `rollback/`, dentro da pasta do
Oka. Se o Tainá não voltar a ficar saudável, ele não te deixa no escuro:
imprime os comandos exatos para voltar para a versão anterior, inclusive como
devolver esse backup.

Fique claro o que essa rede de proteção cobre: ela é só do banco de dados, é
feita na hora e mora na própria caixa. Ela existe para o dia em que uma
atualização sai errada, e não substitui o backup de verdade do
[Passo 8](#passo-8-ligar-o-backup), que é diário, inclui os arquivos e mora
fora da caixa. Os arquivos de storage não são mexidos pela atualização: ficam
no volume ou no disco deles, intocados.

Se você montou a imagem local no [Passo 3](#passo-3-montar-a-imagem-do-tainá),
a atualização é local. Rode `./build-image.sh` antes, ou rode `./update.sh` e
ele já reconstrói para você.

Se algo der errado depois de atualizar, o `update.sh` diz como voltar para a
versão anterior.

## Quando algo dá errado

### `ERRO: imagem do Tainá indisponível`

O `install.sh` não conseguiu baixar a imagem e ela não está na sua máquina.
Rode o [Passo 3](#passo-3-montar-a-imagem-do-tainá) e depois o
[Passo 4](#passo-4-instalar) de novo.

### O navegador avisa que a conexão não é segura

Esperado numa caixa sem domínio: o certificado é local, e o navegador não
confia nele. As páginas funcionam normalmente, com o aviso. Para um HTTPS sem
aviso, use Tailscale ([Passo 7](#passo-7-acessar-de-fora-da-sua-rede)) ou um
domínio no [Passo 4](#usando-um-domínio-público).

### O app não ficou saudável a tempo

Veja os logs:

```sh
docker compose logs -f app
```

A causa mais comum é a primeira migração do banco ainda rodando. Espere um
minuto e rode `./update.sh`, que só reinicia e reconfere a saúde. Se o erro se
repetir, copie a saída dos logs e abra uma issue.

### A caixa rodou o compose antigo (com PostgreSQL)

O Tainá agora guarda tudo num arquivo SQLite; não há mais serviço de banco de
dados. Uma caixa que rodou o compose anterior tem os dados num volume
`postgres_data`, que a versão nova não lê. Sendo alpha, não há dado real a
perder, então recomece:

```sh
docker compose down -v
./install.sh
```

Atenção: o `down -v` apaga os volumes da caixa, incluindo o do storage. Se
há arquivos ali que você queira guardar, copie para fora antes de rodar.

### `docker compose` diz que o comando não existe

O plugin do Compose não está instalado. Veja a seção de instalação do
[Passo 1](#passo-1-instalar-o-docker).

### O convite não abre no celular do morador

O convite é um endereço da sua rede local, e o celular dele está fora. Resolva
o acesso remoto primeiro ([Passo 7](#passo-7-acessar-de-fora-da-sua-rede)) e
convide de novo, agora com o endereço que o guia der.

### O `TAINA_DOMAIN` mudou e o Caddy não pegou o certificado novo

Rode `./update.sh` e acompanhe `docker compose logs -f caddy`. O Caddy só
consegue pedir certificado se as portas 80 e 443 estiverem abertas e o domínio
apontar para a caixa.

### Onde ficam meus arquivos

Por padrão, num volume nomeado (`storage_data`) dentro do Docker. Para vê-los
de fora, aponte uma pasta da sua máquina no `.env`, do mesmo jeito que o
[Passo 8](#passo-8-ligar-o-backup) faz com os backups:

```
TAINA_STORAGE_DIR=/mnt/storage
```

Crie a pasta e dê a ela ao usuário do container antes de subir, senão o Tainá
não consegue escrever:

```sh
sudo mkdir -p /mnt/storage
sudo chown 65534:65534 /mnt/storage
```

Depois rode `./update.sh` para o app subir usando o caminho novo.

O banco da comunidade é um arquivo (`taina.db`) num volume nomeado (`db_data`),
do mesmo jeito. Ele é pequeno e o backup do [Passo 8](#passo-8-ligar-o-backup)
já o inclui, então não precisa mexer nele à mão: o `TAINA_DB_DIR` existe só
para quem quiser guardá-lo num disco próprio.

Para o dia a dia, a tela **Saúde da comunidade** mostra o uso em linguagem
comum, que é o jeito recomendado de olhar.

## Quanto tempo leva

| Passo | Primeira vez | Da próxima vez |
| --- | --- | --- |
| 1. Docker | 5 min | 0 |
| 2. Baixar o Oka | 1 min | 0 |
| 3. Montar a imagem (só se necessário) | 10 a 20 min | 1 a 3 min |
| 4. Instalar | 2 a 5 min | 0 |
| 5. Criar a comunidade | 2 min | 0 |
| 6. Convidar a primeira pessoa | 2 min | 0 |
| 7. Acesso remoto (Tailscale) | 10 min | 0 |
| 8. Ligar o backup | 15 min | 0 |

Do passo 1 ao 6, uma comunidade não-técnica costuma levar **menos de 30
minutos**. Os passos 7 e 8 são uma vez só, quando a comunidade começa a usar de
verdade.

Se demorar muito mais do que isso, algo está travado e o guia está errado. Isso
é informação que vale tanto quanto o Tainá funcionando: abra uma issue
contando onde parou.

## Licença

Este guia é parte do [Tekoá](https://github.com/taina-labs/tekoa), o
repositório de documentação do [Tainá](https://github.com/taina-labs/taina), sob
a [GNU Affero General Public License v3.0](../LICENSE).
