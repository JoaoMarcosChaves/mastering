---
titulo: "Estudo de caso: sistema de chat (1:1, grupos, presença)"
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) + princípios primários (DDIA1); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Estudo de caso: sistema de chat

> Mensagens 1:1 e em grupos pequenos com baixa latência, indicador de presença (online/offline), vários dispositivos por conta e push quando o destinatário está offline. As decisões centrais são o **protocolo de comunicação servidor → cliente** (WebSocket), a separação entre **serviços sem estado e com estado** (conexões persistentes), o **armazenamento do histórico** e a **ordenação e sincronização** das mensagens.

## Por que importa

É o exemplo típico de sistema **com conexões persistentes**, que quebra o modelo requisição/resposta sem estado da maioria das aplicações web. Os mesmos padrões aparecem em notificações em tempo real, colaboração, dashboards ao vivo e jogos.

## Escopo do exercício (SDI1)

- 1:1 e grupos de até 100 pessoas.
- Web e mobile.
- 50 milhões de DAU.
- Apenas texto (< 100.000 caracteres).
- Presença online.
- Vários dispositivos.
- Push.
- Histórico guardado para sempre.
- Criptografia de ponta a ponta fora do escopo inicial.

## Comunicação cliente ↔ servidor

- **Envio:** HTTP com *keep-alive* funciona.
- **Recebimento** (o servidor precisa iniciar):
  - **polling:** o cliente pergunta periodicamente. Desperdiça recursos, porque a resposta é quase sempre "nada novo";
  - **long polling:** a conexão fica aberta até chegar mensagem ou estourar o timeout. Problemas: o servidor que recebe a mensagem pode não ser o que tem a conexão do destinatário; é difícil saber se o cliente desconectou; reconexões periódicas;
  - **WebSocket:** começa como HTTP e é "promovido" a uma conexão **bidirecional e persistente** (portas 80/443, atravessa firewalls). É a solução usual, e dá para usar nos dois sentidos.
- **O resto** (cadastro, login, perfil) continua HTTP comum.

## Arquitetura

- **Serviços sem estado** atrás de um balanceador: login, cadastro, perfil e **descoberta de serviço** (que indica ao cliente a qual servidor de chat conectar).
- **Serviço com estado:** os **servidores de chat**, porque cada cliente mantém uma conexão persistente com um servidor específico.
- **Integração externa:** push para quem está offline (ver [sistema de notificações](sistema-de-notificacoes.md)).
- **Escala:** o SDI1 estima que 1 milhão de conexões simultâneas a ~10 KB cada caberiam em ~10 GB de memória em um servidor. Mas um servidor único é ponto único de falha, então comece assim e distribua. Componentes:
  - servidores de chat;
  - servidores de **presença**;
  - servidores de API;
  - servidores de notificação;
  - **key-value store** para o histórico.

## Armazenamento

- **Dados genéricos** (perfis, configurações, lista de amigos): relacional, com replicação e sharding.
- **Histórico de chat:**
  - volume enorme (o SDI1 cita 60 bilhões de mensagens por dia em 2016 para Messenger e WhatsApp);
  - acesso concentrado em mensagens **recentes**, mas com acesso aleatório para busca, menções e saltos;
  - leitura:escrita ≈ 1:1 em conversas 1:1.
  
  O SDI1 recomenda **key-value** (escala horizontal, baixa latência) e cita HBase no Messenger e Cassandra no Discord.
- **Modelo:**
  - **1:1:** chave primária `message_id` (a ordem vem do ID, não de `created_at`, porque duas mensagens podem ter o mesmo instante);
  - **grupo:** chave composta `(channel_id, message_id)`, com `channel_id` como **chave de partição** (as consultas são sempre por canal).
- **`message_id`:** precisa ser **único e ordenável no tempo**. Opções:
  - `AUTO_INCREMENT` (bancos NoSQL raramente têm);
  - gerador global tipo Snowflake;
  - **sequência local por canal**, mais simples e suficiente, porque a ordem só importa dentro da conversa.

## Aprofundamentos

- **Descoberta de serviço:** após o login, escolhe o melhor servidor de chat (localização, capacidade). O SDI1 usa ZooKeeper como exemplo.
- **Fluxo 1:1:**
  1. A envia ao servidor de chat 1;
  2. obtém ID do gerador;
  3. publica na **fila de sincronização**;
  4. grava no key-value store;
  5. se B está online, encaminha ao servidor de B; se está offline, envia push;
  6. o servidor de B entrega via WebSocket.
- **Vários dispositivos:** cada dispositivo guarda `cur_max_message_id`. Mensagens novas são as destinadas ao usuário com ID maior que esse valor. Cada dispositivo sincroniza de forma independente.
- **Grupo pequeno:** a mensagem é **copiada para a caixa de entrada** (fila de sincronização) de cada membro. Isso simplifica a leitura e é barato com poucos membros (o SDI1 cita o WeChat limitando grupos a 500). É **fan-out na escrita**, inviável para grupos enormes.
- **Presença:**
  - login grava online + `last_active_at`; logout grava offline;
  - **desconexão:** marcar offline a cada queda faria o indicador piscar (túneis, redes instáveis). Use **heartbeat**: o cliente envia a cada ~5 s, e sem heartbeat por X segundos (ex.: 30) o usuário passa a offline;
  - **fan-out da presença:** pub/sub com um canal por par de amigos. Para grupos enormes, buscar a presença só quando o usuário abre o grupo.
