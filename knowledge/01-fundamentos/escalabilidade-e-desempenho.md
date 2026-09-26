---
titulo: Escalabilidade e desempenho
fontes: [DDIA1, SDI1]
status-validacao: base DDIA1 (Lente) + secundária (SDI1); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Escalabilidade e desempenho

> Escalabilidade não é um rótulo ("X escala"). É a resposta a uma pergunta concreta: se a carga crescer *desta forma*, quais são as opções para manter o desempenho? Para responder, é preciso antes descrever a carga com números e medir o desempenho como distribuição, não como média.

## Por que importa

Decisões de arquitetura partem de premissas sobre quais operações são frequentes e quais são raras. Se as premissas estiverem erradas, o esforço para escalar é, na melhor hipótese, desperdiçado e, na pior, prejudicial. Medir antes de escalar é o que separa engenharia de adivinhação.

## Conceitos-chave

### Parâmetros de carga

- **Parâmetro de carga** é o número que descreve a pressão sobre o sistema: requisições por segundo, proporção leitura/escrita, usuários simultâneos, taxa de acerto do cache, tamanho médio dos objetos. A escolha certa depende da arquitetura.
- **O caso extremo pode dominar.** Às vezes o gargalo não é a média, e sim uma cauda de casos grandes.
- **Exemplo clássico: a timeline do Twitter (dados de 2012).** Publicar é raro perto de ler a timeline (ordem de grandeza de milhares × centenas de milhares por segundo). O que torna o problema difícil é o **fan-out**: cada publicação precisa chegar a muitos seguidores. Há duas estratégias:
  1. **Fan-out na leitura:** gravar a publicação uma vez e montar a timeline no momento da leitura, juntando as postagens de quem o usuário segue. Escrita barata, leitura cara.
  2. **Fan-out na escrita:** ao publicar, inserir a postagem na "caixa de entrada" (cache de timeline) de cada seguidor. Leitura barata, escrita cara, e explosiva para quem tem milhões de seguidores.
  
  A solução adotada foi **híbrida**: fan-out na escrita para a maioria e na leitura para contas com muitos seguidores. A lição geral é que **a distribuição** (seguidores por usuário, ponderada pela frequência de postagem) é o parâmetro de carga decisivo, não a média.

### Medindo desempenho

- **Vazão (throughput)** importa em processamento em lote: registros por segundo ou tempo total do job.
- **Tempo de resposta** importa em sistemas online: o tempo entre o cliente enviar a requisição e receber a resposta. Ele inclui tempo de serviço, rede e filas.
- **Latência × tempo de resposta:** latência é o tempo em que a requisição fica *esperando* atendimento. Tempo de resposta é o total percebido pelo cliente. Os termos são usados como sinônimos, mas não são a mesma coisa.
- **Tempo de resposta é uma distribuição.** A mesma requisição varia a cada execução por causa de troca de contexto, perda de pacote, pausa de coleta de lixo (GC), falha de página etc.
- **Percentis > média.** A média não diz quantos usuários sofreram um atraso. A mediana (p50) é o tempo típico. p95, p99 e p999 revelam a cauda.
- **Latência de cauda importa para o negócio.** Os clientes mais lentos podem ser os mais valiosos, por exemplo os que têm mais dados. Otimizar percentis altíssimos (p9999) costuma ter custo alto e retorno decrescente.
- **Amplificação de cauda:** quando uma requisição do usuário depende de várias chamadas internas, basta uma lenta para tudo ficar lento. Quanto mais chamadas, maior a chance.
- **Filas dominam a cauda.** Poucas requisições lentas bloqueiam as seguintes (*head-of-line blocking*). Por isso o tempo de resposta deve ser medido **do lado do cliente**.
- **SLO/SLA usam percentis.** Exemplo típico: mediana abaixo de X ms, p99 abaixo de Y s, disponível 99,9% do tempo.
- **Agregação correta:** calcular média de percentis (entre janelas de tempo ou entre máquinas) é matematicamente inválido. O certo é somar os **histogramas**. Existem estruturas que estimam percentis com pouco custo: forward decay, t-digest, HdrHistogram.

### Abordagens para lidar com carga

