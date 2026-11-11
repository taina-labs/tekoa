# Documentação técnica

A leitura atual cabe em dois documentos:

- [Roadmap](../ROADMAP.md): o que existe, o que falta para o 1.0.0, o que vem depois e o que é só ideia.
- [Arquitetura atual](ARQUITETURA.md): decisões vigentes e regras técnicas do espelho do rolo e do backup externo.

O código, os testes e o OpenAPI respondem pelos detalhes de implementação. As RFCs antigas estão no [arquivo](arquivo/INDICE.md) para preservar contexto, não para serem lidas antes de cada tarefa.

## Como escrever sem criar outra sopa

Mudança pequena ou já decidida: edite o roadmap ou a arquitetura. Não crie documento novo.

Uma nota técnica separada só é útil quando uma decisão ativa envolve vários contextos ou um trade-off que não cabe nesses dois documentos. Comece com uma página, usando esta forma:

```markdown
# Nome do problema

**Status:** proposta ou vigente.

## Problema
O que precisa mudar e para quem.

## Decisão
O que será feito e os limites explícitos.

## Consequências
O que isso facilita, custa ou torna impossível.

## Validação
Como saberemos que funciona.

## Revisar quando
O fato que faria esta decisão deixar de servir.
```

Regras práticas:

- Escreva para alguém que precisa decidir ou implementar, não para demonstrar completude.
- Alternativas entram somente quando mudam a decisão. Não registre toda a pesquisa.
- Proposta não parece produto pronto. Roadmap separa código existente, trabalho aceito e ideia.
- Ao aceitar uma proposta, leve a decisão para a arquitetura e o trabalho para o roadmap. Arquive a discussão longa.
- Atualize o documento atual em vez de empilhar emendas contraditórias.
- Nada de numeração de RFC por default, histórico de versões no cabeçalho ou seções obrigatórias de riscos, métricas e plano de fases.
