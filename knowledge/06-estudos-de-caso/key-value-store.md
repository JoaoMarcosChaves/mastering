---
titulo: "Estudo de caso: key-value store distribuído (estilo Dynamo)"
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) confrontada com primária (DDIA1); contém divergências registradas
atualizado: 2026-09-24
---

# Estudo de caso: key-value store distribuído (estilo Dynamo)

> Projetar um banco chave-valor com `put(key, value)` e `get(key)` que seja **altamente disponível, escalável e com consistência ajustável**. Este estudo de caso junta quase tudo de sistemas distribuídos: particionamento por hashing consistente, replicação, quóruns, versionamento de conflitos, detecção de falhas por gossip, *hinted handoff*, anti-entropia com árvores de Merkle e o motor de armazenamento LSM.

## Por que importa

É o "projeto integrador" dos fundamentos. Mostra como técnicas separadas (hashing consistente, quóruns, vetores de versão, SSTables, filtros de Bloom) se combinam em um sistema real (Dynamo, Cassandra, Riak), e onde cada simplificação cobra seu preço.

## Requisitos do exercício (SDI1)

- Pares pequenos (< 10 KB).
- Grande volume.
- Alta disponibilidade (responde mesmo durante falhas).
- Escalabilidade e escala automática.
- **Consistência ajustável.**
- Baixa latência.

## Componentes

**1. Um servidor:** tabela hash em memória. Otimizações: compressão, e só o que é frequente em memória, com o resto em disco. Satura rápido, então precisa de distribuição.

**2. Particionamento:** **hashing consistente** com nós virtuais. Dá escala automática e heterogeneidade (mais nós virtuais para máquinas maiores). Ver [particionamento](../03-sistemas-distribuidos/particionamento.md).

**3. Replicação:**
- Cada chave vai para os **N primeiros servidores distintos** no sentido horário do anel (ignorando nós virtuais do mesmo servidor físico).
- Réplicas em **datacenters diferentes**, porque falhas no mesmo datacenter são correlacionadas.
- A replicação é assíncrona.

**4. Consistência por quórum:**
- N réplicas, a escrita espera W confirmações e a leitura espera R. Um **coordenador** faz a ponte entre cliente e nós.
- Configurações:
  - W = 1 ou R = 1: rápido;
  - R = 1, W = N: leitura rápida;
  - W = 1, R = N: escrita rápida;
  - típico: N = 3, W = R = 2.
- **Modelos:**
  - forte (nunca lê valor antigo);
  - fraco;
  - **eventual** (com tempo, todas as réplicas convergem).
  
  O SDI1 recomenda consistência eventual, como Dynamo e Cassandra.

**5. Resolução de inconsistências por versionamento:**
- Cada modificação vira uma **versão imutável**. Escritas concorrentes (ex.: "johnSanFrancisco" × "johnNewYork") geram **irmãos** em conflito.
- **Relógio vetorial** (termo do SDI1): pares [servidor, contador] por item. Se todos os contadores de Y são ≥ aos de X, então X é ancestral de Y. Se não, há conflito, e o **cliente resolve**.
- Custos: complexidade no cliente, e vetores crescendo (truncar os mais antigos; o artigo do Dynamo relata que isso não foi problema na prática).

**6. Tratamento de falhas:**
- **Detecção:** não basta um nó dizer que outro morreu. **Gossip**: cada nó mantém a lista de membros com contadores de heartbeat, incrementa o seu e propaga a nós aleatórios. Heartbeat parado por tempo demais significa offline.
- **Falha temporária:** **quórum frouxo** (usa os primeiros W e R nós *saudáveis* no anel) + **hinted handoff** (o substituto devolve os dados quando o dono volta).
- **Falha permanente:** **anti-entropia** com **árvore de Merkle**. Hashes por balde de chaves, subindo até a raiz. Compara as raízes e desce só nos ramos divergentes. O volume sincronizado fica proporcional à **diferença**, não ao tamanho total (ex.: 1 milhão de baldes para 1 bilhão de chaves).
- **Queda de datacenter:** replicação entre datacenters.

**7. Arquitetura:** API simples, coordenador, anel com hashing consistente, **totalmente descentralizado** (todo nó tem as mesmas responsabilidades, sem ponto único de falha) e replicação em vários nós.

**8. Caminhos de escrita e leitura (baseados no Cassandra):**
- **Escrita:** *commit log* → memtable (cache em memória) → quando enche, grava uma **SSTable** em disco.
- **Leitura:** memtable → se não achar, **filtro de Bloom** para descobrir quais SSTables podem conter a chave → lê essas SSTables.
- É a **LSM-tree** descrita no DDIA1. Ver [armazenamento e índices](../02-dados/armazenamento-e-indices.md).

## Divergências entre as fontes

- **CAP:**
  - o **SDI1** apresenta o CAP como "no máximo 2 de 3" e classifica sistemas em **CP, AP e CA** (declarando CA impossível no mundo real);
  - o **DDIA1** chama essa formulação de enganosa, recomenda evitar os rótulos e reformula como "**consistente ou disponível quando particionado**", lembrando que o custo diário real é a **latência**.
  - Ver [consistência e consenso](../03-sistemas-distribuidos/consistencia-e-consenso.md) para a posição da trilha.
