---
titulo: Codificação, evolução de esquema e fluxos de dados
fontes: [DDIA1]
status-validacao: base primária única (DDIA1); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Codificação, evolução de esquema e fluxos de dados

> Sistemas mudam aos poucos. Durante um deploy gradual, versões antigas e novas do código convivem, e os dados sobrevivem ao código que os escreveu. Para isso funcionar, o formato dos dados precisa ser **compatível para trás** (código novo lê dado antigo) e **para frente** (código antigo lê dado novo), qualquer que seja o caminho: banco, API ou fila.

## Por que importa

Deploy sem downtime, apps móveis que o usuário demora a atualizar e serviços mantidos por times diferentes dependem de compatibilidade entre versões. Quebrar a compatibilidade gera erros que só aparecem durante o deploy ou meses depois, ao ler um dado antigo.

## Conceitos-chave

### Codificação

- **Codificar** (serializar, *marshalling*) é traduzir estruturas em memória, cheias de ponteiros, para uma sequência de bytes autocontida. **Decodificar** é o inverso. No DDIA, "serialização" é evitada porque o termo tem outro sentido em transações.
- **Formatos nativos da linguagem** (Java `Serializable`, Python `pickle`, Ruby `Marshal`):
  - prendem os dados a uma linguagem;
  - são **risco de segurança**: decodificar bytes arbitrários pode instanciar classes arbitrárias e levar a execução remota de código;
  - ignoram versionamento e costumam ser ineficientes.
  
  Use só para dados muito transitórios.
- **JSON, XML e CSV:** textuais, universais e bons para troca entre organizações. Têm problemas sutis:
  - **números:** JSON não distingue inteiro de ponto flutuante, e inteiros acima de 2^53 perdem precisão em JavaScript. Por isso APIs devolvem IDs de 64 bits também como string;
  - **binário:** não há suporte a bytes, e o Base64 aumenta o tamanho em ~33%;
  - **esquemas:** JSON Schema e XML Schema são opcionais e complexos;
  - **CSV:** sem esquema e com escape ambíguo.
- **"JSON binário"** (MessagePack, BSON etc.): ganha pouco espaço porque ainda carrega o nome de cada campo em todo registro.

### Formatos binários com esquema

- **Protocol Buffers e Thrift:** o esquema (IDL) define campos com **números de tag**. Os dados codificados carregam só tag + tipo + valor, sem nome de campo. Há geração de código para várias linguagens.
- **Regras de evolução com tags:**
  - pode **renomear** campos (o nome não vai nos dados); **nunca mude a tag**;
  - **adicionar campo:** tag nova, e o campo precisa ser opcional ou ter valor padrão. Código antigo ignora tags desconhecidas (compatível para frente); código novo lê dado antigo (compatível para trás);
  - **remover campo:** só se for opcional, e **nunca reutilize a tag**;
  - **mudar tipo:** arriscado. Um int64 lido como int32 é truncado;
  - no Protocol Buffers, `repeated` permite transformar um campo de valor único em lista.
- **Avro:** sem tags. Os dados são só valores concatenados na ordem do esquema, o que dá o formato mais compacto. A evolução funciona com **dois esquemas**: o do **escritor** e o do **leitor**, que só precisam ser *compatíveis*. A biblioteca resolve as diferenças pelo nome do campo:
  - campo presente só no escritor é ignorado;
  - campo presente só no leitor recebe o valor padrão;
  - só se pode adicionar ou remover campos **com valor padrão**;
  - `null` é explícito, via *union*.
- **Como o leitor descobre o esquema do escritor:**
  - em arquivo grande, o esquema vai uma vez no cabeçalho;
  - em banco com registros avulsos, um número de versão no registro aponta para um **registro de esquemas**;
  - em conexão de rede, o esquema é negociado na abertura.
- O Avro é **amigável a esquemas gerados dinamicamente** (ex.: exportar tabelas de um banco), justamente por não depender de tags atribuídas à mão.
- **Vantagens dos esquemas:** compactação; documentação que não fica desatualizada, porque é necessária para decodificar; checagem de compatibilidade **antes** do deploy; tipagem estática via geração de código. Dão a flexibilidade do schema-on-read com mais garantias.

### Modos de fluxo de dados

**1. Via banco de dados**
- Escrever no banco é "mandar uma mensagem para o seu eu futuro". A compatibilidade para trás é obrigatória.
- Com deploy gradual, código **antigo** pode ler dado escrito por código **novo**, então a compatibilidade para frente também é necessária.
- **Armadilha:** código antigo lê um registro com um campo novo que não conhece, atualiza outro campo e grava de volta, **apagando o campo novo**. É preciso preservar campos desconhecidos ao decodificar para objetos e recodificar.
- **Dados sobrevivem ao código.** Um banco pode ter registros de cinco anos no formato antigo. Adicionar coluna com padrão nulo costuma ser barato; reescrever tudo é caro.
- **Arquivamento e snapshots:** bom momento para recodificar tudo no esquema atual e em formato analítico (ex.: Parquet).

**2. Via serviços: REST e RPC**
- **Serviço** expõe uma API específica da aplicação, diferente da consulta arbitrária de um banco. Isso dá encapsulamento.
- **Microsserviços** buscam deploy e evolução independentes, então versões antigas e novas de clientes e servidores convivem.
- **REST** é uma filosofia sobre HTTP: URLs como recursos, recursos do HTTP para cache, autenticação e negociação de conteúdo. É descrito com OpenAPI.
- **SOAP** é baseado em XML e WSDL, depende de ferramentas e caiu em desuso fora de grandes empresas.
- **O problema do RPC (transparência de localização):** tratar uma chamada de rede como chamada local é enganoso.
  - A rede é imprevisível: requisição ou resposta podem se perder.
  - Existe um terceiro desfecho, o **timeout**, em que você não sabe se a operação aconteceu.
  - Repetir pode executar duas vezes. É preciso **idempotência** e deduplicação.
  - A latência varia muito.
  - Parâmetros precisam ser codificados.
  - Tipos precisam ser traduzidos entre linguagens.
