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
Agente que mantém o **Currículo**: reconfere o **Conteúdo volátil** antes de cada módulo e propõe referências novas.
_Avoid_: agente de conteúdo, pesquisador, criador de dinâmicas

**Agente validador**:
Subagente, independente de quem escreveu o conteúdo, que verifica se o conteúdo proposto ressoa com as referências e se seus exercícios se aplicam a tecnologias e ferramentas reais.
_Avoid_: revisor, auditor

**Triagem**:
Avaliação diagnóstica feita ao entrar na trilha. Estima o nível inicial de cada **Competência**, define o módulo de partida e confere os **Pré-requisitos**.
_Avoid_: prova, teste de entrada, nivelamento

**Pré-requisito**:
Habilidade que a trilha assume e não ensina: terminal, git, Docker, TypeScript/Node e HTTP básico. A **Triagem** confere cada uma; se faltar, o App indica a documentação oficial.
_Avoid_: Nivelamento, Módulo 0

**Responsável pelo Currículo**:
Pessoa que aprova o **Currículo**: revisa Unidades, decide quais fontes viram **Referências primárias** e aprova os pull requests dos agentes. É um papel distinto do aprendiz.
_Avoid_: admin, dono, aprendiz (quando se trata de aprovar conteúdo)

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
Fonte da lista de validação definida pelo **Responsável pelo Currículo** (Primer, Fowler, Newman, Google SRE, AWS Well-Architected e afins). A obra do Kleppmann não entra: é a **Lente**.
_Avoid_: fonte oficial, bibliografia

**Lente**:
Conjunto de perguntas-guia, tiradas do DDIA e dos artigos do Kleppmann, que orientam o aprofundamento de cada **Unidade**. Formula perguntas e nunca conta como prova.
_Avoid_: referência, fonte de validação

**Mapa de competências**:
Estrutura da trilha derivada do Hello Interview e escrita com nossas palavras: os módulos, suas **Competências** e **Unidades**, o formato dos exercícios e as expectativas por nível.
_Avoid_: grade, índice do Hello Interview

**Currículo**:
Todo o conteúdo da trilha organizado por módulo: o **Mapa de competências** mais o conteúdo de cada **Unidade**. Construído antes de a trilha começar.
_Avoid_: grade, material, curso

**Competência**:
Capacidade observável de um módulo (ex.: `M04.C2`), medida na escala 0–4.
_Avoid_: habilidade, skill, objetivo

**Unidade**:
Parte de um módulo com objetivo próprio e identificador estável (ex.: `M04.U3`). Reúne conceitos, trade-offs, **Sementes de exercício**, leituras e fontes.
_Avoid_: aula, lição, tópico

**Semente de exercício**:
Material de uma **Unidade** a partir do qual os agentes geram Pílulas e revisões: decisões com opções ruim, boa e ótima, armadilhas, perguntas de verificação e critérios de rubrica.
_Avoid_: exercício pronto, questão

**Conteúdo volátil**:
Parte de uma **Unidade** que envelhece (versões, cotas gratuitas, preços, links) e registra a data da última conferência.
_Avoid_: dados de ferramenta

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

- A **Triagem** estima o nível inicial (0–2) de cada **Competência** e define o módulo de partida: o primeiro do núcleo com alguma **Competência** abaixo de 2
- Quem começa adiante no núcleo parte do **Checkpoint de módulo** de referência do módulo anterior; os módulos pulados só ficam concluídos com **Prova de antecipação**
- Cada módulo usa o **Projeto prático** a partir de exatamente um **Checkpoint de módulo**, e zero ou mais **Labs isolados**
- O **Agente tutor** apenas ajusta ordem, repetições, rigor e variações de exercícios sobre o **Núcleo validado**; nunca cria conteúdo fora dele
- O **Agente tutor** pode dispensar Pílulas e leituras, mas nunca o critério de **Módulo concluído**; acelerar exige **Prova de antecipação**
- O **Currículo** é construído antes de a trilha começar; o **Agente curador** só o mantém, e o **Agente tutor** nunca produz conteúdo
- A **Probabilidade de domínio** só sustenta níveis de competência 0–2; níveis 3–4 exigem evidência avaliada por rubrica
- Toda **Unidade** nova ou alterada passa pelo **Agente validador** antes de entrar no **Núcleo validado**
- O **Agente curador** propõe novas referências como **Referências secundárias**; só o **Responsável pelo Currículo** as promove a **Referências primárias**
- **Referências secundárias** não contam para a convergência do **Núcleo validado**
- Cada **Unidade** nasce no **Mapa de competências** e é validada por **Referências primárias**; o **Mapa de competências** define a estrutura, nunca a validação
- Cada **Unidade** serve a uma ou mais **Competências** do seu módulo; o **Módulo concluído** exige nível ≥3 em todas as **Competências** do módulo
- A **Lente** formula as perguntas de uma unidade; as respostas precisam de ≥2 **Referências primárias** de outros autores
- O **Agente tutor** agenda revisões espaçadas a partir do **Núcleo validado**, não do conhecimento próprio do modelo

## Flagged ambiguities

- "concluído" exigia repetir o cenário 2–4 semanas depois, o que conflitava com a previsibilidade por módulo — resolvido: a revisão tardia não faz parte do critério de **Módulo concluído**; ela alimenta o **Nível 4**.
- "Projeto de estudo" colidia com **App de estudos** — resolvido: o sistema construído nos labs é o **Projeto prático**.
- "aplicação" era usada tanto para o **App de estudos** quanto para o **Projeto prático** — resolvido: são coisas distintas; o aprendiz usa um e constrói o outro.
