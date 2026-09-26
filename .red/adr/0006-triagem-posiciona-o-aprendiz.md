# Trilha para qualquer aprendiz: a Triagem posiciona e não há Nivelamento

O App de estudos atende qualquer aprendiz, não só quem o idealizou. Por isso a trilha não tem mais Nivelamento: a Triagem, feita ao entrar, estima o nível (0–2) de cada Competência e define por onde o aprendiz começa. O módulo de partida é o primeiro do núcleo com alguma Competência abaixo de 2. Terminal, git, Docker, TypeScript/Node e HTTP básico passam a ser Pré-requisitos: a Triagem confere e, se algum faltar, o App indica a documentação oficial. O Currículo ensina system design, não ferramentas.

O Nivelamento tinha sido desenhado a partir do perfil de uma pessoa e misturava o ensino de ferramentas com o de system design. Também carregava uma tarefa de infraestrutura do próprio App: a migração para uma VM.

## Considered Options

- **Nivelamento condicional antes do M01** (decisão anterior): cobria lacunas de ferramentas dentro da trilha, mas aumentava o Currículo com conteúdo fora do tema e refletia o perfil de um único aprendiz.

## Consequences

- A Probabilidade de domínio só sustenta os níveis 0–2, então a Triagem não conclui módulos. Ela posiciona o aprendiz, e os módulos anteriores ao de partida ficam disponíveis para Prova de antecipação.
- O Agente tutor pode dispensar Pílulas e leituras das Unidades cujas Competências a Triagem já estimou em nível 2.
- Quem começa adiante no núcleo parte do Checkpoint de módulo do anterior. Por isso o Currículo precisa de um Checkpoint de referência do Projeto prático para cada módulo do núcleo.
- Quem aprende e quem cuida do Currículo passam a ser papéis distintos. Promover Referências primárias e revisar Unidades é tarefa do Responsável pelo Currículo, não do aprendiz.
- A migração do App e do Hermes para uma VM sai do Currículo e vira tarefa de infraestrutura do App.
- Decisões tomadas pensando num único aprendiz (o estado em SQLite do ADR 0002, o ambiente no Mac com Tailscale e o uso de uma assinatura pessoal do Codex no ADR 0001) precisam ser revistas à parte.
