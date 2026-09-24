---
titulo: Falhas em sistemas distribuídos (rede, relógios, pausas)
fontes: [DDIA1]
status-validacao: base primária única (DDIA1); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Falhas em sistemas distribuídos: rede, relógios e pausas

> Em uma máquina, o software normalmente funciona ou quebra por inteiro. Em um sistema distribuído, **partes quebram enquanto outras funcionam**: são as *falhas parciais*, e elas são **não determinísticas**. A rede perde e atrasa mensagens sem limite, relógios discordam e voltam no tempo, processos param por segundos sem perceber. Nenhum nó sabe a verdade sozinho.

## Por que importa

Toda chamada entre serviços, todo banco replicado e toda trava distribuída vive neste mundo. Quem ignora essas falhas escreve código que funciona em teste e corrompe dados em produção, geralmente em silêncio.

## Conceitos-chave

### Falhas parciais

- **Supercomputador (HPC) × nuvem.** Em HPC, se algo falha, para tudo e recomeça do último checkpoint. É como um único computador. Serviços online não podem parar, rodam em hardware comum e em rede IP. Em sistemas com milhares de nós, **sempre há algo quebrado**. É preciso construir um **sistema confiável a partir de partes não confiáveis**, como o TCP faz sobre o IP. Mesmo assim existe um limite: o TCP esconde perda e reordenação, não o atraso.
- **Paranoia compensa.** Considere falhas improváveis e **provoque-as em teste**.

### Redes não confiáveis

- **Rede assíncrona de pacotes:** nenhuma garantia de *quando* ou *se* a mensagem chega. Sem resposta, é **impossível distinguir** entre:
  - a requisição se perdeu;
  - ela está numa fila;
  - o nó remoto caiu;
  - o nó remoto está pausado;
  - ele processou, mas a resposta se perdeu;
  - ele processou, mas a resposta está atrasada.
- **Na prática, falhas de rede são comuns.** Um estudo citado encontrou ~12 falhas por mês em um datacenter médio. Redundância de equipamento não protege contra configuração errada. Também existem links que funcionam **em um só sentido**.
- **Tratar falha de rede** não significa necessariamente tolerá-la: mostrar erro ao usuário é válido. Mas o comportamento precisa ser **definido e testado**. Sem isso, clusters travam para sempre ou apagam dados.
- **Detectar falhas é difícil.**
  - Sinais explícitos ajudam quando existem: TCP RST ao conectar numa porta sem processo, script avisando que o processo morreu, informação do switch, ICMP de destino inalcançável.
  - Mas o TCP entregar o pacote não garante que a aplicação processou. **Só a resposta positiva da aplicação confirma o sucesso.**
- **Timeouts:**
  - longos demoram a detectar a falha;
  - curtos declaram mortos nós que estão só lentos, o que gera **ações duplicadas** e **falha em cascata** (a carga do "morto" vai para os vizinhos já sobrecarregados).
  - Em uma rede com atraso máximo *d* e processamento máximo *r*, bastaria 2d + r. **Redes e servidores reais não têm limites.**
- **Filas são a principal fonte de variação:** congestionamento no switch, fila do sistema operacional quando a CPU está ocupada, pausas de VM (*vizinho barulhento*), controle de fluxo do TCP, retransmissões.
- **UDP × TCP:** UDP evita retransmissão e controle de fluxo. Serve quando dado atrasado não vale nada (voz, vídeo): a "retentativa" acontece na camada humana.
- **Escolha de timeout:** meça a distribuição de *round-trip* ao longo do tempo e escolha de forma empírica, ou use detectores adaptativos como o **Phi Accrual** (Akka, Cassandra).
- **Por que não uma rede "garantida" como a telefonia?** Circuitos reservam banda fixa e dão atraso limitado. Pacotes compartilham banda de forma dinâmica e aproveitam melhor o link para tráfego em rajadas. **Atraso variável é um trade-off de custo** (utilização × previsibilidade), não uma lei da natureza. Em nuvem multi-inquilino e internet, não há QoS que ajude: **assuma atraso ilimitado**.

### Relógios não confiáveis

**Dois tipos de relógio**
- **Relógio de parede** (*time-of-day*): data e hora sincronizadas por NTP. Pode **saltar para trás** quando é corrigido e costuma ignorar segundos intercalares. **Não serve para medir duração.**
- **Relógio monotônico:** só avança. Serve para medir **duração** (timeouts, tempo de resposta). O valor absoluto não significa nada e não se compara entre máquinas.

**Sincronização é imprecisa**
- O quartzo deriva (o Google assume 200 ppm, ~17 s por dia sem ressincronizar).
- O NTP pode resetar o relógio, pode ficar bloqueado por firewall sem ninguém notar, é limitado pelo atraso da rede (~35 ms na internet, com picos de ~1 s) e servidores NTP podem estar errados.
- Segundos intercalares já derrubaram grandes sistemas (a mitigação é o *smearing*).
- Em VMs, o relógio "salta" nas pausas.
- Em dispositivos de usuários, o relógio pode estar deliberadamente errado.
- Precisão alta (ex.: 100 µs para regulação financeira) é possível com GPS e PTP, mas custa caro.

