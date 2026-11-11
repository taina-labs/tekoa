# RFC 009: Backup pessoal portátil da casa

> **Ideia arquivada.** Não está no 1.0.0 nem em implementação. O roadmap atual registra somente a direção futura.

**Status:** Proposta
**Data:** 2026-11-01
**Relação:** Complementa o `RFC_007_BACKUP_CIFRADO.md`. A RFC 007 protege a continuidade da Tekoa inteira, com restore da instância e custódia comunitária. Esta RFC protege a portabilidade dos dados de uma pessoa, com leitura fora do Tainá e importação não destrutiva em outra Tekoa. Herda `casa`, `praça`, posse e escopo do `RFC_003_GOVERNANCA_E_TRANSPARENCIA.md`.
**Autoria:** @zoedsoupe + revisão técnica

---

## 1. Resumo executivo

Uma pessoa precisa conseguir tirar uma cópia dos próprios dados da caixa. O caso principal não é apenas desastre: é levar sua `casa` para o computador, para outro armazenamento ou para outra Tekoa sem depender da zeladoria e sem entregar o conteúdo em texto puro para um provedor.

Esta RFC define o **backup pessoal portátil**: um arquivo único, cifrado, gerado sob demanda e acompanhado por um kit individual de recuperação. Depois de aberto, o backup contém uma árvore de pastas e arquivos comuns, legível sem o Tainá, e um `manifest.json` para importação estruturada.

Cinco decisões:

1. O conteúdo aberto é simples: arquivos, pastas e JSON. Não exige banco, kernel ou outro Tainá para ser lido.
2. O recipiente é `tar` cifrado com `age`, dois formatos abertos, com implementações para computador e ARM.
3. Cada backup tem um kit individual, que é uma identity secreta `age`. Não há senha humana, Shamir, grupo de custodiantes ou cerimônia coletiva.
4. Backup pessoal não vira notícia no mural. A portabilidade individual não deve virar vigilância social.
5. Importar em uma Tekoa cria novos arquivos na `casa` da pessoa autenticada. Não restaura conta, permissões, mural ou histórico e nunca sobrescreve o estado atual no primeiro corte.

O que esta RFC não promete: E2E ou proteção contra root na caixa. A caixa lê os arquivos para produzir o backup. A cifra protege o arquivo depois que ele sai da caixa.

---

## 2. Casos de uso e limites

### 2.1 Casos principais

- guardar uma cópia própria no computador;
- mover a `casa` para outra Tekoa;
- guardar o arquivo cifrado num provedor qualquer sem dar a ele o kit;
- abrir os arquivos sem instalar o Tainá;
- importar uma cópia antiga na mesma Tekoa sem substituir os arquivos atuais.

### 2.2 O que este backup não é

- não é backup da instância;
- não restaura uma conta, uma comunidade ou sua chave privada;
- não preserva a cadeia do mural nem concessões de acesso;
- não é sincronização contínua na primeira versão;
- não inclui arquivos de outras pessoas;
- não é uma exportação da `praça` inteira.

A diferença de propósito importa: o backup da instância restaura estado social e operacional; o backup pessoal transporta dados pertencentes à pessoa.

---

## 3. Escopo e consentimento

O backup padrão inclui somente arquivos e pastas que, no ponto de exportação:

- pertencem à pessoa autenticada;
- estão em `casa`;
- não estão apagados nem na lixeira;
- são alcançáveis pela árvore de pastas da própria pessoa.

Não entram:

- arquivos de outras pessoas, mesmo quando existe concessão de leitura;
- contribuições publicadas na `praça` neste primeiro corte;
- miniaturas e outras representações derivadas;
- senha, hash de senha, sessões, tokens ou dispositivos;
- pedidos, concessões, votos, propostas ou entradas do mural;
- chaves da Tekoa, keyrings ou credenciais de backup;
- metadados de outros moradores ou referências de deduplicação.

Uma concessão de leitura permite ler dentro das regras atuais, mas não transforma o arquivo alheio em dado pessoal exportável. Incluir as próprias contribuições na `praça` pode ser uma opção futura, sempre separada da `casa` e explícita.

A seleção passa pelo `current_scope` e pelas regras de posse. Não se produz um subconjunto do SQLite inteiro, porque isso aumentaria o risco de vazamento de registros alheios. O artefato nasce de uma lista autorizada de arquivos e pastas.

