---
titulo: Transações e níveis de isolamento
fontes: [DDIA1]
status-validacao: base primária única (DDIA1); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Transações e níveis de isolamento

> Transação é agrupar leituras e escritas em uma unidade lógica que **ou acontece inteira, ou não acontece**. Ela existe para simplificar a aplicação: falhas parciais e muitas condições de corrida viram um simples "abortou, tente de novo". O problema é que "ACID" e os nomes dos níveis de isolamento significam coisas diferentes em cada banco. É preciso saber **quais anomalias** cada nível deixa passar.

## Por que importa

Bugs de concorrência são raros, dependem de timing e são difíceis de reproduzir, mas já causaram perdas financeiras e corrupção de dados reais. "Usar um banco ACID" não basta: a maioria dos bancos relacionais roda com isolamento fraco por padrão.

## Conceitos-chave

### ACID, com precisão

- **Atomicidade:** se algo falha no meio de várias escritas, **tudo é desfeito**. O nome melhor seria "abortabilidade". Não tem a ver com concorrência. Permite *retry* seguro.
- **Consistência (C):** invariantes da **aplicação** (ex.: créditos = débitos). O banco só ajuda com algumas restrições (chaves estrangeiras, unicidade). O resto é responsabilidade da aplicação. O C "não pertence de verdade" ao ACID.
- **Isolamento:** transações concorrentes não interferem umas nas outras. O ideal teórico é a **serializabilidade**: o resultado é igual ao de rodar uma de cada vez. Na prática, quase sempre se usa algo mais fraco.
- **Durabilidade:** dado confirmado não se perde. Em um nó, significa disco mais WAL; em sistemas replicados, cópia em N nós. **Não existe durabilidade perfeita:**
  - falhas correlacionadas derrubam todas as réplicas;
  - SSDs violam o `fsync` em queda de energia;
  - corrupção silenciosa se espalha para réplicas e backups;
  - SSD desligado perde dados em semanas.
  
  Combine disco, replicação e backups.
- **"Consistência" tem quatro sentidos** diferentes: consistência eventual de réplicas, hashing consistente, o C do CAP (linearizabilidade) e o C do ACID. Sempre pergunte qual.
- **BASE** (*basically available, soft state, eventual consistency*) é ainda mais vago: na prática significa "não é ACID".

### Objeto único × vários objetos

- **Operações de objeto único:** quase todo motor garante atomicidade e isolamento por objeto (sem JSON pela metade, sem valor meio velho meio novo). Existem operações atômicas como **incremento** e **compare-and-set**. Úteis, mas não são "transações", apesar do marketing ("lightweight transactions").
- **Transações de vários objetos** são necessárias para:
  - manter chaves estrangeiras válidas;
  - atualizar dados desnormalizados juntos (ex.: e-mail + contador de não lidos);
  - manter índices secundários em sincronia com os dados.
- **Retry não é trivial:**
  - a transação pode ter sido confirmada e só a resposta se perdeu, e a repetição duplica (precisa deduplicação);
  - retry sob sobrecarga piora a situação (use backoff exponencial e limite de tentativas);
  - só vale a pena para erros transitórios;
  - efeitos colaterais externos (ex.: enviar e-mail) acontecem mesmo se a transação abortar.
  
  Muitos ORMs nem tentam repetir.

### Níveis de isolamento fracos

**Read committed** (padrão em PostgreSQL, Oracle, SQL Server)
- **Sem leitura suja:** só se vê dado confirmado.
- **Sem escrita suja:** só se sobrescreve dado confirmado, com trava de escrita por linha.
- Implementação típica: o banco guarda o valor antigo confirmado e o entrega a quem lê enquanto a escrita não confirma. Não usa trava de leitura.
- **Não impede:** perda de atualização em contadores, leitura enviesada, write skew.

