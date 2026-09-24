---
titulo: Manutenibilidade
fontes: [DDIA1]
status-validacao: base primária única (DDIA1); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Manutenibilidade

> A maior parte do custo de um software vem depois do lançamento: corrigir, operar, investigar, adaptar, pagar dívida técnica. Projetar para manutenibilidade é evitar criar o próprio "sistema legado". O DDIA divide o tema em três princípios: operabilidade, simplicidade e evolutividade.

## Por que importa

Sistemas vivem anos e passam por muitas mãos. Uma arquitetura que ninguém consegue operar, entender ou mudar com segurança é um risco de negócio, mesmo que hoje seja rápida e confiável.

## Conceitos-chave

### Operabilidade: facilitar a vida de quem opera

- Boa operação consegue contornar limitações de um software ruim. Um software bom não roda de forma confiável com operação ruim.
- **Responsabilidades típicas de operação:** monitorar a saúde e restaurar o serviço; investigar causas de falha e degradação; aplicar atualizações e patches de segurança; entender como sistemas afetam uns aos outros; planejar capacidade; padronizar deploy e configuração; executar manutenções complexas (ex.: migrar de plataforma); preservar o conhecimento organizacional quando pessoas saem.
- **O que um sistema pode fazer para ajudar:**
  - dar visibilidade do comportamento em tempo de execução (monitoramento);
  - suportar automação e integração com ferramentas padrão;
  - não depender de máquinas individuais, para permitir manutenção sem parar o todo;
  - ter documentação e um modelo operacional previsível ("se eu fizer X, acontece Y");
  - ter bons padrões que o administrador possa sobrescrever;
  - se autocorrigir quando fizer sentido, mas com controle manual disponível;
  - comportar-se de forma previsível, com o mínimo de surpresas.

### Simplicidade: gerenciar complexidade

- **Sintomas de complexidade:** explosão do espaço de estados, acoplamento forte, dependências emaranhadas, nomes inconsistentes, gambiarras de desempenho, casos especiais para contornar problemas de outro lugar. O extremo é a "grande bola de lama" (*big ball of mud*).
- **Complexidade essencial × acidental.** A complexidade é acidental quando não vem do problema que o usuário vê, e sim da implementação.
- **Abstração** é a principal ferramenta contra a complexidade acidental: esconde detalhes atrás de uma fachada simples e reutilizável. SQL, por exemplo, esconde estruturas em disco, concorrência e recuperação após falhas.
- Em sistemas distribuídos, ainda é difícil empacotar bons algoritmos em boas abstrações.

### Evolutividade: facilitar mudanças

- Requisitos sempre mudam: novos casos de uso, prioridades de negócio, regulação, crescimento que força mudança arquitetural.
- Práticas ágeis (TDD, refatoração) cuidam da mudança em escala local. A evolutividade leva a mesma preocupação para o nível do sistema de dados. Exemplo: como "refatorar" a timeline do Twitter do fan-out na leitura para o fan-out na escrita.
- A facilidade de mudar está diretamente ligada à simplicidade e à qualidade das abstrações.

## Trade-offs e decisões

- **Autocorreção × controle manual:** automação reduz trabalho repetitivo, mas quando falha de forma inesperada pode piorar incidentes. Ter sempre um modo manual.
- **Abstração × vazamento:** abstrações boas economizam esforço, mas toda abstração vaza em algum ponto, e quem opera precisa entender o que há por baixo.
- **Velocidade de entrega × dívida técnica:** atalhos aceleram hoje e cobram juros na manutenção. Registre-os de forma consciente (ex.: em ADRs).

## Aplicação no mundo real

- **Observabilidade desde o primeiro deploy:** logs estruturados, métricas (taxa, erros, duração) e traces (OpenTelemetry). É a base da operabilidade.
- **Infraestrutura como código e configuração versionada:** torna previsível o "se eu fizer X, acontece Y" e facilita revisar mudanças.
- **Registros de decisão (ADRs)** preservam o *porquê*, que é o conhecimento que mais se perde quando pessoas saem.
- **Migrações de esquema reversíveis e deploys graduais** tornam o sistema evolutivo na prática. Ver [codificação e evolução de esquema](../02-dados/codificacao-e-evolucao-de-esquema.md).

## No Projeto prático (encurtador de links)

- Desde a primeira versão: métricas RED (taxa, erros, duração) para redirecionamento e criação.
- Um README operacional curto: como subir, como verificar a saúde, como restaurar o backup.
- Cada mudança arquitetural (cache, fila, réplica) registrada com o porquê e a medição que a motivou.

## Armadilhas comuns

- Confundir simplicidade do sistema com simplicidade da interface de usuário.
- Automatizar sem deixar uma válvula manual.
- Deixar o conhecimento operacional só na cabeça de uma pessoa.
- Adicionar abstrações "para o futuro" que só aumentam a complexidade acidental.

## Perguntas de verificação

1. Diferencie complexidade essencial de acidental com um exemplo do seu dia a dia.
2. Cite quatro características que tornam um sistema de dados fácil de operar.
3. Por que evolutividade depende de simplicidade?

## Referências

- [DDIA1, cap. 1, "Maintainability", pp. 18–22]: operabilidade, simplicidade (complexidade acidental, abstração), evolutividade.
- [DDIA1, cap. 1, "Summary", pp. 22–23]: requisitos funcionais × não funcionais.
