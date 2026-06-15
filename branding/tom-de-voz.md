# Tom de Voz, Tainá (produto e copy)

- **Status:** Vivo (aplicável já)
- **Escopo:** Voz do **produto** (texto de interface, mensagens, vazios, erros) e diretrizes de **design** que carregam essa voz. Complementa o `briefing.md` (identidade _visual_); aqui é sobre **palavras e postura**, não sobre cor e logo.
- **Idioma:** pt-BR primeiro, sempre. Inglês nunca vaza para a interface.

> O Tainá é a caixa de uma comunidade que decidiu cuidar dos próprios dados. A voz é a de uma vizinhança que se organiza, não a de uma plataforma que captura. Cada frase deve soar como alguém da aldeia falando com você, não como um produto otimizando você.

---

## 1. Princípios

### Comunitário, não individualista

Fale da comunidade, não da vaidade de uma pessoa. "Nossa aldeia", "a praça da comunidade", "o que guardamos juntos". Evite perfil, vitrine, métricas de status, curtidas/seguidores, feed algorítmico.

- Bom: "A praça é da comunidade. O que você coloca aqui, todos os moradores veem."
- Evite: "Seu perfil, 142 curtidas, 12 seguidores"

### Simples e acolhedor

Escreva para a pessoa **menos técnica** da comunidade. Sem jargão, sem conhecimento técnico presumido, sem becos sem saída. Ensine, não apenas informe.

- Bom: "Só você vê os arquivos da sua casa. Para outra pessoa abrir, ela pede e você decide."
- Evite: "Recurso protegido por ACL. Permissão de leitura negada (403)."

### Honesto, promessa que se cumpre

Nunca prometa o que o sistema não garante. A privacidade da casa é **promessa de software e de confiança**, não criptografia.

- Bom: "Promessa de software, não cadeado matemático: o Tainá ainda não criptografa os arquivos. Confie no servidor como confiaria em quem hospeda a caixa da comunidade."
- Evite: "Seus arquivos são 100% privados e ninguém nunca poderá vê-los."

### Anti-engajamento, calma é o padrão

"Design for less usage, not more" (`RFC_ARQUITETURA`). Nada de iscas de atenção: sem badge vermelho de urgência, sem sequências/streaks, sem padrões escuros. Avisos são **ambientes e opcionais**.

- Bom: "2 pedidos esperando sua resposta" (um aviso calmo, que some quando resolvido)
- Evite: "VOCÊ TEM 2 PEDIDOS! Não perca! Responda agora!"

### Poder visível

Quem cuida da máquina é **zelador(a)**, nunca dono ou chefe. Todo poder é legível no mural. A linguagem nomeia o cuidado, não o mando.

- Bom: "Quem cuida (Ana) aumentou a cota para 50 GB."
- Evite: "Admin alterou as configurações do sistema."

### Acessível é tom, não acabamento

Texto claro e curto, contraste e rótulos adequados, ícone sempre com texto, respeito a `prefers-reduced-motion`. Acessibilidade é parte da voz acolhedora, não um item separado de QA.

---

## 2. Convenção de nomes (uma fonte de verdade)

Três camadas, deliberadas, nunca misturar ao acaso (ver `tecnico/RFC_003_GOVERNANCA_E_TRANSPARENCIA.md` seção 4):

| Camada             | Quando                                                                  | Exemplos                                                                 |
| ------------------ | ----------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **Tupi-Guarani**   | nomes próprios de subsistema/serviço (como marca)                       | Tekoa, Maracá, Ybira, Jaci, Guará                                        |
| **pt-BR familiar** | vocabulário social do dia a dia (a _mesma_ palavra no código e na tela) | casa, praça, zelador(a), morador(a), mural, pedido de acesso, assembleia |
| **Inglês**         | só encanamento que o usuário nunca lê                                   | -                                                                        |

- Os nomes Tupi são explicados uma vez (onboarding) e nunca anglicizados.
- O vocabulário social é pt-BR familiar de ponta a ponta, sem termo técnico em inglês na tela, sem palavra Tupi para conceitos do cotidiano.

---

## 3. Mecânica de escrita

- **Pessoa e tempo:** "você" no singular acolhedor; "nós/nossa" para os comuns. Voz ativa, presente.
- **Frases curtas.** Uma ideia por frase. Sem subordinadas longas.
- **Verbos diretos:** "Pedir acesso", "Publicar na praça", "Tirar da praça", "Convidar". Evite "Submeter", "Processar", "Gerenciar".
- **Números em pt-BR:** vírgula decimal, "1,1 MB", "há 3 dias", "ontem".
- **Erros explicam e oferecem saída**: nunca culpam nem deixam no vácuo.
  - Bom: "Este convite expirou ou já foi usado. Peça um novo link a quem te convidou."
  - Evite: "Erro: token inválido."
- **Vazios convidam** (e podem ensinar o modelo):
  - Bom: (sem pedidos) "Ninguém acessa seus arquivos sem você deixar. Quando alguém pedir, aparece aqui."

---

## 4. Glossário de termos do produto

| Conceito                   | Termo (UI = código)              | Nunca dizer                                                 |
| -------------------------- | -------------------------------- | ----------------------------------------------------------- |
| Espaço pessoal e privado   | **casa**                         | "meus arquivos privados", "área pessoal"                    |
| Espaço comum da comunidade | **praça**                        | "público", "compartilhado com todos" (use só na explicação) |
| Quem cuida da máquina      | **zelador(a)**                   | "admin", "administrador", "dono"                            |
| Membro da comunidade       | **morador(a)**                   | "usuário", "membro" (em copy)                               |
| Registro do que aconteceu  | **mural**                        | "log", "feed", "atividade"                                  |
| Estado da máquina, legível | **Saúde da comunidade** (painel) | "dashboard", "métricas"                                     |
| Pedir para abrir uma casa  | **pedir acesso**                 | "solicitar permissão"                                       |
| Decisão coletiva (pós-MVP) | **assembleia**                   | "votação", "governança"                                     |

---

## 5. Checklist rápido (antes de mandar copy/tela)

- [ ] Fala como vizinhança, não como plataforma?
- [ ] A pessoa menos técnica entenderia de primeira?
- [ ] Promete só o que o sistema cumpre (sem fingir criptografia)?
- [ ] Sem isca de atenção / padrão escuro / urgência fabricada?
- [ ] Usa o termo do glossário (mesma palavra do código)?
- [ ] Tem estado de vazio/erro com saída clara?
- [ ] Acessível (contraste, rótulo, ícone + texto)?
