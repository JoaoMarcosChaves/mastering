---
titulo: Integração de dados, dados derivados, correção e ética
fontes: [DDIA1]
status-validacao: base DDIA1 (Lente); aguarda convergência com ≥2 primárias. Parte do capítulo-fonte é opinião declarada do autor (sinalizada abaixo)
atualizado: 2026-09-24
---

# Integração de dados, dados derivados, correção e ética

> Nenhum sistema atende a todos os padrões de acesso. Aplicações reais combinam banco transacional, cache, índice de busca, warehouse e modelos. A proposta do DDIA é ter **um sistema de registro** que define a ordem das escritas e **derivar** todo o resto dele por logs de eventos (CDC ou event sourcing), com consumidores **idempotentes e determinísticos**, em vez de transações distribuídas. Para correção, separa **pontualidade** (ver o dado atualizado) de **integridade** (não corromper): a segunda é a que não pode falhar. Fecha com a responsabilidade ética de quem constrói sistemas de dados sobre pessoas.

*Nota de fonte:* neste capítulo, o autor declara que escreve **opiniões pessoais** sobre o futuro dos sistemas de dados. As propostas ("desempacotar o banco", "fluxo de dados de ponta a ponta") são direções defendidas por ele, não consenso da área. Por isso ficam marcadas como **[opinião do autor]**.

## Por que importa

Assim que um sistema tem mais de um armazenamento (e quase todos têm: pelo menos banco + cache), aparece o problema de mantê-los coerentes. Resolver com gravação dupla ou transação distribuída é a origem de muitas inconsistências permanentes e de muitos incidentes.

## Conceitos-chave

### Integração combinando ferramentas especializadas

- **Todo software, mesmo um banco "genérico", foi projetado para um padrão de uso.** Frases como "99% das pessoas só precisam de X" dizem mais sobre quem fala do que sobre a tecnologia.
- **Raciocine sobre o fluxo de dados:**
  - onde o dado é escrito **primeiro**;
  - quais representações derivam de quais;
  - se possível, canalize toda entrada por **um sistema que decide a ordem das escritas** (replicação de máquina de estados), e derive o resto nessa mesma ordem.
  
  Usar CDC ou event sourcing importa menos do que **decidir uma ordem total**.
- **Dados derivados × transações distribuídas:**
  - as duas abordagens buscam o mesmo objetivo;
  - as transações ordenam por travas e garantem "uma vez" por commit atômico, dando linearizabilidade (e com ela ler as próprias escritas);
  - os logs ordenam por posição e garantem "uma vez" por retry determinístico + idempotência, e são assíncronos.
  - **[opinião do autor]** o XA tem tolerância a falhas e desempenho ruins, e dados derivados por log são o caminho mais promissor, sem ignorar a necessidade de garantias como ler as próprias escritas.
- **Limites da ordem total:** ela exige um líder que ordene. Não escala além de uma máquina, não funciona bem entre datacenters, entre microsserviços com estado próprio nem com clientes offline. Consenso que escala além de um nó ainda é pesquisa aberta.
- **Causalidade sutil:** alguém desfaz uma amizade e depois manda uma mensagem falando mal da pessoa. Se amizades e mensagens vivem em sistemas separados, a notificação pode ser processada antes do "desfazer amizade" e chegar ao ex. Pontos de partida:
  - timestamps lógicos;
  - registrar o evento "o que o usuário viu antes de decidir" e referenciá-lo;
  - resolução de conflitos, que não ajuda com efeitos externos como notificações.

### Lote e fluxo juntos

