---
titulo: "Estudo de caso: feed de notícias (timeline)"
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) convergente com o DDIA1 (Lente; exemplo do Twitter); contém divergência registrada
atualizado: 2026-09-24
---

# Estudo de caso: feed de notícias (timeline)

> Um usuário publica e seus amigos ou seguidores veem as publicações na página inicial, em ordem cronológica reversa (ou ranqueada). O problema central é o **fan-out**: entregar cada publicação a muitos leitores. É o mesmo problema da timeline do Twitter que abre o DDIA1. As duas fontes **convergem** na solução híbrida: *push* para a maioria, *pull* para contas com muitos seguidores.

## Por que importa

Aparece em redes sociais, notificações de atividade, dashboards de eventos e caixas de entrada. É o exemplo canônico do trade-off entre **trabalho no caminho de escrita** (pré-computar) e **trabalho no caminho de leitura** (calcular sob demanda), e de como a **distribuição** da carga (seguidores por usuário) decide a arquitetura.

## Escopo do exercício (SDI1)

- Web e mobile.
- Publicar e ver o feed dos amigos.
- Ordem **cronológica reversa**.
- Até 5.000 amigos.
- 10 milhões de DAU.
- Mídia (imagem e vídeo).

## Desenho de alto nível

- **APIs:** `POST /v1/me/feed` (publicar, com `content` e token) e `GET /v1/me/feed` (ler).
- **Publicação:** balanceador → servidores web → **serviço de posts** (grava no banco e no cache), **serviço de fan-out** (empurra para os feeds dos amigos, guardados em cache) e **serviço de notificação**.
- **Leitura:** balanceador → servidores web → **serviço de feed**, que lê do **cache de feed** (IDs das publicações).

## Aprofundamentos

- **Servidores web:** autenticação e **rate limit** de publicações (anti-spam).
- **Fan-out:**

| Modelo | Como | A favor | Contra |
|---|---|---|---|
| **Na escrita (push)** | Ao publicar, inserir no cache de feed de cada amigo | Feed em tempo real, leitura rápida | Usuários com muitos amigos geram **chave quente** e lentidão; desperdício com usuários inativos |
| **Na leitura (pull)** | Ao abrir o feed, buscar as publicações recentes de quem segue | Sem desperdício com inativos, sem chave quente na escrita | Leitura lenta |

- **Híbrido (adotado):** push para a maioria, **pull para celebridades** (seguidores buscam sob demanda).
- **Serviço de fan-out, passo a passo:**
  1. buscar os IDs dos amigos no **banco de grafo**;
  2. filtrar pelas configurações (silenciados, compartilhamento seletivo) usando o cache de usuários;
  3. enviar (lista de amigos, ID da publicação) para uma **fila**;
  4. **workers** gravam no **cache de feed** como mapeamento (ID da publicação, ID do usuário). **Só IDs**, com limite configurável de itens (quase ninguém rola milhares de posts, então o *miss* é raro).
- **Leitura detalhada:**
  1. o serviço busca a lista de IDs no cache de feed;
  2. **hidrata** com objetos de usuário e de post vindos dos caches;
  3. devolve JSON.
  
  Mídia servida pela **CDN**.
- **Cache em 5 camadas:** feed (IDs), conteúdo (posts, com cache quente para os populares), grafo social, ações (curtiu? respondeu?) e contadores (curtidas, respostas, seguidores).
- **Tópicos extras:** escalar o banco (vertical/horizontal, SQL/NoSQL, réplicas, sharding, modelos de consistência), camada sem estado, vários datacenters, filas, métricas (QPS no pico, latência ao atualizar o feed).

## Convergência com o DDIA1

- O DDIA1 usa **exatamente este problema** (timeline do Twitter, dados de 2012) para definir **parâmetros de carga**:
  - publicar é raro perto de ler, então vale fazer mais trabalho na escrita;
  - mas a **distribuição de seguidores** (algumas contas com dezenas de milhões) torna o fan-out na escrita explosivo;
  - a solução foi a mesma abordagem híbrida.
