---
titulo: "Estudo de caso: rate limiter (limitador de taxa)"
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) + princípios do DDIA1 (Lente); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Estudo de caso: rate limiter (limitador de taxa)

> Um rate limiter controla quantas requisições um cliente (usuário, IP, chave de API) pode fazer em um período, e rejeita o excesso com **HTTP 429**. Serve para proteger contra abuso e DoS, controlar custo (principalmente com APIs pagas de terceiros) e evitar sobrecarga. As decisões centrais são **onde** limitar, **qual algoritmo** usar e **como compartilhar contadores** entre vários servidores sem condições de corrida.

## Por que importa

Quase toda API pública precisa de limites, e quase todo sistema interno se beneficia deles: login (força bruta), envio de e-mails, criação de contas, chamadas caras. É também um ótimo exercício de concorrência distribuída em escala pequena.

## Requisitos típicos

- Limitar com precisão.
- **Baixa latência:** o limitador não pode deixar a API mais lenta.
- Pouca memória.
- **Distribuído:** vários servidores compartilham os contadores.
- Erros claros para quem foi limitado.
- **Tolerância a falhas:** se o armazenamento de contadores cair, o sistema não pode cair junto (decisão explícita: *fail-open* ou *fail-closed*).

## Onde colocar

- **No cliente:** não é confiável, porque é fácil de forjar e você não controla o cliente.
- **No servidor**, dentro da aplicação: controle total sobre o algoritmo.
- **Middleware / API gateway:** junto com autenticação, TLS e listas de IP. Faz sentido se já existe um gateway.
- **Critérios:** stack atual, algoritmo necessário (um gateway de terceiros pode limitar as opções) e esforço de engenharia (construir leva tempo, e um gateway comercial pode ser melhor).

## Algoritmos

| Algoritmo | Como funciona | A favor | Contra |
|---|---|---|---|
| **Token bucket** | Balde com capacidade C, reabastecido a R tokens/s; cada requisição consome um token | Simples, pouca memória, **permite rajadas curtas** (usado por AWS e Stripe, segundo o SDI1) | Ajustar C e R não é trivial |
| **Leaky bucket** | Fila FIFO de tamanho fixo, drenada a uma taxa constante | Saída estável, memória limitada (Shopify, segundo o SDI1) | Rajada enche a fila de requisições antigas, e as novas são rejeitadas |
| **Janela fixa** | Contador por janela (ex.: por minuto), zera na virada | Simples, pouca memória | **Rajada na borda** entre janelas deixa passar até 2× o limite |
| **Log em janela deslizante** | Guarda o timestamp de cada requisição (ex.: sorted set) e descarta os antigos | Preciso em qualquer janela | Muita memória (guarda até requisições rejeitadas) |
| **Contador em janela deslizante** | Atual + anterior × fração de sobreposição | Suaviza picos, pouca memória | Aproximado: supõe que a janela anterior foi uniforme. O SDI1 cita um experimento da Cloudflare com erro de ~0,003% |

- **Quantos baldes?** Um por regra × identidade (ex.: postar 1/s, adicionar 150 amigos/dia e curtir 5/s dão 3 baldes por usuário). Um por IP. Ou um global.

## Arquitetura

- **Contadores em memória** com expiração (ex.: Redis com `INCR` + `EXPIRE`), não em banco em disco, por causa da latência.
- **Fluxo:** o middleware lê a regra (em cache), consulta o contador, decide, encaminha à API ou responde 429.
- **Regras** em arquivos de configuração (o exemplo do SDI1 é o formato do limitador open source da Lyft: domínio, descritor, unidade e requisições por unidade), carregadas em cache por workers.
- **Resposta ao cliente:** `429 Too Many Requests` com cabeçalhos de limite, restante e tempo para tentar de novo. Requisições limitadas podem ser **descartadas ou enfileiradas** para depois (ex.: pedidos durante sobrecarga).

### Em ambiente distribuído

- **Condição de corrida:** ler o contador, checar e escrever é um ciclo de ler, modificar e escrever, e duas requisições concorrentes perdem incrementos. É o problema de **perda de atualização** (ver [transações](../03-sistemas-distribuidos/transacoes.md)).
  - Soluções: **operação atômica** (incremento atômico que já devolve o valor; script executado atomicamente no servidor de cache, como Lua no Redis; sorted sets).
  - Evite travas, que deixam tudo lento.