- **"W + R > N garante consistência forte" (SDI1):** o **DDIA1** mostra que **não garante linearizabilidade**. Casos de borda:
  - quórum frouxo (que o próprio SDI1 recomenda para disponibilidade);
  - LWW com relógios dessincronizados;
  - escrita concorrente com leitura;
  - escrita parcialmente falha sem rollback;
  - restauração a partir de réplica antiga;
  - azar de timing.
  
  Leitura correta: **w + r > n aumenta a probabilidade de ler o valor mais recente**, mas não é uma garantia absoluta.
- **Quórum frouxo:** o SDI1 o apresenta como técnica de disponibilidade. O DDIA1 reforça que ele **deixa de ser quórum**: só garante durabilidade em W nós quaisquer, e uma leitura de R nós pode não ver a escrita até o *hinted handoff* terminar.
- **"Relógio vetorial" × "vetor de versão":** o DDIA1 observa que os termos são usados como sinônimos, mas não são a mesma coisa. Para comparar estados de réplicas, **vetores de versão** são a estrutura correta.
- **Resolução de conflitos no cliente:** os dois concordam que isso é complexo. O DDIA1 acrescenta **CRDTs** como forma automática de mesclar, e alerta para **LWW** (o padrão do Cassandra), que descarta escritas em silêncio.

## Trade-offs e decisões

- **Disponibilidade × consistência** durante falhas de rede, ajustável por W e R e pela escolha de quórum estrito ou frouxo.
- **Latência × consistência** mesmo sem falhas: W e R maiores esperam a réplica mais lenta.
- **Resolução de conflitos:** LWW (simples, perde dados) × irmãos + mescla (correto, complexo) × CRDTs (automático, só para certos tipos).
- **Descentralizado × com líder:** sem ponto único de falha e sem failover, contra consistência mais fraca e operação mais sutil.

## Aplicação no mundo real

- **Não implemente o seu.** Use Cassandra, ScyllaDB, Riak, DynamoDB (arquitetura diferente do Dynamo original, segundo o DDIA1) ou Redis (outro modelo: memória, replicação com líder, cluster com *hash slots*). Leia na documentação o que cada um garante de fato, e os relatórios Jepsen quando existirem.
- **Níveis de consistência por operação:** no Cassandra, `ONE`, `QUORUM`, `LOCAL_QUORUM`, `ALL`. Confirme na documentação atual.
- **Modelagem para LWW:** chaves imutáveis (ex.: UUID por evento) evitam perda por escrita concorrente.
- **Árvores de Merkle** aparecem também em Git, sistemas de arquivos, blockchain e *certificate transparency*.

## No Projeto prático (encurtador de links)

- O mapeamento código → URL **é** um key-value. **Exercício de decisão:** PostgreSQL (líder único, `UNIQUE` linearizável) × um key-value distribuído estilo Dynamo.
- Pergunta-guia: **o que acontece com a unicidade do código curto** em um sistema AP com quórum frouxo? Dois nós podem aceitar o mesmo código para URLs diferentes. A saída seria gerar códigos sem colisão (ver [gerador de IDs únicos](gerador-de-ids-unicos.md)) ou aceitar conflitos e resolvê-los.
- **Experimento opcional:** subir um cluster Cassandra de 3 nós em Docker, gravar com `ONE`, derrubar um nó e observar leituras antigas × `QUORUM`.

## Armadilhas comuns

- Tratar w + r > n como consistência forte.
- Usar quórum frouxo e esperar ler as próprias escritas.
- Ignorar a resolução de conflitos (LWW silencioso).
- Réplicas no mesmo datacenter ou rack.
- Esquecer a anti-entropia: dados pouco lidos ficam divergentes.

## Perguntas de verificação

1. Como as N réplicas de uma chave são escolhidas no anel com nós virtuais?
2. Com N = 3, que valores de W e R favorecem leitura rápida? Qual o risco?
3. Explique como vetores de versão detectam conflito entre D([s0,1],[s1,2]) e D([s0,2],[s1,1]).
4. Qual a diferença entre hinted handoff e anti-entropia?
5. Por que uma árvore de Merkle reduz o volume de sincronização?
6. Onde as fontes divergem sobre quóruns e CAP?

## Referências

- [SDI1, cap. 6, "Design a Key-Value Store"]: requisitos, um servidor, CAP, particionamento, replicação, quóruns, modelos de consistência, versionamento com relógios vetoriais, gossip, quórum frouxo, hinted handoff, Merkle, arquitetura, caminhos de escrita e leitura.
- [DDIA1, cap. 5, "Leaderless Replication", pp. 177–191]: quóruns e seus limites, quórum frouxo, detecção de escritas concorrentes, vetores de versão.
- [DDIA1, cap. 9, "Linearizability and quorums" e "The CAP theorem", pp. 334–338]: crítica ao CAP e aos quóruns como consistência forte.
- [DDIA1, cap. 3, "SSTables and LSM-Trees", pp. 76–79]: motor de armazenamento.
