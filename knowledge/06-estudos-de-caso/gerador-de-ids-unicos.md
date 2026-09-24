---
titulo: "Estudo de caso: gerador de IDs únicos em sistemas distribuídos"
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) confrontada com primária (DDIA1); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Estudo de caso: gerador de IDs únicos em sistemas distribuídos

> O `AUTO_INCREMENT` de um banco só funciona enquanto há um único banco. Com vários nós, é preciso gerar IDs **únicos**, muitas vezes **ordenáveis no tempo** e **compactos** (64 bits), sem coordenação a cada ID. As opções vão de incremento com passo por nó, UUID e servidor de tickets até o formato **Snowflake** (timestamp + datacenter + máquina + sequência).

## Por que importa

Chaves primárias, IDs de eventos, IDs de pedidos, códigos curtos: todo sistema distribuído precisa gerar identificadores sem colisão. A escolha afeta desempenho de índice, ordenação, tamanho de armazenamento, privacidade (IDs previsíveis) e dependência de relógios.

## Requisitos do exercício (SDI1)

- Únicos.
- Numéricos.
- 64 bits.
- **Ordenados por data** (IDs gerados à noite maiores que os da manhã, sem precisar incrementar de 1 em 1).
- Mais de 10.000 IDs por segundo.

## Opções

| Opção | Como funciona | A favor | Contra |
|---|---|---|---|
| **Incremento multiprimário** | Cada banco incrementa de k em k (k = número de bancos) | Escala com os bancos | Difícil com vários datacenters, IDs não crescem com o tempo entre servidores, mal com adição e remoção de nós |
| **UUID** | 128 bits aleatórios, gerados localmente | Sem coordenação, escala com os servidores | 128 bits (acima do requisito), não ordenado no tempo (v4), pode ser não numérico |
| **Servidor de tickets** | Um banco central com `AUTO_INCREMENT` (padrão do Flickr) | IDs numéricos, simples, bom para escala pequena e média | **Ponto único de falha**; vários servidores trazem problemas de sincronização |
| **Snowflake** | 64 bits divididos em campos (abaixo) | Atende a todos os requisitos, escala | Depende de relógio e de IDs de máquina bem geridos |

### Layout Snowflake (padrão descrito no SDI1)

| Campo | Bits | Observação |
|---|---|---|
| Sinal | 1 | Sempre 0 |
| Timestamp | 41 | ms desde uma época customizada (o Twitter usa novembro de 2010). 2^41 ms ≈ **69 anos** a partir da época |
| Datacenter | 5 | 32 datacenters |
| Máquina | 5 | 32 máquinas por datacenter |
| Sequência | 12 | 4.096 IDs por ms por máquina; zera a cada ms |

- **Datacenter e máquina** são fixados na inicialização. Mudá-los por engano gera IDs duplicados, então exigem cuidado operacional.
- **Ajuste dos campos:** menos bits de sequência e mais de timestamp servem para baixa concorrência e vida longa.
- **Tópicos extras:** sincronização de relógio (NTP) e **alta disponibilidade** do gerador, que é crítico.

## Conexões com os fundamentos (DDIA1)

- **Relógios não são confiáveis:** se o relógio de uma máquina **voltar** (correção do NTP), o Snowflake pode gerar IDs repetidos ou fora de ordem. Implementações robustas **detectam retrocesso** e esperam ou recusam a geração, e monitoram o desvio do relógio. Ver [falhas em sistemas distribuídos](../03-sistemas-distribuidos/falhas-em-sistemas-distribuidos.md).
- **"Ordenado por tempo" não é "ordenado por causalidade":** o DDIA1 observa que geradores como o Snowflake produzem IDs aproximadamente crescentes, mas **não garantem ordem consistente com a causalidade** entre máquinas (relógios dessincronizados, blocos de IDs). Se B leu o efeito de A, o ID de B pode ser menor. Para ordem causal, use timestamps de Lamport ou um log com líder. Ver [consistência e consenso](../03-sistemas-distribuidos/consistencia-e-consenso.md).
- **Blocos pré-alocados** (cada nó reserva um intervalo) são outra opção citada no DDIA1: rápidos e únicos, mas não causais.
- **Unicidade de verdade** (sem colisão) exige ou um árbitro único (líder, banco) ou partição do espaço de IDs por nó (bits de máquina). É o que o Snowflake faz.

## Aplicação no mundo real

- **UUID v7** (padronizado na RFC 9562): 128 bits com prefixo de timestamp, o que dá **ordenação temporal** e melhor localidade em índices B-tree do que o UUID v4. **ULID** é uma alternativa similar. Confirme o suporte da sua biblioteca e do seu banco.
- **Índices:** chaves aleatórias (UUID v4) espalham inserções pela B-tree, com mais divisões de página e pior cache. Chaves crescentes concentram as inserções no fim (bom para B-tree, potencial ponto quente em particionamento por faixa).
- **Privacidade e segurança:** IDs sequenciais expostos permitem **enumeração** (estimar volume de negócio, raspar recursos). Considere IDs opacos para exposição externa.
- **Sequências do banco** (`SEQUENCE`/`IDENTITY` no PostgreSQL) resolvem a maioria dos casos com um banco primário. É o "servidor de tickets" embutido.

## No Projeto prático (encurtador de links)

- **Opção A (simples):** sequência do PostgreSQL → base62. Único e sem colisão, mas **previsível** (enumerável).
- **Opção B:** Snowflake (64 bits) → base62 dá códigos de ~11 caracteres, e não 7 (ver [encurtador de URLs](encurtador-de-urls.md)).
- **Opção C:** aleatório de 7 caracteres base62 + `UNIQUE` + retry em colisão. Opaco, mas exige verificação de colisão.
- **Experimento:** gerar 1 milhão de IDs de cada tipo, medir a taxa de inserção e o tamanho do índice no PostgreSQL (UUID v4 × v7 × bigint sequencial).

## Armadilhas comuns

- Snowflake sem proteção contra relógio voltando.
- IDs de máquina configurados manualmente e duplicados.
- Assumir que ordem de ID é ordem causal.
- UUID v4 como chave primária clusterizada em tabelas enormes sem avaliar o impacto no índice.
- Expor IDs sequenciais sem pensar em enumeração.

## Perguntas de verificação

1. Por que `AUTO_INCREMENT` não resolve em sistemas distribuídos?
2. Quantos IDs por segundo uma máquina Snowflake gera no máximo? Por quantos anos o timestamp dura?
3. O que acontece se o relógio de um gerador Snowflake voltar 2 segundos?
4. Por que IDs ordenados no tempo não garantem ordem causal?
5. Compare UUID v4 e UUID v7 quanto ao impacto em índices B-tree.

## Referências

- [SDI1, cap. 7, "Design a Unique ID Generator in Distributed Systems"]: requisitos, incremento multiprimário, UUID, servidor de tickets, Snowflake (layout, timestamp, sequência), relógio, disponibilidade.
- [DDIA1, cap. 8, nota sobre geradores distribuídos de sequência (Snowflake), p. 294].
- [DDIA1, cap. 9, "Sequence Number Ordering", pp. 343–347]: geradores não causais, blocos pré-alocados, Lamport.