- **Integrar dados** é garantir que o dado chegue na forma certa a todos os lugares certos. O lote e o fluxo são as ferramentas: têm estilo funcional (funções determinísticas, entrada imutável, saída só acrescentada), e o fluxo acrescenta estado gerenciado e tolerante a falhas.
- **Assincronia é o que dá robustez:** uma falha fica contida localmente, enquanto transações distribuídas amplificam falhas. Índices entre partições também são mais escaláveis quando mantidos de forma assíncrona.
- **Reprocessar para evoluir:** o fluxo reflete mudanças rápido; o **lote reprocessa o histórico** para criar visões novas. Com reprocessamento dá para reestruturar dados por completo, não só adicionar campos opcionais.
  - **Migração gradual:** mantenha a visão antiga e a nova derivadas do mesmo dado, mova uma fração dos usuários, aumente aos poucos e remova a antiga. Cada etapa é reversível.
  - A analogia do livro é a troca de bitola das ferrovias inglesas: trilho duplo durante a transição.
- **Arquitetura lambda:** um lote exato e um fluxo aproximado em paralelo, mesclados na leitura. Popularizou "visões derivadas de eventos imutáveis", mas duplica lógica e operação, dificulta a mescla e acaba em lote incremental complexo.
- **Unificação:** o mesmo motor processa histórico e tempo real. Requisitos: replay, exactly-once e janelas por **tempo do evento** (ex.: Apache Beam sobre Flink ou Dataflow).

### "Desempacotar" o banco [opinião do autor]

- **Criar índice (`CREATE INDEX`)** é o mesmo que montar um seguidor ou iniciar um CDC: snapshot + backlog + manutenção contínua. **O fluxo de dados de uma organização se parece com um grande banco**, em que jobs de lote, fluxo e ETL fazem o papel de triggers e manutenção de views, e cada sistema derivado é um "tipo de índice".
- **Dois caminhos:**
  - **federação** (unificar leituras): uma interface de consulta sobre vários motores (ex.: *foreign data wrappers* do PostgreSQL);
  - **desempacotamento** (unificar escritas): conectar sistemas por CDC e logs, ao estilo Unix.
- **Log assíncrono + consumidores idempotentes** é mais simples e robusto que transação heterogênea, com **acoplamento fraco** em dois níveis:
  - sistêmico: o consumidor lento não derruba ninguém;
  - humano: times evoluem componentes de forma independente.
- **Não é para substituir bancos.** Se um produto faz tudo o que você precisa, use-o. Menos peças móveis é melhor, e construir para uma escala que não existe é otimização prematura. O desempacotamento dá amplitude, não profundidade.
- **O que falta:** um "shell" declarativo para compor sistemas (algo como `mysql | elasticsearch` para "criar um índice em outro sistema").
- **Aplicações em torno de fluxo de dados** ("banco de dentro para fora"): código de aplicação como **função de derivação** (índice, busca, modelo de ML, cache), com código e estado separados.
  - A planilha é o modelo ideal: mudou a entrada, recalcula a fórmula.
  - Assinar mudanças em vez de consultar: em vez de chamar o serviço de câmbio a cada compra, assine o fluxo de cotações e consulte localmente. É mais rápido e robusto: "a requisição de rede mais rápida e confiável é nenhuma requisição de rede". Isso é um join fluxo–tabela, dependente do tempo.

### Caminho de escrita × caminho de leitura

- **Caminho de escrita:** trabalho feito **antecipadamente**, quando o dado chega (avaliação *eager*).
- **Caminho de leitura:** trabalho feito **quando alguém pede** (avaliação *lazy*).
- **O dado derivado é onde os dois se encontram.** Índices, caches e views materializadas **deslocam essa fronteira**:
  - sem índice, a leitura faz tudo (grep);
  - com resultados pré-computados de todas as buscas possíveis, a escrita seria infinita;
  - o meio-termo é o cache das consultas mais comuns.
  
  O exemplo do Twitter do capítulo 1 é exatamente isso.
- **Clientes com estado e offline-first:** o estado no dispositivo é um cache do servidor ("os pixels na tela são uma view materializada").
- **Empurrar mudanças ao cliente** (Server-Sent Events, WebSockets) **estende o caminho de escrita até o usuário final**. Dispositivos offline se reconectam como consumidores de log com offset. A resistência é cultural: bancos, bibliotecas e protocolos assumem requisição e resposta.
- **Leituras também são eventos:** registrar consultas como eventos permite rastrear causalidade e proveniência (o que o cliente viu antes de comprar) e fazer joins entre partições (ex.: pontuação de fraude cruzando reputações particionadas).