**Relógio errado falha em silêncio.** Uma CPU defeituosa derruba a máquina. Um relógio defeituoso **perde dados sem erro**. Se o software depende de relógios sincronizados, **monitore o desvio** e remova nós fora do limite.

**Ordenar eventos por timestamp é perigoso**
- **LWW** com relógios de parede descarta escritas:
  - um nó com relógio atrasado não consegue sobrescrever o valor de um nó adiantado;
  - LWW não distingue escritas sequenciais de concorrentes;
  - dois nós podem gerar o mesmo timestamp.
- Mesmo com NTP bem ajustado, um pacote pode "chegar antes de ser enviado".
- **Relógios lógicos** (contadores) ordenam eventos com segurança. Ver [consistência e consenso](consistencia-e-consenso.md).

**Leitura de relógio é um intervalo, não um ponto.** Com NTP pela internet, a incerteza é de dezenas de milissegundos, e os dígitos de microssegundo não significam nada. O **TrueTime** do Google Spanner expõe o intervalo [mais cedo, mais tarde]. O Spanner **espera o tamanho do intervalo** antes de confirmar uma transação, para que os intervalos não se sobreponham, e usa GPS e relógio atômico por datacenter (~7 ms de incerteza).

**IDs globais crescentes** exigem coordenação. Geradores como o Snowflake distribuem blocos de IDs, mas **não garantem ordem causal** (ver [gerador de IDs únicos](../06-estudos-de-caso/gerador-de-ids-unicos.md)).

### Pausas de processo

- **Exemplo:** um líder confere se o *lease* (trava com prazo) ainda vale e então processa a requisição. Se a thread pausar 15 s entre a checagem e o processamento, o lease já expirou e **outro nó já é líder**, mas a thread não sabe.
- **Causas de pausa arbitrária:**
  - coleta de lixo *stop-the-world* (já registrada em minutos);
  - suspensão ou migração de VM;
  - laptop fechado;
  - troca de contexto e *steal time*;
  - I/O síncrono (inclusive carregamento de classes e discos de rede);
  - swap e *thrashing*;
  - `SIGSTOP`.
- **Um nó precisa assumir que pode ser pausado a qualquer momento**, inclusive no meio de uma função, e que o mundo segue sem ele.
- **Tempo real rígido** (airbag, aeronave) exige sistema operacional de tempo real, bibliotecas com pior caso documentado e memória restrita. É caro e reduz a vazão. **Para servidores, não compensa.**
- **Mitigar GC:** tratar a pausa como indisponibilidade planejada (tirar o nó da rotação antes do GC) ou reiniciar processos periodicamente, como em um deploy gradual.

### Conhecimento, verdade e mentira

- **A verdade é definida pela maioria.** Um nó não pode confiar no próprio julgamento: pode estar meio desconectado ou ter saído de uma pausa de GC sem saber. Decisões (inclusive "fulano morreu") exigem **quórum**. Maioria absoluta: só pode existir uma por vez.
- **"O escolhido"** (líder, dono da trava, dono do nome de usuário) pode ter sido destituído sem saber. Caso real citado: uma trava distribuída mal implementada, em que o cliente pausado volta achando que tem o lease e **corrompe um arquivo**.
- **Fencing token:** a cada concessão de trava, um número crescente. O **recurso** rejeita escritas com token menor que o último visto. A checagem precisa estar no recurso, não no cliente. No ZooKeeper, `zxid` ou `cversion` servem como token.
- **Falhas bizantinas** são nós que "mentem" (corrupção por radiação, participantes desonestos, como em blockchains). Protocolos tolerantes exigem mais de 2/3 dos nós honestos e são caros. **Em um datacenter próprio, assume-se que não há falhas bizantinas.** Contra atacantes, a proteção são autenticação, controle de acesso e criptografia. Contra clientes web, validação de entrada e autoridade do servidor.
- **Proteções contra "mentiras fracas":** checksums na aplicação, validação de entrada (faixa, tamanho), NTP com vários servidores e descarte de valores discrepantes.

### Modelos de sistema

| Tempo | Descrição |
|---|---|
| Síncrono | Atraso, pausa e erro de relógio limitados. Irreal. |
| **Parcialmente síncrono** | Na maior parte do tempo se comporta bem, às vezes os limites estouram. **O mais realista.** |
| Assíncrono | Sem suposições de tempo, sem relógio. Muito restritivo. |

| Falha de nó | Descrição |
|---|---|
| Crash-stop | Cai e nunca volta |
| **Crash-recovery** | Cai e pode voltar; o disco sobrevive, a memória não. **O mais útil**, junto com o parcialmente síncrono. |
| Bizantina | Qualquer comportamento |

