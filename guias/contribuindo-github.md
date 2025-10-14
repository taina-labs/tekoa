# Guia de Contribuição via GitHub

> Este guia explica como contribuir com o Tainá usando GitHub Issues e Projects, independente do seu nível técnico.

## Por que GitHub?

O Tainá é um projeto open source comunitário. Isso significa que toda a organização do trabalho precisa ser:

- **Transparente**: Qualquer pessoa pode ver o que está sendo feito
- **Acessível**: Sem barreiras de entrada ou convites exclusivos
- **Colaborativa**: Discussões públicas sobre decisões do projeto
- **Permanente**: Histórico completo junto com o código

GitHub Issues e Projects atendem todos esses requisitos sem custo adicional e com ferramentas que a maioria da comunidade de software já conhece.

### O que é uma Issue?

Uma issue é uma tarefa, problema ou sugestão registrada no projeto. Pode ser:

- Um bug para corrigir
- Uma nova funcionalidade para implementar
- Uma melhoria na documentação
- Uma sugestão de design
- Uma dúvida ou discussão

### O que é um Project?

Projects são quadros visuais que organizam issues em colunas como "A Fazer", "Em Andamento" e "Concluído". Permitem visualizar o progresso do projeto de forma clara.

---

## Estrutura do Projeto

O Tainá é um monólito. Todo o código vivem em um único repositório:

```
https://github.com/taina-labs/taina
```

### Organização das Issues

As issues são categorizadas por labels que representam contextos de trabalho:

| Label | Descrição | Exemplos |
|-------|-----------|----------|
| `engineering` | Desenvolvimento técnico | Backend, frontend, infraestrutura |
| `design` | Design visual e UX | Logo, mockups, identidade visual |
| `documentation` | Documentação e guias | Tutoriais, tradução, READMEs |
| `economics` | Pesquisa de hardware e custos | Preços, fornecedores, análises |
| `community` | Comunidade e governança | Eventos, outreach, feedback |

https://github.com/taina-labs/taina/labels

### Como navegar no repositório

<img src="../midia/navegacao-basica.gif" />

---

## Como Contribuir por Contexto

### Desenvolvimento Técnico

#### Encontrar uma tarefa

1. Acesse a aba "Issues"
2. Filtre por label `engineering`
3. Procure issues marcadas como `good first issue` para iniciantes

#### Workflow básico

1. **Fork**: Crie uma cópia do repositório na sua conta
2. **Branch**: Crie uma branch para sua mudança
3. **Commit**: Faça commits com mensagens claras
4. **Push**: Envie suas mudanças para seu fork
5. **Pull Request**: Abra um PR para o repositório principal

#### Padrão de commits

Use Conventional Commits para facilitar o rastreamento:

```
feat: adiciona autenticação por email
fix: corrige upload de arquivos no Ybira
docs: atualiza guia de instalação
style: ajusta espaçamento no formulário
```

#### Como linkar sua PR à issue

No título ou descrição do Pull Request, mencione a issue:

```
Resolve #42
```

Isso fecha automaticamente a issue quando o PR for aprovado.

---

### Design

#### Como contribuir com design

1. Abra uma nova issue
2. Use o template de design (se disponível) ou descreva:
   - O que você quer criar/melhorar
   - Qual problema isso resolve
   - Referências visuais (se houver)

#### Anexando mockups e assets

Você pode arrastar arquivos diretamente para o campo de descrição:

- Imagens: PNG, JPG, GIF
- Arquivos: PDF, SVG
- Links: Figma, Adobe XD, etc.

#### Processo de aprovação

1. Mantenedores revisam a proposta
2. Discussão acontece nos comentários da issue
3. Após aprovação, você pode implementar ou alguém pega a tarefa
4. Issue é fechada quando o design é integrado

---

### Documentação

#### Como sugerir melhorias

Se você encontrou um erro ou quer melhorar um documento:

1. Vá até o arquivo no repositório
2. Clique no ícone de lápis (Edit)
3. Faça suas alterações
4. No final da página, descreva o que mudou
5. Clique em "Propose changes"

Isso cria automaticamente um fork e um Pull Request para você. Não precisa instalar nada no seu computador.

#### Traduzindo guias

Para traduzir documentos:

1. Crie uma issue sugerindo a tradução
2. Indique qual documento e para qual idioma
3. Após aprovação, siga o processo de edição acima
4. Mantenha a estrutura e formatação do original

#### Dicas de escrita

- Use linguagem clara e acessível
- Evite jargões técnicos desnecessários
- Prefira exemplos práticos a explicações abstratas
- Revise ortografia e gramática antes de submeter

---

### Economia e Hardware

#### Como atualizar informações de hardware

O guia de hardware está em `guias/compras-hardware.md`. Para atualizá-lo:

1. Abra uma issue reportando o que está desatualizado:
   - Preços que mudaram
   - Links quebrados
   - Novos fornecedores ou produtos

2. Se souber fazer a correção, edite o arquivo direto (processo acima)

#### Sugerindo novos fornecedores

Ao sugerir um fornecedor ou produto:

- Inclua link direto para o produto
- Especifique o preço atual
- Mencione disponibilidade (nacional/importação)
- Se possível, compare com alternativas

---

### Comunidade

#### Reportando feedback de usuários

Se você testou o Tainá ou conversou com potenciais usuários:

1. Abra uma issue com label `community`
2. Descreva o feedback recebido
3. Contextualize: quantas pessoas, perfil, necessidades específicas

#### Organizando eventos

Para propor eventos (workshops, meetups, talks):

1. Crie uma issue descrevendo:
   - Tipo de evento
   - Data/local proposto
   - Público-alvo
   - O que você precisa de ajuda

2. Use a issue para coordenar com outros voluntários