### Buscando correção

**O argumento de ponta a ponta para bancos**
- Transações serializáveis **não protegem** contra a aplicação gravar dado errado.
- **Exatamente uma vez** (efeito final como se não houvesse falhas) é alcançado por **idempotência**.
- A **deduplicação do TCP** vale só dentro de uma conexão. A **transação** está atrelada à conexão. O **usuário** reenvia o formulário depois de um timeout. Nenhuma dessas camadas, sozinha, evita a transferência dupla.
- **Solução: um ID de operação gerado no cliente** (UUID em campo oculto, ou hash dos campos) que percorre todo o caminho até o banco, com `UNIQUE` em uma tabela de requisições. A restrição de unicidade funciona mesmo com isolamento fraco. A tabela de requisições **já é um log de eventos**.
- **Argumento de ponta a ponta** (Saltzer, Reed e Clark, 1984): certas funções só podem ser feitas corretamente nas pontas da comunicação. Isso vale para deduplicação, checksums de integridade e criptografia. Recursos de baixo nível ajudam, mas não bastam.
- **[opinião do autor]** ainda não achamos a abstração certa para tolerância a falhas de ponta a ponta. Transações são boas, mas caras entre sistemas heterogêneos, e mecanismos refeitos na aplicação costumam estar errados.

**Impor restrições**
- **Unicidade exige consenso.** O caminho comum é um líder. É possível **escalar particionando pelo valor único** (hash do nome de usuário). Replicação multilíder assíncrona está descartada se a rejeição precisa ser imediata.
- **Unicidade com logs:**
  1. o pedido vai para a partição do hash do nome;
  2. um processador sequencial decide e emite sucesso ou rejeição;
  3. o cliente aguarda a resposta.
  
  Escala com partições e serve para qualquer restrição em que **escritas conflitantes vão para a mesma partição**.
- **Requisição entre partições sem commit atômico** (transferência A → B):
  1. registrar o **pedido único** com ID no log (uma escrita atômica);
  2. um processador deriva débito (partição A) e crédito (partição B) com o mesmo ID;
  3. consumidores deduplicam pelo ID.
  
  O resultado é aplicado exatamente uma vez nos dois lados, sem 2PC. Validar saldo exige um processador particionado pela conta pagadora antes do passo 1.

**Pontualidade × integridade**
- **Pontualidade** (*timeliness*): o usuário vê o estado atualizado. A violação é temporária: é a **consistência eventual**.
- **Integridade:** ausência de corrupção (nada perdido, nada contraditório, derivação correta). A violação é permanente: é a **inconsistência perpétua**, que exige checagem e reparo explícitos.
- **Na maioria das aplicações, integridade importa muito mais.** Uma compra que demora um dia para aparecer na fatura é normal. Um saldo que não bate com a soma das transações é inaceitável.
- **Sistemas de fluxo desacoplam as duas coisas:** mantêm integridade sem pontualidade, com:
  - a escrita como uma **mensagem única** atômica;
  - derivações **determinísticas**;
  - **ID de requisição de ponta a ponta**;
  - mensagens **imutáveis** e reprocessáveis.

**Restrições relaxadas e compensação**
- Muitos negócios toleram violação temporária com **transação compensatória**: pedir desculpas e oferecer outra opção (nome de usuário, assento), repor estoque com desconto, *overbooking* deliberado de voos e hotéis, taxa por saque a descoberto com limite diário. O processo de desculpas **já existe** para outras causas (empilhadeira que destrói estoque, voo cancelado).
- **Aceitar o custo da desculpa é uma decisão de negócio.** Se aceitável, dá para gravar de forma otimista e validar depois, sempre **antes** de ações irreversíveis. A integridade continua obrigatória.
- **Sistemas que evitam coordenação:** integridade forte sem coordenação síncrona. Funcionam em vários datacenters de forma independente, com coordenação só onde é indispensável. **Coordenação reduz desculpas por inconsistência, mas pode aumentar desculpas por indisponibilidade.** Procure o equilíbrio.

