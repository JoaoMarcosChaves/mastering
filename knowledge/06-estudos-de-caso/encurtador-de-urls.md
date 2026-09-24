---
titulo: "Estudo de caso: encurtador de URLs (o Projeto prático da trilha)"
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) confrontada com primária (DDIA1); contém correções de cálculo e inconsistências registradas
atualizado: 2026-09-24
---

# Estudo de caso: encurtador de URLs

> Dado um URL longo, gerar um código curto. Dado o código, redirecionar para o original. É o **Projeto prático** desta trilha: pouco código, muita decisão (geração de código, redirecionamento 301 × 302, cache, analytics, escala de leitura, abuso). Este arquivo junta o desenho do SDI1 com a crítica do DDIA1 e com as correções encontradas na validação.

## Por que importa

Parece trivial, mas exercita estimativa, modelagem, unicidade sob concorrência, cache, replicação, filas, rate limiting, observabilidade e privacidade, e cresce em etapas mensuráveis. Por isso é o **Projeto prático** da trilha.

## Escopo do exercício (SDI1)

- **Casos de uso:** encurtar (URL longa → curta) e redirecionar (curta → longa). Alta disponibilidade, escala e tolerância a falhas.
- **Premissas:** 100 milhões de URLs por dia; código "o mais curto possível" com [0-9a-zA-Z]; URLs **não** podem ser apagadas nem editadas (simplificação).

### Estimativas

- Escrita ≈ 100 M / 86.400 ≈ **~1.160/s**.
- Leitura (10:1) ≈ **~11.600/s**.
- 10 anos ≈ 100 M × 365 × 10 ≈ **365 bilhões de registros**.
- Armazenamento com ~100 bytes por URL: **365 bilhões × 100 B ≈ 36,5 TB**.
  > **Correção registrada na validação:** o SDI1 multiplica também por 10 anos e chega a 365 TB, mas os 365 bilhões **já incluem** os 10 anos. O valor correto com essas premissas é **~36,5 TB** (antes de índices e replicação). É um exemplo concreto de por que a trilha exige escrever premissas e unidades e conferir o cálculo.
- **Tamanho do código:** o menor n com 62^n ≥ 365 bilhões é **7** (62^7 ≈ 3,5 trilhões).

## Desenho de alto nível

**API REST**
- `POST /api/v1/data/shorten` com `{ longUrl }`, que devolve a URL curta.
- `GET /api/v1/{shortUrl}`, que responde com redirecionamento HTTP para a URL longa.

**301 × 302**
- **301 (permanente):** o navegador guarda o redirecionamento em cache, e as próximas visitas nem chegam ao serviço. Menos carga, **perde analytics**.
- **302 (temporário):** toda visita passa pelo serviço. Mais carga, **permite contar cliques e a origem**.
- É uma decisão de produto. Ver também controle de cache HTTP (`Cache-Control`) e o 307/308, que preservam o método (confirme a semântica na documentação HTTP/MDN).

**Modelo de dados:** tabela relacional (`id`, `short_url`, `long_url`). Uma tabela hash em memória é o ponto de partida, mas não escala em custo.

## Geração do código

| Abordagem | Como | A favor | Contra |
|---|---|---|---|
| **Hash + resolução de colisão** | CRC32, MD5 ou SHA-1 da URL longa, pegar 7 caracteres; em colisão, acrescentar sufixo e repetir | Tamanho fixo; mesma URL → mesmo código (se desejado) | Consulta ao banco a cada tentativa (filtro de Bloom ajuda); colisões crescem com o volume |
| **Base62 de um ID único** | Gerar ID único (ver [gerador de IDs](gerador-de-ids-unicos.md)) e converter para base 62 | Sem colisão por construção | Tamanho varia com o ID; **previsível** (IDs sequenciais expõem volume e permitem enumeração); depende do gerador |

- **Fluxo escolhido no SDI1 (base62):**
  1. a URL longa já existe? devolve o código;
  2. se não, gera ID → base62 → grava (id, código, URL).
- **Base62:** dígitos 0–9 = 0–9, a–z = 10–35, A–Z = 36–61. Exemplo: 11157 = 2×62² + 55×62 + 59 → "2TX".

### Inconsistência registrada entre capítulos do SDI1

O cap. 8 remete ao gerador do cap. 7 (**Snowflake de 64 bits**) e assume códigos de 7 caracteres. Mas um ID Snowflake tem ordem de grandeza ~10^18, que em base62 vira **~11 caracteres** (62^10 ≈ 8×10^17; 62^11 ≈ 5×10^19). O exemplo do livro usa um ID de 13 dígitos (~2×10^12), que cabe em 7 caracteres. Opções para códigos de 7 caracteres:

- sequência compacta própria (servidor de tickets ou sequência do banco, eventualmente em blocos por nó);
- código aleatório de 7 caracteres com checagem de unicidade.

## Redirecionamento

- **Fluxo:** balanceador → servidor web → cache (código → URL). Em caso de *miss*, banco. Não encontrou: código inválido (404).
- A leitura domina, então o cache é o componente principal.

## Tópicos extras (SDI1)

- **Rate limiter** contra abuso na criação.
- Camada web sem estado.
- Replicação e sharding do banco.
- **Analytics** (quantos cliques, quando, de onde).
- Disponibilidade, consistência e confiabilidade.

## O que o DDIA1 acrescenta (e o que o exercício de entrevista omite)