**Isolamento por snapshot** (chamado *repeatable read* no PostgreSQL e no MySQL, e *serializable* no Oracle)
- **Problema resolvido: leitura enviesada** (*read skew*, ou leitura não repetível). Exemplo: ver o saldo de uma conta antes da transferência e o de outra depois; o dinheiro "some". Em backups e consultas analíticas longas, isso vira inconsistência permanente.
- **Solução:** cada transação lê um **snapshot consistente**, o estado confirmado no momento em que começou.
- **MVCC** (*multi-version concurrency control*): o banco guarda várias versões de cada linha, marcadas com o ID da transação que criou e da que apagou. Regras de visibilidade escondem escritas de transações em andamento, abortadas ou posteriores. A coleta de lixo (*vacuum*) remove versões que ninguém mais enxerga.
- Princípio: **quem lê não bloqueia quem escreve, e quem escreve não bloqueia quem lê.**
- Alternativa de implementação: B-tree *append-only / copy-on-write*, em que cada escrita gera uma nova raiz, que já é um snapshot.
- **Confusão de nomes:** o padrão SQL é de 1975, anterior ao isolamento por snapshot. "Repeatable read" significa coisas diferentes em cada banco. No DB2, significa serializável.

### Anomalias entre escritas concorrentes

**Perda de atualização** (*lost update*)
- Dois ciclos de ler, modificar e escrever se atropelam. Exemplos: contador, saldo, item adicionado a uma lista em JSON, página wiki salva inteira.
- **Soluções:**
  - **operações atômicas do banco** (`UPDATE ... SET valor = valor + 1`), normalmente a melhor opção. Cuidado com ORMs que geram ler, modificar e escrever sem perceber;
  - **trava explícita** (`SELECT ... FOR UPDATE`);
  - **detecção automática:** o banco aborta a transação que perderia a atualização. O *repeatable read* do PostgreSQL e o *serializable* do Oracle fazem isso; o *repeatable read* do MySQL/InnoDB não (panorama de 2017);
  - **compare-and-set:** atualizar só se o valor não mudou. Cuidado: pode ler de um snapshot antigo e não proteger.
- **Em bancos replicados** (multilíder ou sem líder), travas e CAS não se aplicam. Use operações **comutativas** (ex.: incremento, adicionar a um conjunto) ou irmãos mais mesclagem. LWW perde atualizações.

**Write skew e fantasmas**
- **Write skew:** duas transações leem o mesmo conjunto, decidem com base nele e escrevem em objetos **diferentes**, e juntas violam uma invariante.
  - Exemplo clássico: dois médicos de plantão pedem saída ao mesmo tempo, cada um vê "há 2 de plantão", e o hospital fica sem ninguém.
  - Outros: reserva dupla de sala, dois jogadores movendo peças para a mesma casa, dois cadastros com o mesmo nome de usuário, **gasto duplo** de saldo.
- **Padrão comum:**
  1. um `SELECT` verifica uma condição;
  2. a aplicação decide;
  3. uma escrita muda o resultado daquela verificação.
- **Fantasma:** a escrita de uma transação altera o resultado da *busca* de outra. É especialmente difícil quando a verificação é pela **ausência** de linhas, porque não há linha para travar.
- **Defesas:**
  - isolamento **serializável** (a melhor);
  - **restrições do banco** quando possível (ex.: unicidade resolve o nome de usuário);
  - `SELECT ... FOR UPDATE` nas linhas que embasam a decisão;
  - **materializar o conflito:** criar linhas-trava (ex.: sala × intervalo de 15 minutos) para ter o que travar. É último recurso, porque vaza o controle de concorrência para o modelo de dados.

### Serializabilidade: três implementações

| Técnica | Como funciona | Custo / limite |
|---|---|---|
| **Execução serial real** | Uma thread executa uma transação por vez; transações como *stored procedures*, dados em memória (VoltDB, Redis, Datomic) | Transações curtas; vazão limitada a um núcleo por partição; transações entre partições muito mais lentas |
| **Two-phase locking (2PL)** | Trava compartilhada para ler, exclusiva para escrever, mantidas até o fim da transação; **travas de predicado** ou de faixa de índice (*next-key locking*) contra fantasmas | Leitores e escritores se bloqueiam; latência instável nos percentis altos; deadlocks frequentes (o banco aborta um lado) |
| **Serializable Snapshot Isolation (SSI)** | Otimista: roda sobre snapshot e, no commit, aborta quem agiu com base em **premissa desatualizada** (leitura de versão MVCC antiga, ou escrita posterior que afetou uma leitura) | Abortos crescem com a contenção; transações de leitura e escrita precisam ser curtas. Usado no `SERIALIZABLE` do PostgreSQL desde a 9.1 |