**Confie, mas verifique**
- Modelos de sistema são probabilísticos, não binários. Bits invertidos em memória (inclusive *rowhammer*), corrupção em disco e rede, **bugs em bancos maduros** (há relatos de falhas de unicidade e de write skew em isolamento serializável em bancos consagrados) e, muito mais, **bugs na aplicação**.
- **Auditoria:** leia os dados de volta e verifique. HDFS e S3 releem e comparam réplicas continuamente. **Teste restaurar backups.**
- **Projetar para auditabilidade:** eventos imutáveis + derivação determinística permitem **recomputar e comparar** o estado derivado, usar hashes no log e depurar "viajando no tempo".
- **Checagens de integridade de ponta a ponta** dão confiança para mudar rápido, como testes automatizados.
- **Ferramentas criptográficas:** árvores de Merkle, *certificate transparency*. Blockchains têm ideias interessantes de verificação de integridade. O autor declara ceticismo sobre prova de trabalho e tolerância bizantina.

### Fazer a coisa certa (ética)

- **Dados frequentemente são sobre pessoas.** A dignidade humana vem primeiro, e a responsabilidade ética é também de quem constrói. Existe o Código de Ética da ACM.
- **Análise preditiva:** decisões automatizadas sobre crédito, emprego, seguro ou justiça podem criar uma "prisão algorítmica": exclusão sistemática, sem prova e sem recurso.
- **Viés e discriminação:** modelos extrapolam o passado e **codificam discriminação** existente. Atributos correlacionados (CEP, IP) funcionam como proxies de características protegidas.
- **Responsabilização:** quem responde pelo erro de um algoritmo? Dá para explicar a decisão a um juiz? Predições são probabilísticas e erram em casos individuais. Dados errados deixam a pessoa **sem recurso**.
- **Ciclos de realimentação:** recomendações criam câmaras de eco. Score de crédito usado em contratação cria espirais de exclusão. É preciso **pensamento sistêmico**.
- **Privacidade e rastreamento:** quando o anunciante é o cliente, o rastreamento vira **vigilância**. Experimento proposto pelo autor: trocar "dados" por "vigilância" nas frases corporativas.
  - O **consentimento** raramente é significativo: políticas obscuras, dados sobre terceiros, relação assimétrica, serviço *de facto* obrigatório.
  - **Privacidade é direito de decidir** o que revelar a quem. A coleta massiva **transfere esse direito** do indivíduo para a empresa.
- **Dados como ativo e como poder:** dados comportamentais são o ativo central de negócios de anúncio. São um "**ativo tóxico**": vazam, são vendidos em falências, exigidos por governos. Colete pensando em **todos os governos futuros**.
- **Paralelo com a Revolução Industrial:** a poluição exigiu regulação, e a proteção de dados é o equivalente. Legislação de proteção de dados propõe finalidade específica e dados **mínimos necessários**, o que colide com a filosofia de "coletar tudo" do Big Data.

## Trade-offs e decisões

- **Log assíncrono × transação distribuída:** robustez e acoplamento fraco contra pontualidade (ler as próprias escritas não vem de graça).
- **Ordem total × escala:** um líder que ordena é simples, mas limitado.
- **Validar antes × compensar depois:** coordenação e latência contra o custo das desculpas.
- **Coletar mais dados × risco:** valor analítico contra responsabilidade, custo de proteção e dano potencial a pessoas.

## Aplicação no mundo real