- **Unicidade sob concorrência:** "a URL já existe? se não, cria" é o padrão *verificar e depois inserir*, vulnerável a **write skew e fantasmas**. Duas requisições simultâneas com a mesma URL podem criar dois códigos. Defesa: `UNIQUE` em `long_url` (ou em um hash dela) com `INSERT ... ON CONFLICT`. Ver [transações](../03-sistemas-distribuidos/transacoes.md).
- **Idempotência de ponta a ponta:** timeout na criação seguido de retry do cliente não pode gerar dois códigos. Use chave de idempotência. Ver [integração de dados e correção](../04-dados-derivados/integracao-de-dados-e-correcao.md).
- **Ler as próprias escritas:** com réplicas de leitura, o usuário cria o link, testa na hora e recebe 404 da réplica atrasada. Ver [replicação](../03-sistemas-distribuidos/replicacao.md).
- **Cache como dado derivado:** se links puderem ser editados ou apagados (e na prática precisam, por abuso), a invalidação do cache vira problema de consistência. Evite gravação dupla. Ver [processamento de fluxo](../04-dados-derivados/processamento-de-fluxo.md).
- **Analytics sem prejudicar o redirecionamento:** publicar o evento de clique em uma fila e agregar de forma assíncrona. Separar OLTP de OLAP. Ver [armazenamento e índices](../02-dados/armazenamento-e-indices.md) e [processamento em lote](../04-dados-derivados/processamento-em-lote.md).
- **Ponto quente:** um link viral concentra leituras em uma chave (cache e CDN resolvem). No contador de cliques, pode virar contenção de escrita (agregar em lote).
- **Parâmetros de carga reais:** a distribuição de acesso por link (poucos muito populares) importa mais que a média.

## Aplicação no mundo real

- **Abuso é o problema número um de encurtadores públicos:** phishing e malware. Verificar URLs de destino contra listas de reputação (ex.: APIs de navegação segura), limitar criação por IP ou conta, permitir denúncia e **desativação**. Portanto, links **precisam** poder ser apagados, e o requisito "imutável" do exercício é irreal.
- **Privacidade:** cliques registram IP e user-agent, que são dados pessoais (LGPD/GDPR). Defina retenção e anonimização.
- **Borda:** redirecionamentos podem ser servidos por funções na CDN (*edge*), com o mapa em key-value distribuído na borda. É latência mínima para links populares.
- **Custos:** a estimativa de armazenamento corrigida (~36,5 TB em 10 anos, sem réplicas) mostra que o problema é mais de **leitura e abuso** do que de volume.

## No Projeto prático: sugestão de evolução por módulos

A §3 do bootstrap pede que cada módulo modifique o mesmo projeto e guarde as medições anteriores.

1. **Base:** API em TypeScript + PostgreSQL. `UNIQUE` no código. Geração por sequência → base62. k6 medindo p50, p95 e p99 de criação e redirecionamento.
2. **Concorrência:** `INSERT ... ON CONFLICT`, chave de idempotência, teste com requisições simultâneas.
3. **Cache:** Redis *cache-aside* no redirecionamento, TTL, medição de hit ratio e do p99 antes e depois.
4. **Escala horizontal:** 2+ instâncias sem estado atrás de um balanceador. Derrubar uma durante a carga.
5. **Réplica de leitura:** medir atraso e resolver ler as próprias escritas.
6. **Analytics assíncrono:** fila (RabbitMQ), worker idempotente, agregação por tempo do evento, DLQ.
7. **Proteção:** rate limiter (token bucket em Redis), validação de URL, lista de bloqueio.
8. **Observabilidade e SLO:** OpenTelemetry + Grafana, SLO de disponibilidade e latência do redirecionamento, alertas.
9. **Falhas:** Toxiproxy (latência e corte), failover do primário, restauração de backup.
10. **(Opcional) Sharding ou borda:** particionar por hash do código ou servir na borda, se a medição justificar.

## Armadilhas comuns

- Verificar e depois inserir sem restrição `UNIQUE`.
- Usar 301 e depois descobrir que o analytics não funciona.
- Gravar o contador de cliques de forma síncrona no caminho do redirecionamento.
- Códigos sequenciais expostos sem pensar em enumeração.
- Ignorar abuso (o serviço vira ferramenta de phishing).
- Confiar em estimativas sem conferir o cálculo (o erro de 10× do exemplo do SDI1).

## Perguntas de verificação

1. Calcule o tamanho mínimo do código para 365 bilhões de URLs com alfabeto de 62 símbolos.
2. 301 ou 302? Justifique para um produto que vende analytics.
3. Compare hash + colisão e base62 de ID quanto a colisões, previsibilidade e tamanho.
4. Por que "verificar se a URL existe e depois inserir" é inseguro sob concorrência?
5. Qual o erro no cálculo de armazenamento do exemplo do SDI1?

## Referências

- [SDI1, cap. 8, "Design a URL Shortener"]: escopo, estimativas, API, 301 × 302, modelo de dados, hash + colisão, base62, fluxos de encurtamento e redirecionamento, tópicos extras.
- [SDI1, cap. 7]: gerador de IDs usado no fluxo base62.
- [DDIA1, cap. 7, "Write Skew and Phantoms", pp. 246–251]: verificar e depois inserir.
- [DDIA1, cap. 5, "Reading Your Own Writes", pp. 162–164].
- [DDIA1, cap. 12, "The End-to-End Argument for Databases", pp. 516–520]: IDs de operação e idempotência.
