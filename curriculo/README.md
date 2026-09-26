# Currículo da Trilha de System Design

Esta pasta guarda o **Currículo**: o Mapa de competências mais o conteúdo de cada Unidade. É daqui que saem as Pílulas, as leituras, os labs e as revisões. Os termos em negrito seguem o glossário em [`.red/CONTEXT.md`](../.red/CONTEXT.md). Em caso de dúvida sobre um termo, o glossário vale mais que este arquivo.

## Organização

```
curriculo/
  README.md            este arquivo
  lente.md             princípios da trilha e perguntas da Lente (L01–L12)
  mapa.md              Mapa de competências: estações, ordem, dependências e objetivos
  _modelo-unidade.md   formato de uma Unidade completa
  preparacao/          Triagem e Nivelamento
  MNN-slug/
    modulo.md          objetivo, Competências, Unidades, lab, transferência e fontes de partida
    UN-slug.md         uma Unidade completa (criada na passada 2)
```

## Identificadores

- Módulo: `M01` a `M15`. Competência: `M04.C2`. Unidade: `M04.U3`. Pergunta da Lente: `L04`.
- Um identificador nunca muda de significado. Se uma Unidade sair do Currículo, o identificador fica aposentado e não é reaproveitado.
- Referencie sempre pelo identificador, por exemplo: "a Pílula treina `M04.U3` e mede `M04.C2`".

## Status

Todo `modulo.md` e toda Unidade têm um campo `status` no cabeçalho:

| Status | Significa | Quem pode usar |
|---|---|---|
| `esqueleto` | Objetivo e posição no Mapa definidos, sem conteúdo | Quem compõe conteúdo |
| `rascunho` | Conteúdo escrito, aguardando o Agente validador | Quem compõe e quem valida |
| `validado` | Conferido pelo Agente validador e revisado pelo aprendiz | Todos, inclusive o Agente tutor |

O Agente tutor só gera Pílulas e revisões a partir de Unidades `validado`.

## O que fazer, conforme o seu papel

- **Compor ou completar uma Unidade:** leia [`_modelo-unidade.md`](_modelo-unidade.md), o `modulo.md` do módulo (seção "Fontes de partida") e as perguntas da Lente que a Unidade lista em [`lente.md`](lente.md). Escreva uma síntese própria, com fontes citadas: siga a política de fontes em [`knowledge/README.md`](../knowledge/README.md).
- **Validar uma Unidade:** confira a regra do **Núcleo validado** no glossário e a lista abaixo. Devolva a Unidade com o motivo quando um item falhar.
  1. Cada conceito e trade-off tem ≥2 **Referências primárias** de autores diferentes, e nenhuma é do Kleppmann ([ADR 0004](../.red/adr/0004-ddia-como-lente-e-nao-como-prova.md)).
  2. As divergências entre fontes aparecem como trade-off.
  3. Os fatos de ferramenta citam documentação oficial e têm data em "Conteúdo volátil".
  4. O lab e os exercícios rodam em ferramentas reais e gratuitas.
  5. As perguntas da Lente listadas na Unidade estão respondidas.
  6. Nenhum trecho foi copiado do Hello Interview, do SDI1 ou do DDIA. Texto do Primer só entra adaptado e com crédito (CC BY 4.0).
- **Gerar Pílulas ou revisões:** parta das Sementes de exercício de uma Unidade `validado`. Use a pergunta da Lente para mostrar a consequência de cada opção. Registre qual Competência cada Pílula mede.
- **Manter o Currículo (Agente curador):** antes de o aprendiz começar um módulo, reconfira cada item de "Conteúdo volátil" das Unidades desse módulo na documentação oficial e atualize a data. Mudança de conceito ou de trade-off volta para `rascunho` e passa de novo pelo validador.

## Por que o Currículo tem este formato

- O Hello Interview dá a estrutura; Primer, SDI1 e DDIA complementam: [ADR 0003](../.red/adr/0003-hello-interview-como-estrutura-base.md).
- O DDIA é Lente, não prova: [ADR 0004](../.red/adr/0004-ddia-como-lente-e-nao-como-prova.md).
- O Currículo é construído antes, e o curador só o mantém: [ADR 0005](../.red/adr/0005-curriculo-construido-antes-da-trilha.md).
- A síntese por tema dos livros fica em [`knowledge/`](../knowledge/README.md); as Unidades apontam para lá.