- **Um sistema de registro por entidade**, e todo o resto derivado dele via CDC (ex.: Debezium) ou outbox. Proíba gravação dupla no código.
- **Chave de idempotência de ponta a ponta:** gerada no cliente, enviada em cabeçalho (padrão comum em APIs de pagamento), gravada com `UNIQUE` na mesma transação do efeito.
- **Reconciliação periódica** (auditoria): jobs que recomputam agregados a partir dos eventos e comparam com o estado atual, alertando em caso de divergência.
- **Backups testados:** restauração automatizada e periódica em ambiente isolado.
- **Migrações com visões paralelas:** *shadow reads* e *feature flags* para mover tráfego gradualmente para o modelo novo.
- **Tempo real até o cliente:** SSE e WebSockets para empurrar mudanças, com reconexão por offset ou cursor.
- **Privacidade por projeto:** coleta mínima, retenção definida, exclusão real (inclusive em sistemas derivados e backups, o que é difícil), pseudonimização. No Brasil, a **LGPD** (Lei 13.709/2018); na UE, o **GDPR** (sucessor da diretiva de 1995 citada no livro). Consulte o texto legal e a orientação jurídica aplicável.

## No Projeto prático (encurtador de links)

- **Sistema de registro:** a tabela de links no PostgreSQL. Cache de redirecionamento e índice de busca (se existir) são **derivados**, atualizados por CDC ou outbox, nunca por gravação dupla.
- **Idempotência de ponta a ponta na criação:** o cliente envia `Idempotency-Key` e a API grava em uma tabela com `UNIQUE`. Um reenvio após timeout devolve o mesmo código.
- **Pontualidade × integridade nos cliques:** o contador pode atrasar minutos (pontualidade), mas nenhum clique pode ser perdido ou contado em dobro (integridade). Faça uma reconciliação diária recomputando dos eventos brutos.
- **Ética e privacidade:** cliques contêm IP e user-agent, que são dados pessoais. Defina retenção (ex.: agregar e descartar IP após N dias), anonimize e documente a finalidade.

## Armadilhas comuns

- Gravação dupla entre banco e sistemas derivados.
- Achar que transação serializável garante correção da aplicação.
- Idempotência só em uma camada (TCP, broker ou banco), e não de ponta a ponta.
- Nunca reconciliar ou auditar dados derivados.
- Tratar coleta de dados pessoais como gratuita e sem risco.

## Perguntas de verificação

1. Por que "decidir uma ordem total" importa mais do que escolher entre CDC e event sourcing?
2. Explique o argumento de ponta a ponta com o exemplo de reenviar um formulário de transferência.
3. Como fazer uma transferência entre partições com correção "exatamente uma vez" sem 2PC?
4. Diferencie pontualidade de integridade. Qual costuma ser mais crítica e por quê?
5. Dê dois exemplos de negócios que aceitam violar uma restrição e compensar depois.
6. O que significa "caches e índices deslocam a fronteira entre caminho de escrita e de leitura"?
7. Cite dois riscos éticos de sistemas de análise preditiva.

## Referências

- [DDIA1, cap. 12, "Data Integration", pp. 490–499]: combinar ferramentas, fluxo de dados, dados derivados × transações distribuídas, limites da ordem total, causalidade, lote e fluxo, reprocessamento, arquitetura lambda, unificação.
- [DDIA1, cap. 12, "Unbundling Databases", pp. 499–515]: composição de armazenamento, federação × desempacotamento, aplicações em torno de fluxo de dados, caminho de escrita × leitura, clientes com estado, leituras como eventos.
- [DDIA1, cap. 12, "Aiming for Correctness", pp. 515–533]: argumento de ponta a ponta, IDs de operação, restrições com logs, requisições entre partições, pontualidade × integridade, restrições relaxadas, sistemas sem coordenação, auditoria.
- [DDIA1, cap. 12, "Doing the Right Thing", pp. 533–544]: análise preditiva, viés, responsabilização, ciclos de realimentação, privacidade, vigilância, consentimento, dados como poder, legislação.