- **Frameworks modernos** (gRPC, Thrift, Finagle) assumem a diferença: futures/promises, streams, descoberta de serviço.
- **REST × RPC binário:** REST é fácil de depurar (navegador, `curl`), universal e tem ecossistema enorme, por isso predomina em APIs públicas. RPC binário rende mais e predomina entre serviços da mesma organização.
- **Compatibilidade em serviços:** dá para supor que os servidores atualizam antes dos clientes. Então basta compatibilidade para trás nas requisições e para frente nas respostas. Em REST, adicionar parâmetros opcionais e campos novos na resposta costuma ser seguro.
- **Versionamento de API:** não há consenso. Opções: versão na URL, no cabeçalho `Accept`, ou configurada por cliente no servidor. Quando não se controla os clientes, versões antigas podem precisar viver indefinidamente.

**3. Via mensagens assíncronas (brokers e atores)**
- Uma mensagem passa por um **broker** que a armazena temporariamente.
- **Vantagens:** absorve picos e indisponibilidade do destinatário; reentrega após falha; o remetente não precisa saber o endereço do destinatário; uma mensagem pode ir a vários consumidores; desacopla emissor e receptor.
- Geralmente é **via única e assíncrona**: quem envia não espera resposta. Respostas, quando existem, vão por outro canal.
- Ao republicar mensagens, preserve campos desconhecidos.
- **Atores distribuídos** (Akka, Orleans, Erlang): a transparência de localização funciona melhor porque o modelo já assume perda de mensagens. Deploy gradual continua exigindo compatibilidade de formato. Os formatos padrão desses frameworks nem sempre a oferecem (panorama de 2017).

## Trade-offs e decisões

| Formato | A favor | Contra |
|---|---|---|
| JSON (sem esquema) | Universal, legível, fácil de depurar | Números ambíguos, sem binário, compatibilidade depende de disciplina |
| Protocol Buffers / Thrift | Compacto, regras claras de evolução, geração de código | Tags manuais, dados ilegíveis sem decodificar |
| Avro | O mais compacto, esquemas dinâmicos, resolução escritor/leitor | Exige distribuir o esquema do escritor (registro de esquemas) |
| Nativo da linguagem | Conveniente | Inseguro, preso à linguagem, sem versionamento |

## Aplicação no mundo real

- **Migrações de esquema em etapas (expand/contract):**
  1. adicionar coluna nullable;
  2. código novo escreve nos dois formatos;
  3. backfill;
  4. código passa a ler o novo;
  5. remover o antigo.
  
  Cada etapa é compatível com a anterior, o que permite deploy gradual e rollback.
- **Contratos de API:** OpenAPI para REST e `.proto` para gRPC. Ferramentas de checagem de compatibilidade (ex.: `buf breaking` para Protocol Buffers) e registros de esquema (ex.: Confluent Schema Registry, para Avro ou Protobuf em Kafka) barram mudanças incompatíveis antes do deploy. Confirme nomes e comandos na documentação oficial.
- **IDs grandes em JSON:** envie IDs de 64 bits como string para clientes JavaScript.
- **Nunca desserialize formato nativo de fonte não confiável** (ex.: `pickle` de entrada do usuário).
- **Idempotência em APIs:** chaves de idempotência (ex.: um cabeçalho `Idempotency-Key`, padrão comum em APIs de pagamento) tornam os retries seguros.

## No Projeto prático (encurtador de links)

- **API pública REST** (`POST /links`, `GET /{codigo}`). Evolução só com campos opcionais novos.
- **Idempotência na criação:** um retry do cliente não pode gerar dois códigos para a mesma requisição.
- **Evento de clique na fila:** defina um esquema com versão desde o primeiro dia. Quando surgir um campo novo (ex.: país de origem), consumidores antigos precisam continuar funcionando.
- **Experimento sugerido:** deploy gradual de uma versão que adiciona um campo ao evento de clique, com um consumidor antigo ainda rodando, verificando que nada quebra e nada se perde.

## Armadilhas comuns

- Remover um campo obrigatório ou reutilizar a tag de um campo removido.
- Código antigo que regrava registros e apaga campos que não conhece.
- Tratar chamada de rede como chamada local, sem timeout, retry ou idempotência.
- Versionar a API só quando algo já quebrou.
- Supor que todos os dados do banco estão no formato mais recente.

## Perguntas de verificação

1. Defina compatibilidade para trás e para frente e diga qual é mais difícil de garantir.
2. Por que nunca se reutiliza um número de tag no Protocol Buffers?
3. Como o Avro evolui esquemas sem tags?
4. Cite três diferenças entre chamada local e chamada de rede e a consequência prática de cada uma.
5. Por que um broker de mensagens melhora a confiabilidade em relação a RPC direto?

## Referências

- [DDIA1, cap. 4, "Formats for Encoding Data", pp. 112–128]: formatos nativos, JSON/XML/CSV, MessagePack, Thrift, Protocol Buffers, Avro, méritos dos esquemas.
- [DDIA1, cap. 4, "Modes of Dataflow", pp. 128–139]: fluxo via banco, REST/RPC e seus problemas, versionamento de API, brokers de mensagens, atores distribuídos.
