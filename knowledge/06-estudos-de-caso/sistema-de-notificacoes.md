---
titulo: "Estudo de caso: sistema de notificações (push, SMS, e-mail)"
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) + princípios primários (DDIA1); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Estudo de caso: sistema de notificações

> Enviar milhões de notificações por dia (push iOS e Android, SMS, e-mail) através de provedores terceiros, com a regra de ouro "**pode atrasar ou chegar fora de ordem, mas não pode se perder**". O desenho clássico é serviço de notificações + **filas por canal** + workers + retry + deduplicação + preferências do usuário + rate limit + observabilidade.

## Por que importa

Quase todo produto envia notificações, e o envio depende de serviços externos que falham, limitam e cobram. O problema concentra padrões essenciais: desacoplamento por fila, entrega "pelo menos uma vez" com deduplicação, retry com backoff, isolamento de falhas por canal e respeito às preferências do usuário.

## Escopo do exercício (SDI1)

- **Canais:** push, SMS e e-mail.
- **Tempo real "suave":** o mais rápido possível, mas atraso sob carga é aceitável.
- **Dispositivos:** iOS, Android, desktop.
- **Disparos:** vindos de clientes ou agendados no servidor.
- **Opt-out:** permitido.
- **Volume:** 10 milhões de push, 1 milhão de SMS e 5 milhões de e-mails por dia.

## Como cada canal funciona

- **Push iOS:**
  - *provedor* (seu servidor) monta a requisição com o **token do dispositivo** e o **payload** JSON;
  - envia ao **APNs**, o serviço da Apple, que entrega ao dispositivo.
- **Push Android:** tipicamente o **FCM**, o serviço de mensagens do Firebase.
- **SMS:** provedores comerciais (ex.: Twilio, Vonage/Nexmo).
- **E-mail:** servidor próprio é possível, mas provedores comerciais (ex.: SendGrid, Mailchimp) entregam melhor e trazem analytics.
- **Disponibilidade regional:** um provedor pode não existir em certos mercados (o SDI1 cita o FCM indisponível na China). Projete provedores **plugáveis**.

## Coleta de contatos

- Na instalação ou no cadastro, a API grava e-mail e telefone (tabela de usuários) e **tokens de dispositivo** (tabela de dispositivos).
- Um usuário pode ter vários dispositivos.

## Evolução do desenho

**Inicial:** serviços 1..N chamam um servidor de notificações, que chama os provedores. Problemas:
- ponto único de falha;
- difícil de escalar componentes separadamente;
- gargalo de desempenho (montar conteúdo e esperar provedores lentos).

**Melhorado:**
- banco e cache fora do servidor;
- vários servidores de notificação com autoescala;
- **filas de mensagens para desacoplar**, com **uma fila por tipo de notificação**, para que a queda de um provedor não afete os outros canais.

**Fluxo:**
1. um serviço chama a API interna;
2. o servidor busca metadados (usuário, token, preferências) no cache ou no banco;
3. publica o evento na fila do canal;
4. os workers consomem;
5. os workers chamam o provedor;
6. o provedor entrega ao dispositivo.

**Servidores de notificação:** API **interna ou autenticada** (anti-spam), validações (e-mail, telefone), busca de dados para montar o conteúdo, publicação na fila.

## Aprofundamentos

- **Não perder dados:** persistir cada notificação em um **log de notificações** no banco + mecanismo de retry.
- **Exatamente uma vez?** **Não é possível garantir.** Na maior parte do tempo, a entrega acontece uma vez, mas a natureza distribuída gera duplicatas. Mitigação: **deduplicação por ID do evento** (já visto? descarta).
- **Templates:** formato consistente, menos erros, menos tempo.
- **Preferências:** tabela (usuário, canal, opt-in). **Checar antes de enviar.**
- **Rate limit por usuário:** evita sobrecarregar a pessoa, que acabaria desligando as notificações.
- **Retry:** em falha do provedor, a notificação volta para a fila. Se persistir, alerta aos desenvolvedores.
- **Segurança:** appKey e appSecret, só clientes autenticados podem enviar.
- **Monitorar a fila:** o tamanho da fila é a métrica-chave. Se cresce, faltam workers.
- **Rastreamento de eventos:** entregue, aberto, clicado, engajamento. Integração com analytics.
- **Desenho final:** autenticação, rate limit, retry, templates, monitoramento e rastreamento.