- **Sincronização entre vários limitadores:** sessões *sticky* não escalam. Use um **armazenamento central de contadores** compartilhado.
- **Desempenho:**
  - limitar na **borda**, perto do usuário (várias localizações);
  - sincronizar entre regiões com **consistência eventual**, aceitando pequena imprecisão.
- **Monitoramento:** o algoritmo e as regras estão funcionando? Regras rígidas demais derrubam requisições válidas. Picos (ex.: *flash sale*) podem pedir token bucket.

### Tópicos extras

- **Limite rígido × flexível:** nunca passar do limite, ou tolerar excesso por curto período.
- **Camadas:** aplicação (HTTP, camada 7) ou rede (IP, camada 3, ex.: iptables).
- **Boas práticas do cliente:** cache local, respeitar os limites, tratar 429, **backoff** nos retries.

## Conexões com os fundamentos (DDIA1)

- **Contador compartilhado = perda de atualização:** operações atômicas são a solução preferida quando o problema pode ser expresso nelas.
- **Tempo:** janelas dependem do relógio. Em vários servidores, use o relógio do armazenamento central ou aceite o desvio. Timestamps de dispositivos não são confiáveis. Ver [falhas em sistemas distribuídos](../03-sistemas-distribuidos/falhas-em-sistemas-distribuidos.md).
- **Falha do armazenamento de contadores:** *fail-open* mantém a disponibilidade e aceita abuso temporário; *fail-closed* protege e causa indisponibilidade. É uma escolha de negócio explícita, e a trilha pede que ela seja registrada como decisão.
- **Contagem aproximada entre regiões** é pontualidade sacrificada. A integridade que importa é não deixar passar abuso significativo. Ver [integração de dados e correção](../04-dados-derivados/integracao-de-dados-e-correcao.md).

## Aplicação no mundo real

- **Soluções prontas:** limitação em gateways e proxies (Nginx `limit_req`, Envoy, Kong, API Gateway da AWS), WAF e regras de rate limit na CDN (ex.: Cloudflare). Em Node.js, bibliotecas de rate limit com armazenamento em Redis. Confirme parâmetros e semântica na documentação oficial.
- **Cabeçalhos padronizados:** há trabalho de padronização no IETF para cabeçalhos `RateLimit`. Os `X-RateLimit-*` do exemplo do SDI1 são convenção de mercado. Verifique o estado atual.
- **Limites por plano** (free × pago) são configuração por identidade de cliente.

## No Projeto prático (encurtador de links)

- **Onde limitar:**
  - criação de links (anti-spam): ex.: 10/min por IP anônimo, 100/min por chave de API;
  - redirecionamento: limite alto por IP para conter robôs.
- **Implementação didática:** token bucket em Redis com script atômico. **Experimento:** comparar com a versão ingênua (ler, modificar, escrever) sob 200 requisições concorrentes e contar quantas passaram além do limite.
- **Decisão a registrar:** o que fazer se o Redis cair (*fail-open* no redirecionamento, *fail-closed* na criação?).

## Armadilhas comuns

- Contador com ler, modificar e escrever não atômico.
- Janela fixa sem considerar a rajada na borda.
- Rate limit só no cliente.
- Não devolver 429 com o tempo para tentar de novo, o que faz o cliente martelar.
- Não definir o comportamento quando o armazenamento de contadores falha.

## Perguntas de verificação

1. Compare token bucket e leaky bucket quanto a rajadas.
2. Por que a janela fixa pode deixar passar o dobro do limite?
3. Qual condição de corrida aparece em um limitador distribuído, e como resolvê-la sem travas?
4. *Fail-open* ou *fail-closed*: qual você escolheria para login? E para redirecionamento de links?

## Referências

- [SDI1, cap. 4, "Design a Rate Limiter"]: requisitos, onde colocar, algoritmos (token bucket, leaky bucket, janela fixa, log e contador em janela deslizante), arquitetura com Redis, regras, cabeçalhos, 429, corrida, sincronização, otimização, monitoramento.
- [DDIA1, cap. 7, "Preventing Lost Updates", pp. 242–246]: operações atômicas × ler, modificar e escrever.
- [DDIA1, cap. 8, "Unreliable Clocks", pp. 287–299]: relógios e janelas de tempo.
