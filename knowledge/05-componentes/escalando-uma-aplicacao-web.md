---
titulo: Escalando uma aplicação web (de um servidor a milhões de usuários)
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) com contraponto do DDIA1 (Lente); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Escalando uma aplicação web: de um servidor a milhões de usuários

> Uma aplicação costuma crescer em etapas. Começa com tudo em um servidor e, a cada gargalo, ganha um componente novo: banco separado, balanceador de carga, réplicas, cache, CDN, camada web sem estado, vários datacenters, filas, observabilidade e sharding. Este arquivo é o **mapa dos componentes**. Cada peça tem um arquivo próprio mais aprofundado.

## Por que importa

É o vocabulário básico de qualquer discussão de arquitetura web. Saber **qual problema cada componente resolve** (e qual custo traz) evita adicionar peças por moda e ajuda a reconhecer o próximo gargalo.

## Conceitos-chave: a sequência de evolução

1. **Um servidor só.** Aplicação, banco e cache na mesma máquina.
   - Fluxo: o DNS resolve o domínio para um IP, o cliente faz requisição HTTP e recebe HTML ou JSON.
   - O tráfego vem de clientes web e de apps móveis (API JSON).
2. **Separar camada web e camada de dados.** Cada uma escala de forma independente.
   - **Relacional ou NoSQL?** Para a maioria, relacional (maduro, com joins).
   - O SDI1 sugere NoSQL quando: é preciso latência muito baixa, os dados não têm relações, a necessidade é só serializar e desserializar, ou o volume é massivo.
   - Ver [modelos de dados](../02-dados/modelos-de-dados-e-consulta.md) para uma análise mais criteriosa.
3. **Vertical × horizontal.** Escalar verticalmente é simples, mas tem teto de hardware e não oferece failover. Escalar horizontalmente é o caminho para sistemas grandes.
4. **Balanceador de carga.**
   - Distribui tráfego entre servidores web.
   - Os clientes falam só com o IP público do balanceador; os servidores ficam em **IPs privados**.
   - Se um servidor cai, o tráfego vai para os outros. Se a carga cresce, basta adicionar servidores ao pool.
5. **Replicação do banco (primário e réplicas).**
   - Escritas vão ao primário, leituras às réplicas (a maioria das aplicações lê muito mais do que escreve).
   - Ganhos: desempenho (leituras em paralelo), confiabilidade (cópias em outros lugares) e disponibilidade.
   - Réplica caiu: leituras vão a outras réplicas ou ao primário.
   - Primário caiu: uma réplica é promovida. O SDI1 reconhece que, na prática, a réplica pode estar desatualizada e exigir scripts de recuperação. Ver [replicação](../03-sistemas-distribuidos/replicacao.md), onde o DDIA1 detalha esses riscos.
6. **Cache.** Camada em memória que evita consultas repetidas ao banco. Ver [cache](cache.md).
   - Padrão descrito no SDI1: consulta o cache; em caso de *miss*, lê o banco e grava no cache (*read-through*).
   - **Considerações:**
     - use para dados muito lidos e pouco alterados, porque o cache é volátil e não serve como armazenamento durável;
     - defina **expiração** (nem curta demais, nem longa demais);
     - cuide da **consistência** cache × banco (não há transação única entre os dois);
     - evite **ponto único de falha** (vários nós, memória sobrando);
     - escolha a **política de despejo** (LRU é a mais comum; também existem LFU e FIFO).
7. **CDN.** Servidores geograficamente distribuídos que servem conteúdo **estático** (imagens, vídeo, CSS, JS) perto do usuário. Ver [CDN](cdn.md).
   - Fluxo: em caso de *miss*, a CDN busca na **origem** (servidor web ou object storage) e guarda pelo **TTL**.
   - **Considerações:** custo por transferência (não coloque itens pouco acessados), TTL adequado, **fallback** para a origem se a CDN cair, invalidação por API ou por **versionamento** na URL (`image.png?v=2`).
8. **Camada web sem estado.**
   - Tirar o estado (ex.: sessão) dos servidores web e colocá-lo em um armazenamento compartilhado (banco, Redis, NoSQL).
   - **Com estado:** cada usuário precisa voltar ao mesmo servidor (*sticky sessions*), o que complica adicionar e remover servidores e tratar falhas.
   - **Sem estado:** qualquer servidor atende qualquer requisição. É mais simples, robusto e permite **autoescala**.
9. **Vários datacenters.**
   - **GeoDNS** roteia o usuário ao datacenter mais próximo. Se um cai, todo o tráfego vai para o outro.
   - Desafios: redirecionar tráfego, **sincronizar dados** entre regiões (replicação assíncrona entre datacenters), e testar e fazer deploy em todas as regiões.
10. **Fila de mensagens.** Componente durável que desacopla produtores e consumidores, absorve picos e permite escalar cada lado de forma independente. Exemplo: processamento de fotos por workers, com mais workers quando a fila cresce. Ver [processamento de fluxo e filas](../04-dados-derivados/processamento-de-fluxo.md).
11. **Logs, métricas e automação.**
    - Logs centralizados.
    - Métricas de host (CPU, memória, I/O), agregadas por camada e **de negócio** (usuários ativos diários, retenção, receita).
    - Integração contínua e automação de build, teste e deploy.
