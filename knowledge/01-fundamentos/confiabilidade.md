---
titulo: Confiabilidade
fontes: [DDIA1]
status-validacao: base primária única (DDIA1); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Confiabilidade

> Um sistema confiável continua fazendo a coisa certa, no desempenho esperado, mesmo quando partes dele falham ou quando pessoas erram. Não se trata de evitar toda falha, e sim de impedir que falhas locais virem indisponibilidade para o usuário.

## Por que importa

Quase todo sistema real guarda algo que alguém não pode perder: pedidos, fotos, histórico financeiro, progresso de estudo. A confiabilidade é o primeiro requisito não funcional a ser negociado. Às vezes é aceitável abrir mão dela (protótipo, mercado não validado), mas essa deve ser uma decisão consciente, nunca um acidente.

## Conceitos-chave

- **Falha de componente × falha do sistema.** O DDIA separa *fault* (um componente se desvia da sua especificação) de *failure* (o sistema como um todo deixa de entregar o serviço). A meta de engenharia é tolerância a falhas: aceitar que componentes quebram e impedir que isso vire indisponibilidade.
- **Tolerância é sempre para tipos específicos de falha.** Não existe sistema "à prova de tudo". Projetar confiabilidade começa por listar quais falhas o sistema precisa suportar: perder um disco, uma máquina, uma zona, um datacenter inteiro.
- **Injetar falhas de propósito.** Como muitos bugs graves vivem no código de tratamento de erro, provocar falhas deliberadamente (matar processos, derrubar nós) mantém esse código exercitado. O exemplo clássico é o Chaos Monkey da Netflix.
- **Prevenir quando não há cura.** Em segurança, um vazamento não pode ser desfeito. Nesses casos, prevenir é melhor do que tolerar.

### Três famílias de falha

| Família | Característica | Resposta típica |
|---|---|---|
| **Hardware** (disco, RAM, energia, rede) | Aleatória e pouco correlacionada. Em frotas grandes, é rotina: com milhares de discos, algum quebra todo dia. | Redundância de componentes (RAID, fontes duplas) e, cada vez mais, tolerância à perda de máquinas inteiras por software. |
| **Software** (bugs sistemáticos) | Correlacionada: o mesmo bug derruba todas as instâncias ao mesmo tempo. Exemplos: entrada malformada, processo que esgota um recurso compartilhado, dependência lenta, falhas em cascata. | Não há solução rápida. Revisar premissas, testar, isolar processos, deixar processos caírem e reiniciarem, monitorar em produção e ter autochecagens de invariantes (ex.: mensagens que entram = mensagens que saem). |
| **Humana** (operação, configuração) | Estudos citados no livro apontam erro de configuração de operadores como causa líder de indisponibilidade; hardware aparece em apenas 10–25% dos casos. | Interfaces que facilitam o certo; ambientes sandbox com dados realistas; testes em todos os níveis; recuperação rápida (rollback, deploy gradual, ferramentas para recomputar dados); telemetria detalhada. |

- **Por que tolerar a perda de máquinas.** Em nuvem, máquinas virtuais podem sumir sem aviso. Um sistema que tolera isso também ganha uma vantagem operacional: dá para aplicar patches nó a nó (*rolling upgrade*) sem parar tudo.

## Trade-offs e decisões

- **Custo × confiabilidade.** Cada nível adicional de tolerância (mais réplicas, mais zonas, mais regiões) custa dinheiro e complexidade. A pergunta útil não é "é confiável?", e sim "contra quais falhas, com que custo, e qual o impacto de não tolerá-las?".
- **Redundância de hardware × tolerância por software.** Redundância de hardware é simples e mantém uma máquina viva por anos, mas não cobre a perda da máquina inteira. Tolerância por software (replicação, failover) cobre isso, ao preço de sistemas distribuídos (ver [replicação](../03-sistemas-distribuidos/replicacao.md)).
- **Restringir interfaces × contornos.** Interfaces restritivas demais levam as pessoas a contorná-las, o que anula o benefício. O equilíbrio é delicado.

## Aplicação no mundo real

- **Liste os modos de falha antes de escolher a arquitetura:** "o que acontece se o banco cair?", "se a fila encher?", "se o deploy tiver um bug?". Cada resposta vira um requisito testável.
- **Chaos engineering em escala pequena:** em um ambiente local com Docker Compose, derrubar um contêiner (`docker kill`) durante um teste de carga já exercita retries, timeouts e reconexões. Ferramentas como o Chaos Monkey e os experimentos de falha de provedores de nuvem levam isso a produção.
- **Autochecagem de invariantes em produção:** jobs periódicos que conferem regras de negócio (ex.: soma de saldos, contagem de mensagens) pegam bugs sistemáticos que nenhum alerta de CPU pegaria.
- **Recuperação de erro humano:** feature flags, deploy canário ou progressivo, rollback em um comando e backups *testados* (restaurar de verdade, periodicamente).
- *Ferramentas citadas aqui são indicativas; confirme detalhes na documentação oficial antes de usá-las em exercício.*

## No Projeto prático (encurtador de links)

- **Falha de hardware:** o que acontece com os redirecionamentos se o único banco cair? Esse é o gancho para réplicas de leitura.
- **Falha de software:** uma URL de destino malformada pode derrubar todas as instâncias? Validar entrada e isolar o processamento.
- **Falha humana:** uma migração de esquema errada apaga os links. Isso exige backup restaurável e migração reversível.
- **Invariante autochecável:** todo código curto aponta para exatamente um destino, e nenhum código é reutilizado.

## Armadilhas comuns

- Tratar "confiável" como propriedade binária, sem dizer contra quais falhas.
- Ter backup e nunca ter restaurado.
- Monitorar só recursos da máquina (CPU, memória) e não invariantes de negócio.
- Achar que falhas de software são independentes entre nós. Elas costumam ser correlacionadas.

## Perguntas de verificação

1. Qual a diferença entre falha de componente e falha do sistema, e por que ela orienta o projeto?
2. Por que bugs de software tendem a causar mais indisponibilidade do que falhas de hardware?
3. Cite três práticas para reduzir o impacto de erro humano e explique qual risco cada uma ataca.
4. Em que situação faz sentido *aumentar* a taxa de falhas de um sistema de propósito?

## Referências

- [DDIA1, cap. 1, "Reliability", pp. 6–10]: falha de componente × falha do sistema, hardware, software, erro humano, importância da confiabilidade.
- [DDIA1, cap. 1, "Thinking About Data Systems", pp. 4–6]: sistemas de dados compostos e a responsabilidade de quem os integra.
