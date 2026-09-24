# Trilha de System Design

Experiência de estudo gamificada para aprender a decidir arquiteturas de aplicações reais, partindo do zero, com entrevista de system design como objetivo secundário.

## Language

### Experiência

**App de estudos**:
Aplicação que o aprendiz abre para estudar, onde ficam as Pílulas, o progresso e o Agente tutor.
_Avoid_: aplicação (sem qualificador), app

**Pílula**:
Desafio curto disponível no **App de estudos**, sempre iniciado pelo aprendiz, nunca disparado pelo sistema.
_Avoid_: notificação, lembrete, lição disparada

**Agente tutor**:
Agente que avalia o desempenho do aprendiz e ajusta repetição, rigor e extensão das dinâmicas.
_Avoid_: agente adaptador, criador de dinâmicas, revisor

**Agente curador**:
Agente que pesquisa referências e estrutura o conteúdo do próximo módulo antes de o aprendiz chegar a ele.
_Avoid_: agente de conteúdo, pesquisador, criador de dinâmicas

**Agente validador**:
Subagente, independente do **Agente curador**, que verifica se o conteúdo proposto ressoa com as referências e se seus exercícios se aplicam a tecnologias e ferramentas reais.
_Avoid_: revisor, auditor

**Triagem**:
Avaliação diagnóstica de conceitos de system design, ferramentas (terminal, git, Docker) e fundamentos de infraestrutura (redes, Linux) feita antes do primeiro módulo.
_Avoid_: prova, teste de entrada

**Nivelamento**:
Conteúdo preparatório definido pela **Triagem** para cobrir lacunas antes do primeiro módulo.
_Avoid_: Módulo 0

### Progresso

**Módulo concluído**:
Estado de um módulo cujo critério de conclusão foi atingido ao fim do próprio módulo, sem depender de revisão tardia.
_Avoid_: Módulo consolidado, módulo dominado

**Prova de antecipação**:
Demonstração antecipada do critério de **Módulo concluído** que permite pular o restante de um módulo.
_Avoid_: pular módulo, dispensa

**Probabilidade de domínio**:
Estimativa, por competência, de que o aprendiz domina aquele tema, atualizada a cada resposta na **Triagem** e nas **Pílulas**.
_Avoid_: nota, score

**Nível 4**:
Nível de competência demonstrado ao justificar de novo uma decisão 2–4 semanas depois; é métrica e Conquista, nunca requisito de conclusão.
_Avoid_: Consolidação

### Conteúdo

**Núcleo validado**:
Conteúdo da trilha cujos conceitos e trade-offs convergem em ≥2 **Referências primárias** independentes além do Primer (fatos de ferramenta: só documentação oficial), ou cuja divergência está exposta como trade-off.
_Avoid_: Conteúdo base, material oficial

**Referência primária**:
Fonte da lista de validação definida pelo aprendiz (Primer, Kleppmann, Fowler, Newman, Google SRE, AWS Well-Architected e afins).
_Avoid_: fonte oficial, bibliografia

**Referência secundária**:
Fonte usada para estruturar e comunicar cenários (Hello Interview, ByteByteGo), que não substitui medições.
_Avoid_: referência de apoio

### Laboratório

**Projeto prático**:
Encurtador de links com analytics de cliques, construído do zero pelo aprendiz e evoluído ao longo dos módulos.
_Avoid_: Projeto espinha, Projeto de estudo, projeto laboratório, app de exemplo

**Lab isolado**:
Exercício prático autocontido para um tema que não cabe no **Projeto prático**.
_Avoid_: Mini-projeto

**Checkpoint de módulo**:
Estado limpo e versionado do **Projeto prático** a partir do qual um módulo começa.
_Avoid_: Snapshot (no bootstrap, snapshot designa registros de medições, não estado do código)

## Relationships

- A **Triagem** define zero ou mais itens de **Nivelamento** antes do primeiro módulo
- Cada módulo usa o **Projeto prático** a partir de exatamente um **Checkpoint de módulo**, e zero ou mais **Labs isolados**
- O **Agente tutor** apenas ajusta ordem, repetições, rigor e variações de exercícios sobre o **Núcleo validado**; nunca cria conteúdo fora dele
- O **Agente tutor** pode dispensar Pílulas e leituras, mas nunca o critério de **Módulo concluído**; acelerar exige **Prova de antecipação**
- O **Agente curador** produz o conteúdo de um módulo à frente do progresso do aprendiz; o **Agente tutor** nunca produz conteúdo
- A **Probabilidade de domínio** só sustenta níveis de competência 0–2; níveis 3–4 exigem evidência avaliada por rubrica
- Todo conteúdo do **Agente curador** passa pelo **Agente validador** antes de entrar no **Núcleo validado**
- O **Agente curador** propõe novas referências como **Referências secundárias**; só o aprendiz as promove a **Referências primárias**
- **Referências secundárias** não contam para a convergência do **Núcleo validado**
- O **Agente tutor** agenda revisões espaçadas a partir do **Núcleo validado**, não do conhecimento próprio do modelo

## Flagged ambiguities

- "concluído" exigia repetir o cenário 2–4 semanas depois, o que conflitava com a previsibilidade por módulo — resolvido: a revisão tardia não faz parte do critério de **Módulo concluído**; ela alimenta o **Nível 4**.
- "Projeto de estudo" colidia com **App de estudos** — resolvido: o sistema construído nos labs é o **Projeto prático**.
- "aplicação" era usada tanto para o **App de estudos** quanto para o **Projeto prático** — resolvido: são coisas distintas; o aprendiz usa um e constrói o outro.