- **2PL não é 2PC.** Two-phase *locking* é isolamento. Two-phase *commit* é atomicidade distribuída (ver [consistência e consenso](consistencia-e-consenso.md)).
- **Pessimista × otimista:** travar antes (2PL, serial) ou verificar no fim (SSI). O otimista vence com pouca contenção e capacidade sobrando, e perde com muita contenção.
- **Transações não esperam pelo humano.** Na web, uma transação cabe dentro de uma requisição HTTP.

## Trade-offs e decisões

| Nível | Impede | Não impede | Custo |
|---|---|---|---|
| Read committed | Leitura e escrita sujas | Perda de atualização, read skew, write skew | Baixo |
| Snapshot | + read skew (alguns bancos também detectam perda de atualização) | Write skew, fantasmas em leitura e escrita | Baixo a médio |
| Serializável | Tudo | — | Médio a alto (bloqueios ou abortos e retries) |

- Escolher um nível fraco é aceitável **se** você sabe quais anomalias ele permite e protege as invariantes críticas com restrições, operações atômicas ou travas explícitas.

## Aplicação no mundo real

- **Saiba o padrão do seu banco:** o PostgreSQL usa *read committed* por padrão e oferece `REPEATABLE READ` (snapshot) e `SERIALIZABLE` (SSI). O MySQL/InnoDB usa *repeatable read* por padrão, com semântica própria. Confirme sempre na documentação da versão em uso.
- **Invariantes críticas primeiro no banco:** `UNIQUE`, `CHECK`, chaves estrangeiras e, no PostgreSQL, *exclusion constraints* com tipos de intervalo impedem reserva sobreposta sem lógica na aplicação.
- **Contadores e saldos:** `UPDATE ... SET x = x + ?` em vez de ler, modificar e escrever no ORM.
- **Transações serializáveis exigem retry:** trate o erro de falha de serialização (SQLSTATE `40001` no PostgreSQL) com backoff.
- **Idempotência** ao repetir operações que atravessam a rede: chave de idempotência gravada na mesma transação da operação.
- **Efeitos externos** (e-mail, pagamento): use o padrão *outbox*. Grave a intenção na mesma transação e um processo separado publica depois.

## No Projeto prático (encurtador de links)

- **Colisão de código curto:** dois pedidos geram o mesmo código ao mesmo tempo. Defesa: `UNIQUE` no código mais retry com novo código. É o caso "nome de usuário": restrição, não verificação prévia.
- **Alias personalizado** (`/meu-evento`): mesma lógica, com `UNIQUE` resolvendo a corrida.
- **Contador de cliques:** `UPDATE links SET cliques = cliques + 1` sob alta concorrência é seguro, mas vira ponto de contenção em links virais. É o gancho para agregar cliques de forma assíncrona (fila e lote).
- **Experimento sugerido:** 100 requisições concorrentes incrementando o contador com ler, modificar e escrever no código × incremento atômico. Contar as atualizações perdidas.

## Armadilhas comuns

- Achar que o padrão do banco é serializável.
- Verificar disponibilidade com `SELECT` e inserir depois, sem restrição nem trava (write skew clássico).
- ORM fazendo ler, modificar e escrever silenciosamente.
- Retry sem idempotência, que duplica efeitos.
- Confiar em compare-and-set que lê de um snapshot antigo.
- Confundir 2PL com 2PC.

## Perguntas de verificação

1. Por que a atomicidade do ACID não tem a ver com concorrência?
2. Qual anomalia o isolamento por snapshot resolve que o read committed não resolve? Dê um exemplo.
3. Explique write skew com o exemplo dos médicos. Por que uma trava de linha nem sempre resolve?
4. O que é um fantasma e por que "materializar o conflito" é o último recurso?
5. Compare 2PL e SSI quanto a bloqueio, latência e comportamento sob contenção.
6. Quando um retry de transação pode causar dano?

## Referências

- [DDIA1, cap. 7, "The Slippery Concept of a Transaction", pp. 222–233]: ACID, objeto único × vários objetos, tratamento de erros e abortos.
- [DDIA1, cap. 7, "Weak Isolation Levels", pp. 233–251]: read committed, snapshot/MVCC, perda de atualização, write skew, fantasmas, materialização de conflitos.
- [DDIA1, cap. 7, "Serializability", pp. 251–266]: execução serial, stored procedures, 2PL, travas de predicado e de faixa de índice, SSI.