## Conexões com os fundamentos (DDIA1)

- **Entrega "pelo menos uma vez" + idempotência = efeito "uma vez".** O ID do evento deve ser gerado **na origem** e percorrer todo o caminho (argumento de ponta a ponta). Deduplicar só no worker não evita duplicatas causadas pelo serviço de origem que repetiu a chamada. Ver [integração de dados e correção](../04-dados-derivados/integracao-de-dados-e-correcao.md).
- **Efeitos externos não se desfazem:** enviar e-mail ou SMS é um efeito colateral que **não pode ser revertido** por abortar uma transação. Por isso: gravar a intenção (outbox ou log de notificações), enviar de forma assíncrona e deduplicar. Nunca envie de dentro de uma transação que pode abortar.
- **Fila tradicional × log:** o SDI1 usa filas por canal (balanceamento de carga entre workers), o que é adequado porque cada envio é independente e a ordem importa pouco. Reentrega pode **reordenar** mensagens, o que é aceitável pelos requisitos ("pode chegar fora de ordem"). Ver [processamento de fluxo](../04-dados-derivados/processamento-de-fluxo.md).
- **Causalidade sutil:** o exemplo do DDIA1 (desfazer amizade e depois mandar mensagem) mostra que um serviço de notificação pode processar eventos fora da ordem causal e **notificar quem não devia**. Checar preferências e permissões **no momento do envio**, com estado atualizado, reduz o risco.
- **Retry sob sobrecarga:** use backoff exponencial com jitter e limite de tentativas. Mensagens que falham sempre vão para uma **fila de mensagens mortas (DLQ)**, para não entupir a fila principal.

## Aplicação no mundo real

- **Filas gerenciadas** (SQS, Pub/Sub) ou RabbitMQ, com DLQ, *visibility timeout* e métricas de profundidade da fila e idade da mensagem mais antiga.
- **Chave de idempotência** também nos provedores que a suportam. Confirme na documentação de cada provedor.
- **Conformidade:** consentimento e opt-out obrigatórios (ex.: leis anti-spam, LGPD/GDPR, regras de SMS por país), link de descadastro em e-mails, horários permitidos.
- **Entregabilidade de e-mail:** configuração de SPF, DKIM e DMARC no domínio; reputação do IP de envio.
- **Tokens de push expiram ou são revogados:** trate as respostas de "token inválido" do APNs e do FCM removendo o token.

## No Projeto prático (encurtador de links)

- **Notificação de marco:** "seu link atingiu 1.000 cliques" por e-mail. Gerada pelo agregador de cliques, com evento na fila, worker de e-mail, deduplicação por (link, marco) e respeito às preferências.
- **Exercício:** derrubar o "provedor" (simulado) e verificar que as notificações aguardam e são reenviadas, sem duplicar e sem afetar o redirecionamento.

## Armadilhas comuns

- Enviar e-mail ou SMS de forma síncrona dentro da requisição do usuário.
- Uma fila única para todos os canais (um provedor lento afeta todos).
- Retry infinito sem DLQ.
- Não checar opt-out antes do envio.
- Deduplicação sem ID de ponta a ponta.

## Perguntas de verificação

1. Por que ter uma fila por canal?
2. Por que "exatamente uma vez" não é garantido, e como aproximar o efeito?
3. Onde deve ser gerado o ID usado na deduplicação? Por quê?
4. Qual métrica indica que faltam workers?
5. Por que nunca enviar notificação de dentro de uma transação que pode abortar?

## Referências

- [SDI1, cap. 10, "Design a Notification System"]: canais, coleta de contatos, desenho inicial e melhorado, filas por canal, workers, confiabilidade, deduplicação, templates, preferências, rate limit, retry, segurança, monitoramento e rastreamento.
- [DDIA1, cap. 7, "Handling errors and aborts", pp. 231–232]: efeitos externos em transações abortadas.
- [DDIA1, cap. 11, "Acknowledgments and redelivery", pp. 445–446].
- [DDIA1, cap. 12, "Ordering events to capture causality" e "The end-to-end argument", pp. 493–494, 516–520].