- **Correção = propriedades.** Exemplo, para tokens de trava: unicidade, sequência monotônica, disponibilidade.
- **Segurança (safety) × vivacidade (liveness):**
  - **segurança:** "nada de ruim acontece". Quando violada, dá para apontar o instante, e o dano não se desfaz. Precisa valer **sempre**;
  - **vivacidade:** "algo bom acaba acontecendo". Pode ter ressalvas (ex.: se a maioria estiver viva). **Consistência eventual é uma propriedade de vivacidade.**
- **Modelo × realidade:** discos corrompem, nós "esquecem" o que gravaram (e isso quebra quóruns). A implementação precisa de código para o "impossível", nem que seja chamar um humano. **Análise teórica e teste empírico são igualmente importantes.**
- **Conselho final:** se der para resolver em **uma máquina**, resolva. Distribua quando precisar de tolerância a falhas, latência geográfica ou escala.

## Trade-offs e decisões

- **Timeout curto × longo:** detecção rápida contra falsos positivos e cascata.
- **Previsibilidade × utilização:** recursos reservados (latência garantida, caro) contra compartilhados (barato, variável).
- **Relógio físico × lógico:** o físico dá significado humano ("quando"); o lógico dá ordem correta.
- **Tolerar × simplesmente falhar:** às vezes mostrar erro é a resposta certa, desde que seja um comportamento definido.

## Aplicação no mundo real

- **Toda chamada de rede tem timeout**, retry com *backoff exponencial e jitter*, e idempotência no destino.
- **Circuit breakers e bulkheads** (ex.: bibliotecas de resiliência, ou configuração em service mesh) evitam que um dependente lento derrube todo mundo.
- **Durações sempre com relógio monotônico:** `performance.now()` e `process.hrtime` no Node.js, `System.nanoTime()` em Java.
- **Não use timestamp de parede para ordenar escritas concorrentes.** Use versões ou sequências do banco.
- **Travas distribuídas** (ex.: Redis, etcd, ZooKeeper) exigem fencing no recurso protegido. Uma trava sem fencing é só uma otimização de eficiência, não uma garantia de correção.
- **Testes de caos:** ferramentas como Toxiproxy (latência e corte de rede entre contêineres) e `tc netem` (atraso e perda de pacotes no Linux) reproduzem essas falhas localmente. Confirme o uso na documentação oficial.
- **Monitorar desvio de NTP** (ex.: métricas do `chrony`) em qualquer sistema que dependa de relógios.

## No Projeto prático (encurtador de links)

- **Timeout e retry na criação de link:** o cliente não sabe se o link foi criado quando a resposta se perde. Isso exige chave de idempotência para não criar dois códigos.
- **Expiração de links** (se existir): use o relógio do banco, não o de cada instância, e aceite alguns segundos de imprecisão.
- **Experimento sugerido:** com Toxiproxy entre a API e o PostgreSQL, injetar 500 ms de latência e depois cortar a conexão. Observar p99, erros e se há criações duplicadas com e sem idempotência.
- **Worker de cliques pausado** (ex.: `docker pause` no contêiner): verificar se outro worker assume e se algum lote é processado duas vezes.

## Armadilhas comuns

- Chamada de rede sem timeout.
- Retry imediato e infinito (tempestade de retries).
- Medir duração com `Date.now()` ou relógio de parede.
- LWW com relógios de parede para dados importantes.
- Trava distribuída sem fencing token.
- Achar que "o nó está vivo porque o TCP conectou".

## Perguntas de verificação

1. Liste as situações indistinguíveis quando uma requisição fica sem resposta.
2. Por que um timeout curto pode causar falha em cascata?
3. Quando usar relógio monotônico e quando usar relógio de parede?
4. Por que LWW com timestamps pode perder escritas **não** concorrentes?
5. Explique o problema da trava com lease e pausa de GC, e como o fencing token o resolve.
6. Diferencie propriedades de segurança e de vivacidade. Em que categoria está a consistência eventual?

## Referências

- [DDIA1, cap. 8, "Faults and Partial Failures", pp. 274–277]: falhas parciais, HPC × nuvem, sistemas confiáveis a partir de partes não confiáveis.
- [DDIA1, cap. 8, "Unreliable Networks", pp. 277–287]: falhas de rede na prática, detecção, timeouts, filas, TCP × UDP, redes síncronas × assíncronas.
- [DDIA1, cap. 8, "Unreliable Clocks", pp. 287–299]: relógio de parede × monotônico, sincronização, ordenação por timestamp, intervalos de confiança, TrueTime, pausas de processo, tempo real.
- [DDIA1, cap. 8, "Knowledge, Truth, and Lies", pp. 300–311]: quóruns, líder e trava, fencing tokens, falhas bizantinas, modelos de sistema, segurança × vivacidade.