12. **Escalar o banco.**
    - **Vertical:** máquinas enormes existem. O SDI1 cita o Stack Overflow de 2013 com um único primário para mais de 10 milhões de visitantes mensais. Mas há teto, ponto único de falha e custo alto.
    - **Horizontal (sharding):** mesmo esquema e dados distintos por shard. A **chave de sharding** deve distribuir bem os dados.
    - Desafios: **resharding** quando um shard enche ou fica desigual (hashing consistente ajuda), **problema da celebridade** (chave quente; pode exigir shard dedicado) e **joins entre shards**, que levam a desnormalizar.
    - Ver [particionamento](../03-sistemas-distribuidos/particionamento.md).

### Resumo do SDI1 para escalar

- Camada web sem estado.
- Redundância em todas as camadas.
- Cache sempre que fizer sentido.
- Vários datacenters.
- Estáticos na CDN.
- Sharding na camada de dados.
- Separar camadas em serviços.
- Monitorar e automatizar.

## Divergências e nuances entre as fontes

- **"Faça cache o máximo que puder" (SDI1) × custo da consistência (DDIA1).** O DDIA1 trata cache como **dado derivado**. Todo cache é um problema de sincronização (gravação dupla, invalidação) e deve ser introduzido **quando a medição justificar**. Posição da trilha: cache é uma decisão guiada por parâmetros de carga e percentis, não um padrão automático.
- **Sharding com `user_id % 4` (exemplo do SDI1)** × a crítica do DDIA1 ao `mod N`: ao mudar N, quase tudo se move. O próprio SDI1 aponta o hashing consistente como remédio no cap. 5.
- **Failover "simples" de réplica (SDI1)** × os riscos detalhados no DDIA1: perda de escritas assíncronas, split brain, chaves reutilizadas, timeout.
- **NoSQL "fácil de escalar" para sessões (SDI1):** é uma escolha razoável, mas o DDIA1 lembra que qualquer banco distribuído traz trade-offs de consistência. Para sessões, Redis com expiração é o padrão de mercado.
- **Escala vertical:** o DDIA1 reforça que uma máquina só vai muito longe e que distribuir cedo demais é complexidade prematura. O SDI1 concorda com o exemplo do Stack Overflow.

## Trade-offs e decisões

| Componente | Resolve | Custo introduzido |
|---|---|---|
| Balanceador | Disponibilidade e escala da camada web | Mais um salto; ele próprio precisa de redundância |
| Réplicas de leitura | Escala de leitura, disponibilidade | Leitura atrasada, failover delicado |
| Cache | Latência e carga no banco | Consistência, invalidação, avalanche ao expirar |
| CDN | Latência para estáticos, carga na origem | Custo por transferência, invalidação |
| Camada sem estado | Autoescala, tolerância a falhas | Estado vai para outro armazenamento |
| Vários datacenters | Latência geográfica, desastre regional | Sincronização de dados, complexidade operacional |
| Fila | Desacoplamento, picos, trabalho assíncrono | Consistência eventual, duplicatas, monitorar lag |
| Sharding | Escala de escrita e de volume | Resharding, pontos quentes, joins, operação |

## Aplicação no mundo real

- **Ordem pragmática:** meça primeiro. Em geral, a sequência de maior retorno é: índices e consultas → cache de leitura → réplicas → fila para trabalho lento → escala vertical do banco → só então sharding.
- **Balanceadores:** Nginx, HAProxy, Envoy ou os de nuvem (ALB, Cloud Load Balancing). Configure health checks.
- **Sessões sem estado:** JWT (cuidado com a revogação) ou sessão em Redis com TTL.
- **Observabilidade:** logs estruturados, métricas RED/USE e traces com OpenTelemetry. Dashboards e alertas por SLO (ver o capítulo de SLO do Google SRE, que é Referência primária da trilha).
- *Produtos e limites citados mudam. Confirme na documentação oficial.*

## No Projeto prático (encurtador de links)

Esta é praticamente a trilha de módulos do Projeto prático:

1. uma instância + PostgreSQL;
2. medir com k6;
3. cache de redirecionamento (Redis);
4. balanceador + 2 instâncias sem estado;
5. réplica de leitura;
6. fila para cliques;
7. CDN para páginas estáticas e, se fizer sentido, redirecionamentos na borda;
8. sharding só se a medição exigir, como exercício.

Cada etapa com **hipótese → medição antes → mudança → medição depois**, conforme a §2 do bootstrap.

## Armadilhas comuns

- Adicionar componentes sem um gargalo medido.
- Guardar sessão em memória do servidor e depois descobrir que não dá para escalar.
- Cache sem expiração nem estratégia de invalidação.
- Balanceador ou cache como novo ponto único de falha.
- Sharding precoce, com `mod N`.

## Perguntas de verificação

1. Qual problema o balanceador resolve, e qual problema novo ele cria?
2. Por que a camada web sem estado é pré-requisito para autoescala?
3. Cite quatro considerações ao usar uma CDN.
4. Quais três desafios o sharding introduz?
5. Em que ponto da evolução você colocaria uma fila? Dê um exemplo.

## Referências

- [SDI1, cap. 1, "Scale from Zero to Millions of Users"]: servidor único, banco, vertical × horizontal, balanceador, replicação, cache, CDN, camada sem estado, datacenters, fila, logs e métricas, sharding.
- [DDIA1, cap. 1, "Approaches for Coping with Load", pp. 17–18]: vertical × horizontal, sem estado × com estado.
- [DDIA1, cap. 5 e 6]: aprofundamento de replicação e particionamento (ver arquivos correspondentes).