---

## 4. Formato aberto

O arquivo baixado tem nome sem username ou nome da comunidade:

```text
taina-casa-<timestamp>.tar.age
```

Depois de decifrado, o `tar` contém:

```text
README.txt
manifest.json
files/
|-- Documentos/
|   `-- ata.pdf
`-- Fotos/
    `-- festa.jpg
```

`files/` preserva a hierarquia e os nomes originais sempre que o filesystem de destino permitir. Uma pessoa pode abrir esses arquivos com qualquer programa compatível, sem entender o manifesto.

`README.txt` explica em português simples:

- o que o pacote contém;
- que o kit é necessário para abrir outra cópia cifrada;
- como verificar o manifesto;
- que importar em uma Tekoa cria novos arquivos, sem recuperar histórico social.

Não há banco SQLite, histórico ou formato binário específico do Tainá no pacote. O JSON é a única camada adicional que uma importação estruturada precisa entender.

### 4.1 Manifesto

O manifesto usa JSON UTF-8 e chaves versionadas. Exemplo:

```json
{
  "format": "taina-casa-v1",
  "export_id": "01K...",
  "created_at": "2026-10-31T12:00:00Z",
  "source": {
    "owner": "zoey"
  },
  "folders": [
    {
      "path": "Documentos",
      "created_at": "2026-01-10T14:00:00Z"
    }
  ],
  "files": [
    {
      "path": "Documentos/ata.pdf",
      "size": 182010,
      "sha256": "...",
      "mime_type": "application/pdf",
      "created_at": "2026-01-10T14:03:00Z",
      "updated_at": "2026-01-10T14:03:00Z"
    }
  ]
}
```

O manifesto contém somente dados necessários para conferência e importação. Não contém CIDs de outros registros, contagens globais de referências, permissões, nomes de outros moradores ou histórico de operações.

`path` é o caminho legível dentro de `files/`. O importador deve rejeitar caminhos absolutos, `..`, separadores ambíguos, links, dispositivos e nomes que escapem do staging. Colisões de nomes em filesystems de destino são tratadas pela política de importação, nunca por sobrescrita silenciosa.

`sha256` é calculado sobre o conteúdo original. O importador confere todos os hashes antes de criar registros. Tamanho declarado, bytes reais e MIME precisam ser compatíveis.

---

## 5. Cifra e kit individual

Cada export gera um novo par de chaves `age`. O destinatário público cifra o `tar`; a identity privada forma o kit:

```text
# Backup pessoal do Tainá
# export: <export_id>
# criado em: <timestamp>
AGE-SECRET-KEY-...
```

O kit é um arquivo de identity `age` padrão. Não carrega arquivos, manifesto ou chave da Tekoa. O checksum e a codificação seguem o formato `age`, sem criar uma codificação paralela.

Propriedades:

- uma identity nova por export;
- nenhuma senha humana ou KDF de senha;
- nenhum compartilhamento `k-de-n`;
- nenhuma confirmação de outras pessoas;
- nenhuma cópia persistente da identity no banco;
- nenhum registro da identity em logs, mural ou telemetria;
- perda do kit torna aquele pacote ilegível, mas não impede gerar um novo export enquanto a caixa e a conta estão disponíveis.

A caixa mantém a identity somente durante a cerimônia curta de geração e entrega. Depois da confirmação ou expiração, descarta o material pendente. Como na RFC 007, não se promete zeroização física de toda cópia na BEAM; reduz-se tempo de vida, cópias e exposição.

A implementação `age` deve usar biblioteca revisada ou ferramenta externa compatível com ARM. Não se implementa o formato ou as primitivas manualmente. A escolha concreta fica aberta até medir dependência, streaming e suporte no release.

---

## 6. Fluxo de exportação

1. A pessoa escolhe "Fazer backup da minha casa".
2. A tela informa quantidade de arquivos e tamanho estimado, sem listar dados alheios.
3. O sistema explica que o pacote ficará cifrado e que o kit será mostrado uma vez.
4. Um job cria o ponto de exportação, a lista autorizada e os arquivos em staging.
5. O sistema gera o manifesto, confere os hashes e produz o `tar.age` em streaming.
6. A pessoa baixa ou imprime o kit e confirma que o guardou. Não há redigitação nem aprovação de terceiros.
7. A pessoa baixa o pacote cifrado por rota autenticada.
8. O sistema apaga staging e o material pendente segundo a expiração.

Kit e pacote devem ser guardados separadamente. A UI explica que salvar os dois no mesmo lugar elimina o benefício da cifra.

O download do pacote exige a mesma conta que pediu o export. Não existe URL pública permanente. O artefato temporário tem expiração curta e pode ser apagado manualmente. Se expirar, a pessoa gera um novo backup e recebe um novo kit.

Exports grandes rodam fora do request LiveView, com progresso e retry. O trabalho de IO e cifra precisa ser limitado por buffers, não pelo tamanho total da `casa`.

---

## 7. Abrir no computador

A interoperabilidade mínima é:

1. instalar ou usar uma implementação `age` disponível para o sistema;
2. decifrar com a identity do kit;
3. extrair o `tar` com uma ferramenta comum;
4. abrir `files/` diretamente.

O formato não exige uma aplicação Tainá no computador. A documentação oferece comandos de exemplo, mas o fluxo principal da UI continua sendo baixar e guardar, sem exigir terminal.

`README.txt` dentro do pacote explica a estrutura. O runbook futuro pode incluir uma pequena interface gráfica de abertura se o drill mostrar que `age` e `tar` são uma barreira relevante, mas isso não muda o formato.

---

## 8. Importação em uma Tekoa

A importação aceita um `tar.age` e o kit correspondente. A pessoa autenticada escolhe uma pasta de destino na própria `casa`.

O fluxo:

1. validar tamanho e tipo do arquivo antes do upload completo;
2. enviar para staging temporário;
3. decifrar com o kit em memória de trabalho;
4. verificar a estrutura segura do `tar` e o `format` do manifesto;
5. conferir caminhos, tamanhos, hashes, MIME e cota;
6. mostrar um resumo da importação;
7. criar novas pastas e arquivos pertencentes à pessoa autenticada;
8. registrar os novos uploads normalmente na cadeia da Tekoa de destino;
9. apagar staging e o kit temporário.

A importação nunca reutiliza ids internos, `rkey`, actor ou entradas antigas do mural. Os novos registros pertencem a quem importa, ainda que o username de origem seja diferente. Datas originais podem permanecer no manifesto e nos metadados permitidos; os fatos da cadeia acontecem no momento da importação.

O primeiro corte sempre cria uma nova raiz, por exemplo "Backup importado de 31/10/2026". Não há merge, replace ou propagação de exclusões. Importar duas vezes cria duas cópias, e a UI avisa antes.

Uma falha de validação não cria nenhum registro. Uma falha durante a escrita usa a mesma transação/compensação de uploads existentes para não deixar manifesto e blobs parcialmente relacionados.

---

## 9. Mural, logs e privacidade

Backup pessoal **não** gera notícia no mural. Outros moradores e a zeladoria não recebem um evento social dizendo que alguém exportou a própria `casa`.

Isso não elimina observabilidade operacional. Logs locais podem registrar `export_id`, estágio, bytes, duração e razão técnica de falha para diagnóstico. Eles não contêm kit, identity, conteúdo, caminhos ou nomes de arquivos. A retenção desses logs segue a política soberana da caixa.

O backup da instância segue a regra oposta, definida na RFC 007: cada conclusão vira notícia no mural porque a operação da infraestrutura é coletiva. A distinção é intencional:

- operação da comunidade: transparente para a comunidade;
- portabilidade dos dados pessoais: privada da pessoa.

Nenhum dos dois fluxos envia telemetria para fora da caixa.

---

## 10. Fronteira técnica

### 10.1 Núcleo puro

Um módulo `Ypy.PersonalBackup` pode conter:

- tipos e validação do manifesto `taina-casa-v1`;
- cálculo e verificação de hashes;
- normalização e validação de caminhos;
- montagem pura do plano de exportação a partir de dados autorizados;
- plano de importação e detecção de erros;
- limites e invariantes de segurança do pacote.

O núcleo não consulta Repo, não abre arquivos e não conhece `age` como processo externo. IO, streaming e integração `age` ficam na casca.

### 10.2 Produto

`Taina.Ybira.PersonalBackup` ou módulo equivalente:

- resolve a seleção por `current_scope`;
- cria o ponto consistente de exportação;
- fixa os blobs necessários contra purga durante o trabalho;
- escreve o `tar` e chama a implementação `age` em streaming;
- guarda o artefato temporário com acesso exclusivo do dono;
- executa importação em staging e chama os uploads normais.

Um worker Oban cuida de exports e imports longos. Artefatos e staging têm expiração e limpeza automática. Um export pessoal não reutiliza o destino nem as chaves do cofre comunitário.

---

## 11. UI e Penpot

Boards mínimos:

1. **Iniciar backup:** escopo, tamanho estimado, explicação e botão.
2. **Progresso:** preparação, cifra, conclusão, retry e expiração.
3. **Kit individual:** baixar/imprimir, guardar separado e confirmar.
4. **Download:** pacote pronto, expiração e apagar.
5. **Importar:** upload, kit, validação e resumo.
6. **Resultado:** pasta criada, arquivos importados e erros sem escrita parcial.

Estados necessários: `casa` vazia, backup grande, falha de disco, kit perdido, pacote adulterado, manifesto incompatível, cota insuficiente e importação repetida.

A copy usa "backup da minha casa" e "kit para abrir o backup". Não usa "cofre", "identity", "recipient", "Shamir" ou nomes internos de subsistemas.

---

## 12. Sequência de build

1. Manifesto e validadores puros, com golden tests e path safety.
2. Seleção autorizada e plano de exportação por escopo.
3. Geração de `tar` em streaming e integração `age`, com kit individual.
4. Worker, staging, download autenticado, expiração e limpeza.
5. Abertura no computador documentada e drill com pessoa não técnica.
6. Importação não destrutiva em nova pasta, com validação completa antes da escrita.
7. Penpot e LiveViews dos dois fluxos.

Portão do vertical: uma pessoa exporta, abre os arquivos no computador sem o Tainá e importa o pacote em outra Tekoa usando somente pacote e kit.

---

## 13. Testes

- export contém somente arquivos da pessoa em `casa`;
- concessão de leitura sobre arquivo alheio não o inclui;
- `praça`, lixeira, miniaturas e dados de conta ficam de fora;
- blobs compartilhados por deduplicação não revelam outras referências;
- pacote abre com implementação `age` independente;
- kit errado ou de outro export falha fechada;
- adulteração de `tar`, manifesto ou arquivo é detectada;
- nomes com Unicode, espaços e caracteres especiais fazem round-trip seguro;
- path traversal, links e caminhos absolutos são rejeitados;
- hash, tamanho ou MIME divergente impede importação;
- export grande usa memória limitada;
- download por outra conta é negado;
- expiração apaga pacote e material pendente;
- importação cria novos registros na `casa` de quem importa;
- importação não sobrescreve nem reutiliza histórico social;
- nenhum backup pessoal gera notícia no mural;
- logs não contêm kit, paths ou conteúdo.

`mix precommit` e o drill de abertura/importação são os portões de saída.

---

## 14. Não objetivos, riscos e questões abertas

### 14.1 Não objetivos do primeiro corte

- backup pessoal agendado ou incremental;
- envio direto para Google Drive, iCloud ou S3;
- inclusão da `praça` ou da lixeira;
- merge, replace ou propagação de exclusões;
- restore de conta, permissões ou histórico do mural;
- múltiplos custodiantes e recuperação `k-de-n`;
- proteção contra root na caixa.

### 14.2 Riscos

1. **Kit perdido torna o pacote ilegível.** Mitigação: kit simples, download/impressão e possibilidade de gerar novo export enquanto a origem existe.
2. **Kit e pacote guardados juntos anulam a separação esperada.** Mitigação: UX explícita, sem impedir a escolha da pessoa.
3. **Exports grandes consomem disco temporário.** Mitigação: preflight, streaming, limite e limpeza.
4. **Abrir `age` e `tar` pode ser difícil para parte do público.** Mitigação: guia e drill; avaliar interface gráfica somente se necessário.
5. **Importar dados não recupera o contexto social.** Mitigação: copy clara e importação como novos arquivos.

### 14.3 Questões abertas

- qual implementação `age` entra no release sem comprometer ARM, streaming e tamanho da imagem;
- qual tamanho máximo de export/import por job e por cota;
- qual expiração padrão equilibra download em conexão lenta e uso de disco;
- se uma interface gráfica de abertura será necessária depois do drill;
- quando incluir opcionalmente as próprias contribuições na `praça`, sem transformar isso em export da praça inteira.