- **Extras:**
  - mídia (compressão, object storage, miniaturas);
  - **criptografia de ponta a ponta**;
  - cache no cliente;
  - borda geográfica (o SDI1 cita o cache de borda do Slack);
  - queda de servidor de chat (a descoberta de serviço redireciona e os clientes reconectam);
  - reenvio com retry e fila.

## Conexões com os fundamentos (DDIA1)

- **Ordem e causalidade:** a sequência **por canal** garante ordem total dentro da conversa (como um log particionado por `channel_id`: ordem total por partição, nenhuma entre partições). Entre canais, não há ordem, e na maioria dos casos isso é aceitável. Ver [consistência e consenso](../03-sistemas-distribuidos/consistencia-e-consenso.md).
- **Sincronização por offset:** o `cur_max_message_id` por dispositivo é exatamente o **offset do consumidor** de um log. O dispositivo que volta a ficar online pede "tudo depois de X". O DDIA1 (cap. 12) propõe estender esse modelo até o cliente final. Ver [processamento de fluxo](../04-dados-derivados/processamento-de-fluxo.md).
- **Detecção de falhas por heartbeat e timeout:** o mesmo dilema de timeout curto × longo (falsos positivos × lentidão). Ver [falhas em sistemas distribuídos](../03-sistemas-distribuidos/falhas-em-sistemas-distribuidos.md).
- **Entrega e duplicatas:** reenvio pode duplicar. O cliente deve deduplicar pelo `message_id`, e o envio deve ter um ID gerado no cliente para idempotência de ponta a ponta.
- **Relacional × key-value para o histórico:** o SDI1 afirma que bancos relacionais "não lidam bem com a cauda longa" de dados. Isso é uma **generalização discutível**. O DDIA1 mostra que o que importa é o padrão de acesso e a chave de partição (`channel_id`, `message_id`), que funcionam bem tanto em LSM (Cassandra) quanto em tabelas relacionais particionadas. A escolha deve ser guiada por volume de escrita, operação e medição.

## Trade-offs e decisões

- **WebSocket × long polling × SSE:** bidirecional e persistente contra simplicidade. Server-Sent Events serve quando só o servidor envia.
- **Sequência local × global:** simplicidade contra ordem entre conversas.
- **Caixa de entrada por membro (fan-out na escrita) × leitura do canal (fan-out na leitura):** depende do tamanho do grupo.
- **Precisão da presença × custo:** frequência do heartbeat e do fan-out da presença.

## Aplicação no mundo real

- **Servidores WebSocket:** mantenha-os leves e com estado apenas de conexão. O roteamento entre servidores (quem tem a conexão de B?) costuma usar pub/sub (ex.: Redis pub/sub, NATS) ou um registro de conexões.
- **Balanceadores** com suporte a conexões longas e *draining* em deploys, para reconectar clientes de forma gradual.
- **Criptografia de ponta a ponta:** o Signal Protocol é a referência de mercado. É complexo, e implementações auditadas são preferíveis.
- **Push:** APNs e FCM para acordar apps em segundo plano.

## No Projeto prático (encurtador de links)

- **Painel de cliques ao vivo:** o dono vê cliques chegando em tempo real. Implementar com **SSE** (o servidor só envia), alimentado pelo worker de agregação via pub/sub.
- **Exercícios:** reconexão com "desde o último ID visto" (offset) e comparação entre SSE e polling a cada 5 s quanto a carga no servidor e latência percebida.

## Armadilhas comuns

- Usar `created_at` para ordenar mensagens.
- Marcar offline a cada queda de conexão.
- Fan-out da presença para grupos enormes.
- Não deduplicar mensagens reenviadas.
- Deploy que derruba todas as conexões de uma vez (tempestade de reconexões).

## Perguntas de verificação

1. Por que o long polling tem problemas quando o destinatário está conectado a outro servidor?
2. Por que o servidor de chat é um serviço com estado, e o que isso implica para escalar e fazer deploy?
3. Como o `cur_max_message_id` por dispositivo se relaciona com offsets de consumidor em logs?
4. Por que uma sequência local por canal basta para a ordem das mensagens?
5. Como o heartbeat evita que o indicador de presença pisque?

## Referências

- [SDI1, cap. 12, "Design a Chat System"]: escopo, polling, long polling, WebSocket, serviços com e sem estado, escala, armazenamento, modelo de dados, `message_id`, descoberta de serviço, fluxos 1:1 e de grupo, vários dispositivos, presença, heartbeat, extras.
- [DDIA1, cap. 11, "Partitioned Logs" e "Consumer offsets", pp. 446–450].
- [DDIA1, cap. 12, "Pushing state changes to clients", pp. 512–513].
- [DDIA1, cap. 8, "Detecting Faults" e "Timeouts and Unbounded Delays", pp. 280–287].
