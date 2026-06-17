# Guia de Acesso Remoto ao Tainá

> Documento escrito em 17/06/2026, atualizado conforme o deploy evolui.

O Tainá roda numa caixa da sua comunidade (um Raspberry Pi, um mini PC ou uma
VPS). Dentro da sua rede local você acessa pelo endereço da caixa. Este guia
trata de como acessar de fora: do celular na rua, da casa de outra pessoa, de
qualquer lugar.

Não existe uma resposta única. Você escolhe o ponto da escala que combina com a
sua comunidade, do mais fácil ao mais soberano. O Tainá nunca "liga para casa"
em nenhum dos casos: o acesso remoto é uma camada de rede que você coloca por
fora, separada do app. Se um dia você desligar o acesso remoto, o Tainá
continua funcionando normalmente na rede local.

## Índice

1. [A escala de opções](#a-escala-de-opções)
2. [O que "soberania" realmente significa aqui](#o-que-soberania-realmente-significa-aqui)
3. [Opção 1: Tailscale (recomendado)](#opção-1-tailscale-recomendado)
4. [Opção 2: Headscale (soberano)](#opção-2-headscale-soberano)
5. [Opção 3: WireGuard puro (avançado)](#opção-3-wireguard-puro-avançado)
6. [Opção 4: Domínio público + proxy](#opção-4-domínio-público--proxy)
7. [TLS na prática](#tls-na-prática-o-que-combina-com-o-que)
8. [Resumo e como escolher](#resumo-e-como-escolher)

---

## A escala de opções

| Caminho                   | Quem opera o controle     | Atravessa NAT | Esforço             | Para quem                 |
| ------------------------- | ------------------------- | ------------- | ------------------- | ------------------------- |
| Tailscale (recomendado)   | Tailscale (só metadados)  | sim           | quase zero          | comunidade não-técnica    |
| Headscale (soberano)      | você, num host alcançável | sim           | alto (segundo host) | quem não quer terceiros   |
| WireGuard puro (avançado) | você                      | manual        | máximo              | quem administra rede      |
| Domínio público + proxy   | n/a (IP público)          | não precisa   | baixo               | VPS ou IP público em casa |

A recomendação oficial do MVP (RFC 002, decisão D7) é o Tailscale, porque a tese
do Tainá é uma comunidade não-técnica colocar o servidor no ar em menos de 30
minutos. As outras opções existem porque o Tainá é software livre e auto-
hospedado: quem decide é você. Nada neste guia te prende a um fornecedor.

## O que "soberania" realmente significa aqui

Vale entender o limite de confiança antes de escolher, sem mito.

Uma rede tipo Tailscale ou Headscale tem dois planos separados:

- **O plano de controle** (o "coordenador"): é o serviço que faz os aparelhos se
  descobrirem e atravessarem o NAT do roteador. Ele enxerga chaves públicas,
  nomes de aparelhos e metadados de coordenação (quem está na rede, quando se
  conecta). Ele não enxerga o conteúdo dos seus arquivos.
- **O plano de dados**: o tráfego de verdade vai criptografado de ponta a ponta
  (WireGuard) direto entre os seus aparelhos. Quando a conexão direta não é
  possível, o tráfego passa por um relay (DERP), mas o relay só encaminha
  pacotes que não consegue ler.

Ou seja: usar o Tailscale não entrega seus arquivos a um terceiro. O custo de
soberania é outro, e é honesto deixar claro: você depende de um plano de
controle fechado, operado por uma empresa, e os metadados (quem conecta com
quem, e quando) passam por lá. Se isso incomoda a sua comunidade, o Headscale é
a resposta: ele é uma reimplementação livre desse coordenador, que você mesmo
roda.

Não há resposta certa universal. Uma associação de bairro provavelmente quer a
praticidade do Tailscale; um coletivo com ameaça de modelo mais forte vai
preferir o Headscale ou o WireGuard puro.

## Opção 1: Tailscale (recomendado)

Tailscale é uma rede privada (parecida com uma VPN) que "simplesmente funciona":
você instala um app na caixa e nos seus aparelhos, faz login, e eles passam a se
enxergar como se estivessem na mesma rede, sem abrir portas no roteador.

Passos:

1. Crie uma conta gratuita em tailscale.com. O plano gratuito cobre de sobra uma
   comunidade.
2. Na caixa do Tainá, instale o Tailscale e rode `tailscale up`. Anote o nome
   MagicDNS que aparece (algo como `taina.seudominio.ts.net`).
3. Instale o Tailscale nos celulares e computadores das pessoas da comunidade e
   faça login na mesma conta (a mesma "tailnet").
4. Acesse o Tainá pelo nome MagicDNS da caixa, de qualquer lugar.

### HTTPS sem dor (certificado de verdade, sem domínio próprio)

Este é o melhor caminho para ter o cadeado verde sem comprar domínio e sem
configurar nada no roteador. O Tailscale emite um certificado Let's Encrypt real
para o nome `*.ts.net` da sua caixa. Há duas formas:

```sh
# Forma A: deixar o Tailscale terminar o TLS e fazer proxy do Tainá (porta 4000).
tailscale serve --bg 4000

# Forma B: gerar o certificado e usá-lo no Caddy (caso prefira manter o Caddy
# como porta de entrada).
tailscale cert taina.seudominio.ts.net
```

Com isso o navegador confia no certificado sem você instalar nada nos aparelhos.
É exatamente o que o Home Assistant faz, e é o caminho mais tranquilo para
comunidades não-técnicas. Quando você usa esse caminho, deixe `TAINA_DOMAIN`
vazio no `.env`: o Caddy não precisa emitir certificado, o Tailscale já
resolveu.

### Verificando

No celular com Tailscale ligado, mas fora do Wi-Fi de casa (use os dados
móveis), abra `https://taina.seudominio.ts.net/health`. Deve responder com um
JSON de status. Se responder, o acesso remoto está de pé.

## Opção 2: Headscale (soberano)

Headscale é um coordenador Tailscale livre, que você mesmo hospeda. Os apps
clientes são os mesmos do Tailscale, só apontando para o seu servidor em vez do
serviço da empresa. Nenhum terceiro participa.

O ponto honesto: a caixa do Tainá normalmente está atrás de NAT (rede
doméstica), então ela não serve como coordenador para o mundo, porque o mundo
não consegue iniciar conexão até ela. Você vai precisar de um segundo host
alcançável pela internet rodando o Headscale, por exemplo uma VPS barata ou um
servidor em casa com porta liberada e IP fixo (ou DNS dinâmico). Isso é mais
infraestrutura para manter, e por isso não é o caminho padrão do MVP. Em troca,
elimina qualquer dependência de terceiros.

Topologia recomendada:

- Um host com IP público (uma VPS barata resolve) rodando o `headscale`.
- A caixa do Tainá entra na rede como um nó comum:

  ```sh
  tailscale up --login-server https://seu-headscale.exemplo.org
  ```

- Os aparelhos da comunidade entram do mesmo jeito, apontando o
  `--login-server` para o seu Headscale.

Rodar o Headscale na própria caixa do Tainá só faz sentido se ela já tiver IP
público e porta liberada (por exemplo, se o Tainá já está numa VPS). Nesse caso
o segundo host deixa de ser necessário.

Sobre HTTPS: o nome interno que o Headscale distribui (MagicDNS) também pode ter
certificado, mas a emissão é mais manual do que no Tailscale. Documente o passo
exato para a sua comunidade, ou use a Opção 4 (domínio público) se a caixa for
alcançável.

## Opção 3: WireGuard puro (avançado)

WireGuard é a tecnologia por baixo de Tailscale e Headscale. Dá para configurar
na mão, par a par, sem nenhum coordenador. É o caminho mais soberano e o mais
trabalhoso: você gerencia chaves, endereços de IP da rede privada e, se a caixa
estiver atrás de NAT, a travessia de NAT por conta própria (normalmente com um
peer de IP público fazendo o "salto" e `PersistentKeepalive` para manter o túnel
aberto).

Indicado para quem já administra rede e quer controle total. Para a maioria das
comunidades, Tailscale ou Headscale entregam o mesmo WireGuard por baixo, sem o
trabalho manual.

## Opção 4: Domínio público + proxy

Se a sua caixa tem IP público (uma VPS, ou casa com IP fixo e porta liberada), o
caminho mais direto é um domínio apontando para ela:

1. Aponte um domínio (ex.: `nuvem.suacomunidade.org`) para o IP da caixa (um
   registro A, ou AAAA para IPv6).
2. Libere as portas 80 e 443 no roteador/firewall em direção à caixa.
3. Rode o instalador informando o domínio:

   ```sh
   TAINA_DOMAIN=nuvem.suacomunidade.org ./install.sh
   ```

O Caddy pega um certificado Let's Encrypt real automaticamente (desafio HTTP-01,
que usa a porta 80) e o redirecionamento de HTTP para HTTPS sai de graça. Não há
warning de certificado, porque é um certificado público de verdade.

### E se a caixa tem domínio mas está atrás de NAT?

Use o desafio DNS-01: o certificado é emitido validando um registro DNS, sem
precisar abrir as portas 80/443. Atenção a uma pegadinha importante: a imagem
oficial do Caddy (`caddy:2`) não inclui os módulos de provedor de DNS. Para
DNS-01 você precisa de uma imagem do Caddy construída com o módulo do seu
provedor de DNS (via `xcaddy` ou a imagem `caddy:builder`), além de um token de
API do provedor. É um caminho avançado, documentado, opt-in: troque a imagem do
serviço `caddy` no `docker-compose.yml` pela sua imagem custom e ajuste o
`Caddyfile` com a diretiva `tls { dns <provedor> <token> }`.

## TLS na prática: o que combina com o que

| Sua situação                              | Estratégia de TLS                                              |
| ----------------------------------------- | -------------------------------------------------------------- |
| Acessa via Tailscale/Headscale            | Certificado real do overlay (`*.ts.net`), `TAINA_DOMAIN` vazio |
| Domínio público, portas 80/443 abertas    | Caddy + Let's Encrypt HTTP-01 (`TAINA_DOMAIN` preenchido)      |
| Domínio público atrás de NAT, ou wildcard | Caddy + DNS-01 (precisa de imagem custom do Caddy)             |
| Rede local pura, sem overlay, sem domínio | Caddy `tls internal` (CA local)                                |

Sobre o `tls internal`: o Caddy cria uma autoridade certificadora local e assina
o próprio certificado. O navegador vai avisar que o certificado não é confiável
até você instalar e confiar nessa raiz local em cada aparelho (no Android, em
Configurações > Segurança > Certificados; no iOS, instalando o perfil e
confiando manualmente). Isso é chato para gente não-técnica, e é justamente por
isso que, quando há acesso via Tailscale/Headscale, preferimos o certificado
real do overlay. Use `tls internal` só como último recurso numa caixa de rede
local pura, em que todo mundo está disposto a confiar na raiz uma vez.

## Resumo e como escolher

- Começou agora e a comunidade é não-técnica? **Tailscale**, com `tailscale
serve` para HTTPS real. Caminho recomendado, menos de 30 minutos.
- Não quer nenhum terceiro? **Headscale**, sabendo que vai manter um segundo
  host alcançável pela internet.
- Administra rede e quer controle total? **WireGuard puro**.
- Tem IP público e domínio? **Domínio + Caddy** com Let's Encrypt automático.

Qualquer que seja a escolha, ela é sua e reversível: o acesso remoto é uma
camada por fora do Tainá, não uma dependência dele.