---

## Navegação no GitHub

### Issues

#### Criando uma issue

1. Vá em "Issues" no topo do repositório
2. Clique em "New Issue"
3. Preencha:
   - **Título**: Breve e descritivo
   - **Descrição**: Contexto completo do problema/sugestão
   - **Labels**: Selecione o contexto adequado
4. Clique em "Submit new issue"

#### Comentando em issues

Para participar de discussões:

1. Role até o final da issue
2. Digite seu comentário (suporta Markdown)
3. Use @username para mencionar alguém
4. Clique em "Comment"

#### Linkando issues relacionadas

Para conectar issues que tratam do mesmo tema:

```
Relacionado a #42
Depende de #38
```

Isso cria links clicáveis entre as issues.

### Projects

O Tainá usa GitHub Projects para visualização do roadmap.

#### Como acessar

1. Na página principal do repositório
2. Clique na aba "Projects"
3. Selecione "Tainá Roadmap"

Ou acesse diretamente: https://github.com/orgs/taina-labs/projects/1

#### Views disponíveis

- **Board**: Kanban tradicional (Todo, In Progress, Done)
- **Roadmap**: Timeline visual das fases do projeto (Phase 1-5)
- **By Context**: Agrupado por label (design, engineering, docs, etc)

#### Filtrando por contexto

Você pode filtrar issues por label diretamente no Project:

```
label:design
label:documentation
```

### Discussions

Para conversas que não são tarefas específicas, use o fórum Discourse:

https://taina-forum.zeetech.io

#### Quando usar Discourse vs Issue

**Use Discourse para:**
- Perguntas gerais ("Como funciona X?")
- Ideias ainda não estruturadas
- Compartilhar experiências
- Anúncios para a comunidade

**Use Issue para:**
- Tarefas específicas
- Bugs concretos
- Features bem definidas
- Trabalho que alguém vai executar

---

## FAQ para Iniciantes

### Nunca usei GitHub. É complicado?

Não. Para contribuições básicas (abrir issues, comentar, editar docs), você só precisa de:

1. Criar uma conta no GitHub
2. Saber escrever em Markdown (é simples, veja abaixo)
3. Entender o básico de navegação web

Se você sabe usar um fórum online, já sabe o suficiente.

### Preciso saber programar para contribuir?

Não. O Tainá precisa de contribuições em várias áreas:

- Design visual e de interface
- Escrita de documentação
- Pesquisa de hardware
- Feedback de usuários
- Organização de eventos
- Tradução de conteúdo

Apenas desenvolvimento técnico requer conhecimento de programação.

### O que é Markdown?

Markdown é uma forma simples de formatar texto. Exemplos:

```markdown
# Título grande
## Título médio

**negrito**
*itálico*

- Item de lista
- Outro item

[link](https://exemplo.com)
```

Você pode usar o botão "Preview" ao escrever para ver como ficará.

### Como sei se alguém já está trabalhando em algo?

Verifique:

1. **Assignees**: Se a issue tem alguém atribuído
2. **Comments**: Leia os comentários recentes
3. **Pull Requests**: Veja se já tem um PR linkado

Se nenhum dos três existir, pergunte nos comentários:

```
Olá! Posso trabalhar nessa issue?
```

### Cometi um erro. Como corrijo?

Depende do tipo de erro:

**Na issue/comentário**: Clique nos três pontos (...) e "Edit"

**No código enviado**: Faça um novo commit na mesma branch

**PR já foi aprovado**: Abra uma nova issue ou PR com a correção

Não se preocupe. Erros fazem parte do processo, e a comunidade está aqui para ajudar.

---

## Boas Práticas

### Ao criar issues

- **Seja específico**: "O botão de upload não funciona no Safari" em vez de "Upload quebrado"
- **Contextualize**: Inclua informações de ambiente (navegador, OS, versão)
- **Um problema por issue**: Não misture vários temas em uma única issue
- **Busque antes**: Verifique se já existe issue similar

### Ao comentar

- **Seja respeitoso**: Lembre-se que todos são voluntários
- **Adicione valor**: Evite comentários tipo "+1" (use reações 👍 para concordar)
- **Seja construtivo**: Critique ideias, não pessoas
- **Mantenha foco**: Se o assunto divergir muito, abra nova issue

### Ao fazer Pull Requests

- **Descreva o que mudou**: Não apenas "Fix #42"
- **Teste suas mudanças**: Pelo menos teste localmente antes de submeter
- **Responda feedback**: Code review é uma conversa, não crítica pessoal
- **Seja paciente**: Revisão leva tempo

---

## Recursos Adicionais

### Aprendendo Git e GitHub

- [GitHub Docs](https://docs.github.com) - Documentação oficial
- [GitHub Skills](https://skills.github.com) - Tutoriais interativos
- [Markdown Guide](https://www.markdownguide.org) - Guia de Markdown

### Comunidade Tainá

- [Fórum Discourse](https://taina-forum.zeetech.io) - Discussões gerais
- [GitHub Issues](https://github.com/taina-labs/taina/issues) - Tarefas e bugs
- [GitHub Projects](https://github.com/orgs/taina-labs/projects/1) - Roadmap visual
- Consulte também o [Código de Conduta](https://github.com/taina-labs/tekoa/blob/main/CODE_OF_CONDUCT.md)

---

## Ajuda e Suporte

Se você ficou travado ou tem dúvidas:

1. Consulte a [documentação do GitHub](https://docs.github.com)
2. Pergunte no [fórum Discourse](https://taina-forum.zeetech.io)
3. Comente na issue relevante

Não tenha vergonha de perguntar. Todos começaram do zero algum dia.

---

_Este guia é um documento vivo e será atualizado conforme a comunidade cresce._
