---
titulo: Método de design de sistemas (do requisito à decisão)
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) para o roteiro; princípios de requisitos do DDIA1; aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Método de design de sistemas

> Um bom projeto começa pelas **perguntas certas**, não pela solução. O SDI1 propõe um roteiro de quatro etapas pensado para entrevistas: entender o problema e o escopo, propor um desenho de alto nível, aprofundar os pontos críticos e fechar com gargalos e melhorias. O mesmo roteiro serve para decisões reais se for complementado com **medição e registro de decisão**.

## Por que importa

A maioria dos projetos ruins resolve com excelência o problema errado. Um roteiro explícito força a esclarecer requisitos, explicitar premissas, comparar alternativas e reconhecer limites. Isso vale para entrevistas, para documentos de design (RFCs) e para as dinâmicas desta trilha.

## Conceitos-chave

### O que se avalia (e o que se evita)

- Em entrevistas, a avaliação cobre **colaboração, trabalho sob pressão, lidar com ambiguidade e fazer boas perguntas**, além da técnica.
- **Sinais de alerta:** **excesso de engenharia** (pureza de design ignorando trade-offs e custos acumulados), mente fechada e teimosia.

### As quatro etapas (SDI1)

**1. Entender o problema e definir o escopo** (~3–10 min de 45)
- **Não responda rápido.** O livro usa o personagem "Jimmy", o aluno que responde antes de pensar, como exemplo do que não fazer.
- Pergunte: quais funcionalidades? quantos usuários? que ritmo de crescimento (3, 6, 12 meses)? que stack e serviços já existem?
- Quando não houver resposta, **assuma e anote**.
- Exemplo (feed de notícias): web e mobile? quais funcionalidades principais? ordem cronológica ou ranqueada? quantos amigos? qual volume (DAU)? tem mídia?

**2. Propor desenho de alto nível e obter concordância** (~10–15 min)
- Rascunho inicial com diagrama de caixas (clientes, APIs, servidores, bancos, cache, CDN, filas).
- Trate o entrevistador (ou o time) como colega e peça feedback.
- **Estimativas** para checar se o desenho cabe na escala (ver [estimativas de capacidade](estimativas-de-capacidade.md)).
- Percorra **casos de uso concretos**, que revelam casos de borda.
- API e esquema de dados só se a escala do problema pedir: em "projete o Google", não; em "backend de pôquer", sim.

**3. Aprofundar** (~10–25 min)
- Priorize com o interlocutor os componentes críticos: a função de hash em um encurtador, latência e presença em um chat.
- **Gestão de tempo:** não se perca em detalhes que não demonstram capacidade (ex.: algoritmo de ranqueamento em detalhe).

**4. Fechar** (~3–5 min)
- Gargalos e melhorias: **nunca diga que o desenho é perfeito**.
- Recapitule.
- Casos de erro (queda de servidor, perda de rede).
- Operação: métricas, logs, deploy.
- A próxima curva de escala (de 1 para 10 milhões de usuários).
- Refinamentos que faria com mais tempo.

### Fazer e não fazer

- **Faça:** pedir esclarecimentos; entender os requisitos (não existe resposta certa, e a solução de uma startup difere da de uma empresa grande); pensar em voz alta; propor várias abordagens; detalhar primeiro os componentes mais críticos; nunca desistir.
- **Não faça:** chegar despreparado; pular para a solução; se aprofundar em um componente antes do panorama; ficar travado em silêncio; achar que terminou antes de o interlocutor dizer.

### Requisitos funcionais × não funcionais (DDIA1)

- **Funcionais:** o que o sistema faz (armazenar, buscar, processar).
- **Não funcionais:** confiabilidade, escalabilidade, manutenibilidade, segurança, conformidade, compatibilidade. Cada um deve virar **números e cenários**: parâmetros de carga, percentis de latência, SLOs, modos de falha tolerados. Ver [confiabilidade](confiabilidade.md), [escalabilidade e desempenho](escalabilidade-e-desempenho.md) e [manutenibilidade](manutenibilidade.md).
- **Premissas de carga definem a arquitetura.** Se estiverem erradas, o esforço de escala é desperdício ou prejuízo. Em produto não validado, iterar rápido vale mais do que escalar para uma carga hipotética.

## Do roteiro de entrevista ao projeto real

O SDI1 é explicitamente um guia de **entrevista**: números e desenhos são hipóteses didáticas. Para projetos reais, esta trilha acrescenta quatro elementos, alinhados à §1 do bootstrap:

1. **Requisitos mensuráveis:** SLOs com percentis, orçamento de erro e parâmetros de carga, em vez de "deve ser rápido".
2. **Hipótese → experimento → medição:** cada decisão relevante tem uma previsão ("o cache reduzirá o p99 de leitura para < X ms") testada com carga real ou sintética.
3. **Registro de decisão (ADR):** contexto, alternativas consideradas, decisão e consequências, para que o "porquê" sobreviva.
4. **Revisão:** revisitar a decisão quando a carga ou o requisito mudar ("que condição me faria mudar de ideia?", pergunta que a §5 do bootstrap pede ao fim de cada leitura).

## Trade-offs e decisões

- **Profundidade × amplitude:** primeiro o todo, depois os pontos críticos.
- **Simplicidade × escala futura:** desenhe para a próxima ordem de grandeza, não para a décima. O excesso de engenharia é um sinal de alerta explícito no SDI1 e um custo explícito no DDIA1.

## Aplicação no mundo real

- **Documento de design (RFC):** contexto e problema, requisitos (funcionais, não funcionais, fora de escopo), estimativas, alternativas com prós e contras, desenho escolhido, riscos, plano de rollout e observabilidade, questões abertas.
- **Revisão em par:** apresentar o desenho para alguém que faça as perguntas da etapa 4.
- **Entrevista:** mesma estrutura, dentro de 45–60 minutos.

## No Projeto prático (encurtador de links)

Aplique as quatro etapas antes de cada módulo:

1. escopo (ex.: "links expiram? alias personalizado? analytics em tempo real ou diário?");
2. desenho de alto nível com estimativa;
3. aprofundamento no componente do módulo (ex.: geração de código, cache);
4. fechamento com gargalos, métricas e próxima curva de escala.

Registre cada decisão como uma decisão arquitetural documentada, que vale 20 XP na §2 do bootstrap.

## Armadilhas comuns

- Começar pela tecnologia ("vamos usar Kafka") em vez do problema.
- Requisitos não funcionais sem números.
- Um único desenho, sem alternativas comparadas.
- Declarar o desenho "pronto" sem discutir falhas e operação.

## Perguntas de verificação

1. Que perguntas você faria nos primeiros 5 minutos de "projete um encurtador de links"?
2. Qual a diferença entre o objetivo da etapa 2 e o da etapa 3?
3. Por que excesso de engenharia é um sinal de alerta?
4. O que a trilha acrescenta ao roteiro de entrevista para torná-lo útil em projetos reais?

## Referências

- [SDI1, cap. 3, "A Framework for System Design Interviews"]: o que se avalia, as quatro etapas, fazer e não fazer, divisão do tempo.
- [DDIA1, cap. 1, "Summary" e "Approaches for Coping with Load", pp. 17–23]: requisitos funcionais × não funcionais, premissas de carga, produto não validado.
