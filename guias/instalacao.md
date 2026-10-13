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
3. [Passo 2: instalar a CLI](#passo-2-instalar-a-cli)
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
- Docker. É o [Passo 1](#passo-1-instalar-o-docker).
- A CLI `taina`. É o [Passo 2](#passo-2-instalar-a-cli).
- Storage de verdade: um pendrive, um disco externo ou uma partição do próprio
  disco que não seja do sistema.

Um cartão SD comum é o pior lugar para guardar os arquivos de uma comunidade.
Ele é lento e as fotos da família o matam. Se a sua caixa é um Raspberry Pi, use
um SSD USB. Isso muda mais a experiência do Tainá do que qualquer configuração
deixada de fora.

## Passo 1: instalar o Docker

O Tainá roda dentro do Docker. Isso mantém o resto da sua máquina limpo e faz a
atualização ser um comando só.

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
```

Uma linha com o número de versão, e está pronto.

**No Windows e no macOS**, instale o Docker Desktop pelo site
[docker.com](https://www.docker.com/products/docker-desktop/) e abra o
aplicativo uma vez antes de continuar. No Windows, abra o Docker Desktop e
confirme que ele está no modo Linux.

## Passo 2: instalar a CLI

A CLI `taina` é o programa que opera a caixa: instala, atualiza, mostra os logs
e faz o backup. Baixe o binário da
[página de releases do Oka](https://github.com/taina-labs/oka/releases) para a
arquitetura da sua caixa: `aarch64-linux` num Raspberry Pi, `x86_64-linux` num
mini PC ou VPS. Depois:

```sh
sudo install -m755 taina-aarch64-linux /usr/local/bin/taina
taina version
```

O último comando deve imprimir algo como `taina 0.1.0`. Se imprimir, está
pronto.

Quem usa Nix instala direto:

```sh
nix profile install github:taina-labs/oka
```

## Passo 3: montar a imagem do Tainá

**Este passo só é necessário se você quiser partir do código-fonte**, ou se a
imagem publicada ainda não existir. A forma normal de instalar é o
[Passo 4](#passo-4-instalar), que baixa a imagem pronta do Tainá. Se o `setup`
reclamar que a imagem não está disponível no registro, volte para cá e rode:

```sh
git clone https://github.com/taina-labs/oka.git
cd oka
./build-image.sh
```

O que isso faz: baixa o código do Tainá para dentro da pasta `oka/taina-src` e
compila a imagem na sua própria máquina. Na primeira vez leva vários minutos,
porque baixa o compilador do Elixir e as dependências todas. Depois disso leva
pouco.

Ao terminar, ele diz como apontar a CLI para essa imagem:

```sh
TAINA_IMAGE=taina:local taina setup
```

Para publicar uma versão nova depois de mexer no código, rode o
`./build-image.sh` de novo (ele reusa a pasta `taina-src`) e depois
`taina update`.

## Passo 4: instalar

```sh
taina setup
```

O que acontece, em ordem:

1. Ele confere se o Docker está presente.
2. Ele escreve um arquivo `taina.conf` com um segredo sorteado, que assina as
   sessões. Guarde esse arquivo com cuidado. Quem tem ele tem acesso à caixa.
3. Ele cria a rede e sobe o container do Tainá, com o banco da comunidade (um
   arquivo SQLite que o app mesmo cuida).
4. Ele espera o aplicativo responder que está saudável, o que prova que o app
   subiu. Isso leva cerca de um minuto na primeira vez.
5. Ele imprime o endereço para o passo seguinte.

Rode o comando mais de uma vez sem problema: um `taina.conf` que já existe
nunca é sobrescrito, então seu segredo e seus dados continuam onde estão.

### Expor ao mundo

A CLI **não configura o HTTPS**, e isso é uma decisão: quem expõe a caixa é
você. Há dois caminhos.

- **Tailscale (recomendado).** `tailscale serve --bg 4000` publica a caixa com
  HTTPS de verdade, sem abrir porta nenhuma. É o
  [Passo 7](#passo-7-acessar-de-fora-da-sua-rede).
- **Seu domínio com Let's Encrypt.** Precisa de um proxy na frente da caixa. O
  Oka traz um `Caddyfile` de exemplo para montá-lo, e o
  `taina setup --domain nuvem.suacomunidade.org` escreve o endereço certo nas
  URLs que o Tainá monta (convites, links).

O `taina.conf` guarda o domínio. Para mudar depois, edite a linha
`TAINA_DOMAIN=` no arquivo e rode `taina update`.

## Passo 5: criar a comunidade

Abra no navegador o endereço que o `setup` imprimiu no final. Ele termina em
`/setup`. É algo como `https://nuvem.suacomunidade.org/setup`.

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
3. Na caixa, publique o Tainá no Tailscale:

   ```sh
   tailscale serve --bg 4000
   ```

Pronto. Cada pessoa passa a acessar o Tainá pelo endereço Tailscale, com HTTPS
de verdade, sem aviso do navegador. Escreva esse endereço no `TAINA_DOMAIN` do
`taina.conf` e rode `taina update`, para que os convites saiam com ele.

## Passo 8: ligar o backup

O backup vem **desligado**. Uma comunidade de verdade precisa dele ligado, e
precisa de uma cópia em outro lugar, não só no mesmo disco.

O [runbook de backup e restauração](runbook-backup-restore.md) tem o
procedimento inteiro, incluindo como restaurar e como ensaiar. O resumo:

1. **Monte um disco externo** na caixa. Por exemplo, um disco em `/mnt/backup`.
2. **Dê o disco ao Tainá.** O container roda como um usuário sem privilégios
   (`nobody`, uid 65534), então a pasta precisa pertencer a ele, senão o
   backup falha na hora de escrever:

   ```sh
   sudo chown 65534:65534 /mnt/backup
   ```

3. **Aponte o backup para ele.** No `taina.conf`, descomente (ou escreva) a
   linha do caminho de backup:

   ```
   TAINA_BACKUP_DIR=/mnt/backup
   ```

   Sem essa linha, o backup mora num volume nomeado (`taina_backups`) dentro do
   Docker: fica no mesmo disco do sistema, e some com o aparelho. Um caminho de
   disco é o seu disco, e você pode trocar, levar embora ou guardar em outro
   cômodo. O mesmo vale para os arquivos de verdade: o `TAINA_STORAGE_DIR`
   aponta a pasta de uploads para outro lugar (veja
   ["Onde ficam meus arquivos"](#onde-ficam-meus-arquivos) mais abaixo).
4. **Ligue o backup.** No `taina.conf`, mude a linha:

   ```
   TAINA_BACKUP_ENABLED=true
   ```

5. **Aplique:**

   ```sh
   taina update
   ```

   A partir daí um backup é feito todo dia às 04:00 UTC.

6. **Gere um agora e ensaie restaurar.** Um backup que nunca foi testado não é
   backup. O runbook tem o passo a passo do ensaio:

   ```sh
   taina backup --out /mnt/backup
   ```

Sobre a pasta de backup: uma cópia que mora no mesmo aparelho que ela protege
desaparece junto se o aparelho for roubado, queimar ou o disco falhar. Tenha
pelo menos uma cópia em outro lugar. Um pendrive que você guarda guardado vale
mais do que uma pasta no mesmo disco.

## Atualizar o Tainá

```sh
taina update
```

Ele baixa a versão nova, recria o container e confere a saúde. As migrações de
banco rodam sozinhas no boot, então atualizar é seguro e você não precisa rodar
nenhum comando no banco.

Antes de mexer em qualquer coisa, o `update` guarda uma cópia do banco na pasta
`rollback/`, ao lado de onde você rodou o comando. Se o Tainá não voltar a ficar
saudável, ele não te deixa no escuro: imprime os comandos exatos para voltar
para a versão anterior, inclusive como devolver esse banco.

Fique claro o que essa rede de proteção cobre: ela é só do banco de dados, é
feita na hora e mora na própria caixa. Ela existe para o dia em que uma
atualização sai errada, e não substitui o backup de verdade do
[Passo 8](#passo-8-ligar-o-backup), que é diário, inclui os arquivos e mora
fora da caixa. Os arquivos de storage não são mexidos pela atualização: ficam
no volume ou no disco deles, intocados.

Se algo der errado depois de atualizar, o `update` diz como voltar para a
versão anterior.

## Quando algo dá errado

### O `setup` diz que a imagem não está disponível

Ele não conseguiu baixar a imagem do Tainá. Rode o
[Passo 3](#passo-3-montar-a-imagem-do-tainá) e depois o
[Passo 4](#passo-4-instalar) de novo.

### O navegador avisa que a conexão não é segura

Esperado numa caixa sem domínio próprio: você está com um certificado local (ou
acessando por IP). As páginas funcionam normalmente, com o aviso. Para um HTTPS
sem aviso, use Tailscale ([Passo 7](#passo-7-acessar-de-fora-da-sua-rede)) ou
um domínio com proxy na frente.

### O app não ficou saudável a tempo

Veja os logs:

```sh
taina logs -f
```

A causa mais comum é a primeira migração do banco ainda rodando. Espere um
minuto e rode `taina health`. Se o erro se repetir, copie a saída dos logs e
abra uma issue.

### A caixa rodou o Oka antigo (com `docker compose`)

O Tainá passou a ser operado pela CLI, e o banco é um arquivo SQLite; não há
mais `docker-compose.yml` nem serviço de banco de dados. Sendo alpha, não há
dado real a perder, então recomece do zero:

```sh
docker rm -f taina-app
docker volume rm taina_db taina_storage taina_backups
taina setup
```

Atenção: isso apaga os dados da caixa, incluindo o storage. Se há arquivos ali
que você queira guardar, copie para fora antes.

### O convite não abre no celular do morador

O convite é um endereço da sua rede local, e o celular dele está fora. Resolva
o acesso remoto primeiro ([Passo 7](#passo-7-acessar-de-fora-da-sua-rede)) e
convide de novo, agora com o endereço que o guia der.

### Onde ficam meus arquivos

Por padrão, num volume nomeado (`taina_storage`) dentro do Docker. Para vê-los
de fora, aponte uma pasta da sua máquina no `taina.conf`, do mesmo jeito que o
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

Depois rode `taina update` para o app subir usando o caminho novo.

O banco da comunidade é um arquivo (`taina.db`) num volume nomeado (`taina_db`),
do mesmo jeito. Ele é pequeno e o backup do [Passo 8](#passo-8-ligar-o-backup)
já o inclui, então não precisa mexer nele à mão: o `TAINA_DB_DIR` existe só
para quem quiser guardá-lo num disco próprio.

Para o dia a dia, a tela **Saúde da comunidade** mostra o uso em linguagem
comum, que é o jeito recomendado de olhar.

## Quanto tempo leva

| Passo | Primeira vez | Da próxima vez |
| --- | --- | --- |
| 1. Docker | 5 min | 0 |
| 2. Instalar a CLI | 1 min | 0 |
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
