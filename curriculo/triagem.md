---
id: triagem
status: esqueleto
---

# Triagem

**Pergunta:** qual é o seu nível em cada Competência da trilha, e por onde você começa?

## Objetivo

Posicionar o aprendiz no Mapa de competências antes da primeira Pílula: estimar o nível inicial de cada Competência, definir o módulo de partida e conferir os Pré-requisitos ([ADR 0006](../.red/adr/0006-triagem-posiciona-o-aprendiz.md)).

## O que mede

1. **As 45 Competências de M01 a M15**, listadas em cada `modulo.md`, só até o nível 2. A Probabilidade de domínio não sustenta os níveis 3 e 4, que exigem evidência avaliada por rubrica.
2. **Os Pré-requisitos** listados em [`mapa.md`](mapa.md#pré-requisitos).

## Como o resultado posiciona o aprendiz

- **Nível inicial:** a estimativa (0–2) de cada Competência vira a Probabilidade de domínio de partida.
- **Módulo de partida:** o primeiro módulo do núcleo (M01–M10) com alguma Competência estimada abaixo de 2. Se todas estiverem em 2, o aprendiz começa pelos aprofundamentos (M11–M14).
- **Projeto prático:** quem começa adiante recebe o Checkpoint de módulo de referência do módulo anterior ao de partida.
- **Módulos anteriores ao de partida:** ficam disponíveis para Prova de antecipação. Só ela os marca como concluídos.
- **Dentro de cada módulo:** o Agente tutor pode dispensar Pílulas e leituras das Unidades cujas Competências já estão em nível 2. O critério de Módulo concluído continua valendo.
- **Pré-requisito ausente:** o App avisa e indica a documentação oficial da ferramenta. O Currículo não ensina ferramentas.

Exemplo:

- **Entrada:** Competências de M01 a M03 estimadas em 2; `M04.C1` em 2; `M04.C2` e `M04.C3` em 1; Docker não conferido.
- **Saída:** começa no M04, a partir do Checkpoint de referência `M03`. M01–M03 ficam disponíveis para Prova de antecipação. No M04, as Pílulas de `M04.U1` e `M04.U6` (que medem `M04.C1`) podem ser dispensadas. O App indica a documentação oficial do Docker antes do primeiro lab.

## A construir na passada 2

- Um banco de itens diagnósticos: 2 a 3 itens por Competência, nos níveis 1 e 2, com a resposta esperada.
- Tarefas curtas e verificáveis para cada Pré-requisito (ex.: "suba este `compose.yaml` e informe a porta do serviço").
- Os Checkpoints de módulo de referência do Projeto prático, de `M01` a `M10`.

A ordem e a quantidade de itens aplicados a cada pessoa (a Triagem não deve aplicar o banco inteiro) ficam para as ADRs de dinâmica.
