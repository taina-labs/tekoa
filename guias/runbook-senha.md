# Runbook de Recuperação de Senha do Tainá

> Documento escrito em 22/10/2026, depois do achado C1 da revisão 0.1.0.

O Tainá não usa e-mail: ninguém recebe um "esqueci minha senha" por fora. A
recuperação existe em três camadas, da mais comum para a mais grave. Leia as
três antes de agir: cada uma serve para uma pessoa em uma situação diferente.

## Índice

1. [As três camadas](#as-três-camadas)
2. [Camada 1: a pessoa lembra da senha atual](#camada-1-a-pessoa-lembra-da-senha-atual)
3. [Camada 2: a pessoa não consegue entrar](#camada-2-a-pessoa-não-consegue-entrar)
4. [Camada 3: quem esqueceu foi o zelador](#camada-3-quem-esqueceu-foi-o-zelador)

## As três camadas

| Quem | Situação | Caminho |
|---|---|---|
| Qualquer morador | Ainda entra (ou lembra da senha atual) | Conta, "Trocar minha senha" |
| Qualquer morador | Não entra | Conta de quem cuida, "Gerar link de redefinição" em Moradores |
| O zelador (ou quem cuida) | Não entra e ninguém com acesso à interface pode gerar o link | A tarefa `mix taina.senha` no shell (este guia, camada 3) |

Regra de ouro: as duas primeiras camadas bastam para 99% dos casos e não
exigem shell. A terceira existe porque uma comunidade pequena pode ter um
único zelador, e se for ele quem esquece, não sobra ninguém dentro para gerar
o link. É o cenário mais grave e o motivo desta tarefa existir.

## Camada 1: a pessoa lembra da senha atual

Quem está com a conta aberta (ou lembra da senha que usa hoje) troca a própria
senha em **Conta > Trocar minha senha**. Confirma a senha atual, escolhe uma
nova com pelo menos 8 caracteres, e pronto. Ninguém mais é avisado: é assunto
da pessoa.

Recomende essa troca de vez em quando, do jeito que se recomenda trocar a
senha de qualquer serviço.

## Camada 2: a pessoa não consegue entrar

Quem cuida da comunidade (zelador) gera um link de redefinição em
**Moradores**, no menu da pessoa (ícone de link, ao lado do nome). O link vale
1 hora e só pode ser usado uma vez. Entregue pelo mesmo canal do convite
(WhatsApp, carta, o que a comunidade já usa).

Detalhe importante: o link não aparece para a própria linha do zelador, porque
quem está dentro troca a senha na camada 1. O link é para quem está fora.

## Camada 3: quem esqueceu foi o zelador

Cenário: a única pessoa que cuida da caixa esqueceu a senha. Não há ninguém
logado para gerar o link da camada 2. O operador (quem tem acesso à máquina:
o zelador de antes, um parente, quem quer que instale a caixa) define uma
senha nova no shell, com a tarefa `taina.senha`.

### O que você precisa

- Uma máquina com o código-fonte do Tainá (a mesma versão da caixa ou mais
  nova), Elixir e Erlang instalados. Como a verificação de backup do runbook
  de restauração, esta tarefa é do Mix: ela não roda dentro do container do
  release, que não traz o Mix.
- Acesso ao arquivo do banco da caixa (o SQLite). Na instalação padrão ele
  vive em `/app/data/taina.db` no volume da caixa.

### O procedimento

1. **Prepare a máquina.** No diretório do código-fonte:

   ```sh
   mix deps.get
   ```

2. **Rode a tarefa apontando para o banco da caixa.** O usuário aceita só o
   nome (`maria`) ou o handle inteiro (`maria@tekoa.example.org`):

   ```sh
   DATABASE_PATH=/caminho/da/taina.db mix taina.senha maria
   ```

   A senha nova é pedida duas vezes, com a entrada escondida. Para rodar
   scriptado, passe tudo de uma vez:

   ```sh
   DATABASE_PATH=/caminho/da/taina.db mix taina.senha maria --password nova-senha-123
   ```

   A caixa pode estar ligada durante a tarefa: a escrita é pequena e o banco
   serializa sozinho.

3. **Confirme.** A tarefa responde "Senha de Maria definida." Entre na
   interface com o usuário e a senha nova. Se a conta estava desativada, a
   tarefa avisa: reative em Moradores antes de tentar entrar.

4. **Troque a senha de novo, como a pessoa.** A senha que o operador digitou
   no shell passou pela sua mão. Entre como o usuário (ou mande a pessoa
   entrar) e troque em Conta, camada 1, para uma senha que só ela saiba.

### O que a tarefa faz e o que não faz

- Faz: define a senha nova e queima qualquer link de redefinição pendente
  (ninguém mais entra com ele).
- Não faz: reativar conta desativada, mudar papel, apagar nada.
- Não entra no mural: o mural registra atos de pessoas dentro da caixa, e
  este ato aconteceu no shell. Se a sua comunidade anota esses eventos num
  caderno, anote também este, com data e quem executou.