- O DDIA1 também formaliza o feed como **view materializada** de um join entre tabelas (posts ⋈ seguidores), mantida por processamento de fluxo (join tabela–tabela). Postar ou apagar muda os feeds dos seguidores. Seguir ou deixar de seguir adiciona ou remove posts. Ver [processamento de fluxo](../04-dados-derivados/processamento-de-fluxo.md).
- **Fronteira escrita/leitura:** o cache de feed é o ponto onde os dois caminhos se encontram, e o híbrido **desloca essa fronteira de forma diferente para celebridades e para usuários comuns** (DDIA1, cap. 12).

## Divergências entre as fontes

- **"Hashing consistente ajuda a mitigar o problema da chave quente" (SDI1):** para o fan-out de uma **única conta muito seguida**, o DDIA1 é explícito: hashing **não resolve** uma chave única quente (ela sempre cai na mesma partição). A mitigação real é o próprio modelo híbrido (pull para celebridades), ou dividir a chave na aplicação. O hashing consistente ajuda a distribuir **muitas** chaves, não **uma**.
- **Causalidade e privacidade:** o DDIA1 alerta para dependências causais entre sistemas. Se alguém é bloqueado e em seguida publica, um fan-out fora de ordem pode entregar o post a quem não devia. O filtro por configurações deve usar estado atualizado.

## Trade-offs e decisões

- **Push × pull × híbrido:** conforme a distribuição de seguidores e a proporção de usuários ativos.
- **Guardar IDs × objetos no cache de feed:** memória contra custo de hidratação.
- **Cronológico × ranqueado:** o ranqueamento exige outro pipeline (features, modelo) e muda a estratégia de cache.
- **Consistência:** feeds são tipicamente eventualmente consistentes. Aceitável, com exceções como ler as próprias escritas: o autor precisa ver o próprio post imediatamente.

## Aplicação no mundo real

- **Estruturas de cache:** listas ou sorted sets limitados por usuário (ex.: Redis) guardam os IDs recentes do feed.
- **Paginação por cursor** (ID ou timestamp do último item), não por offset.
- **Contadores em alta escala:** incrementos atômicos ou agregados em lote. Contadores exatos em tempo real custam caro.
- **Grafo social:** muitas empresas mantêm o grafo em bancos relacionais bem indexados ou em stores especializados. Um banco de grafo não é obrigatório. Ver [modelos de dados](../02-dados/modelos-de-dados-e-consulta.md).

## No Projeto prático (encurtador de links)

- **Feed de atividade do dono:** "seus links mais clicados hoje" e "novos cliques". Discuta se vale pré-computar (fan-out na escrita a partir do agregador de cliques) ou calcular na leitura (consulta agregada). Com poucos donos e muitos cliques, pré-computar por dono via fluxo é o análogo do push.
- **Exercício de raciocínio:** um dono com um link viral (milhões de cliques por hora) é a "celebridade". Onde fica o ponto quente e como o híbrido se aplica?

## Armadilhas comuns

- Fan-out na escrita para contas com milhões de seguidores.
- Guardar objetos completos no cache de feed.
- Paginação por offset em feeds que mudam o tempo todo.
- Não garantir que o autor veja o próprio post (ler as próprias escritas).

## Perguntas de verificação

1. Compare fan-out na escrita e na leitura. Qual parâmetro de carga decide entre eles?
2. Por que o híbrido trata celebridades de forma diferente?
3. Por que o cache de feed guarda só IDs?
4. Por que hashing consistente não resolve a chave quente de uma celebridade?
5. Como o feed pode ser visto como uma view materializada?

## Referências

- [SDI1, cap. 11, "Design a News Feed System"]: escopo, APIs, publicação e leitura, fan-out push/pull/híbrido, serviço de fan-out, hidratação, cache em 5 camadas, tópicos de escala.
- [DDIA1, cap. 1, "Describing Load", pp. 11–13]: exemplo do Twitter, parâmetros de carga, híbrido.
- [DDIA1, cap. 11, "Table-table join (materialized view maintenance)", pp. 474–475].
- [DDIA1, cap. 12, "Materialized views and caching", pp. 510–511]: fronteira entre caminhos de escrita e leitura.
- [DDIA1, cap. 6, "Skewed Workloads and Relieving Hot Spots", p. 205].