- **Vertical (scale up) × horizontal (scale out).** Escalar horizontalmente distribui a carga em várias máquinas (*shared-nothing*). Uma máquina só é mais simples, mas máquinas topo de linha ficam caras. Na prática, arquiteturas boas misturam as duas.
- **Elástico × manual.** Autoescalar ajuda com carga imprevisível. Escalar manualmente é mais simples e traz menos surpresas operacionais.
- **Sem estado × com estado.** Distribuir serviços sem estado é fácil. Distribuir dados (com estado) traz muita complexidade. Por isso, por muito tempo, a regra foi manter o banco em um nó só até custo ou disponibilidade exigirem distribuí-lo.
- **Não existe arquitetura escalável genérica.** 100 mil requisições por segundo de 1 kB e 3 requisições por minuto de 2 GB têm a mesma vazão de dados e arquiteturas completamente diferentes.
- **Uma ordem de grandeza por vez.** Uma arquitetura adequada para certa carga dificilmente aguenta 10× mais. Espere repensar a cada ordem de grandeza.
- **Produto novo:** iterar rápido em funcionalidades vale mais do que escalar para uma carga hipotética.

## Trade-offs e decisões

| Decisão | A favor | Contra |
|---|---|---|
| Fan-out na escrita | Leitura rápida e previsível | Escrita cara; explode com contas muito seguidas |
| Fan-out na leitura | Escrita barata; sem trabalho desperdiçado com quem não lê | Leitura cara e variável |
| Escalar verticalmente | Simplicidade; sem coordenação distribuída | Custo cresce rápido; teto físico; ponto único de falha |
| Escalar horizontalmente | Capacidade e tolerância a falhas | Complexidade de dados distribuídos |
| Autoescala | Absorve picos imprevisíveis | Surpresas operacionais; tempo de reação |

## Aplicação no mundo real

- **Defina os parâmetros de carga do seu sistema** antes de desenhar a arquitetura: QPS de leitura e de escrita, tamanho dos objetos, distribuição de acesso (há itens muito mais acessados que outros?).
- **Meça com histogramas.** Prometheus (histogramas e `histogram_quantile`) e OpenTelemetry expõem percentis. Nunca faça média de percentis em dashboards.
- **Teste de carga em modelo aberto.** O DDIA alerta que o gerador de carga não pode esperar a resposta anterior para enviar a próxima, senão as filas ficam artificialmente curtas. No k6, isso corresponde a usar *executors* de taxa de chegada (ex.: `constant-arrival-rate`) em vez de apenas usuários virtuais em loop fechado. Confirme os detalhes na documentação oficial do k6.
- **Orçamento de latência:** ao compor serviços, some e multiplique as caudas. Três chamadas em série com p99 de 100 ms não dão p99 de 100 ms no total.

## No Projeto prático (encurtador de links)

- **Parâmetros de carga:** proporção redirecionamento/criação (tipicamente muito mais leitura), distribuição de acessos por link (poucos links muito populares) e tamanho da tabela de links.
- **Medição:** p50/p95/p99 do redirecionamento medidos no cliente (k6), com histograma no servidor.
- **Decisão guiada por medição:** só introduzir cache quando o p99 de leitura ou a carga no banco justificarem.

## Armadilhas comuns

- Relatar só o tempo médio de resposta.
- Medir no servidor e ignorar o tempo de fila percebido pelo cliente.
- Gerador de carga em loop fechado, que mascara a formação de filas.
- Calcular média de percentis de várias máquinas.
- Escalar para uma carga imaginada antes de validar o produto.

## Perguntas de verificação

1. Por que "o sistema X é escalável" é uma afirmação vazia? Como reformulá-la?
2. Explique fan-out na escrita e na leitura. Em que distribuição de seguidores cada um vence?
3. Por que p99 é mais informativo do que a média? E por que otimizar p9999 pode não valer a pena?
4. O que é amplificação de latência de cauda?
5. Qual erro de metodologia de teste de carga o modelo de chegada aberto evita?

## Referências

- [DDIA1, cap. 1, "Scalability", pp. 10–18]: parâmetros de carga, exemplo do Twitter, percentis, SLO/SLA, head-of-line blocking, amplificação de cauda, scale up × scale out, elasticidade.
- [SDI1, cap. 1, "Scale from Zero to Millions of Users"]: escala vertical × horizontal no contexto de uma aplicação web (ver [escalando uma aplicação web](../05-componentes/escalando-uma-aplicacao-web.md)).
