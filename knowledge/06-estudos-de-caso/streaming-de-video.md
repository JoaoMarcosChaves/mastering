---
titulo: "Estudo de caso: plataforma de vídeo (upload, transcodificação, streaming)"
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) + princípios primários (DDIA1); números de custo são exemplos didáticos de ~2020
atualizado: 2026-09-24
---

# Estudo de caso: plataforma de vídeo (estilo YouTube)

> Upload rápido e streaming suave de vídeos, com qualidade adaptável a cada dispositivo e rede, e custo de infraestrutura baixo. O desenho usa **serviços de nuvem** (object storage e CDN) em vez de construir tudo, um **pipeline de transcodificação** em DAG com filas e workers, **URLs pré-assinadas** para upload seguro e otimizações de **custo** guiadas pela popularidade (cauda longa).

## Por que importa

Qualquer sistema com arquivos grandes (vídeo, imagens, documentos, backups) enfrenta os mesmos problemas: upload resiliente, processamento pesado e assíncrono, armazenamento em object storage, distribuição por CDN e o custo de banda de saída, que costuma ser o maior item da conta.

## Escopo do exercício (SDI1)

- Upload e reprodução.
- Mobile, web e smart TV.
- 5 milhões de DAU, 30 minutos por dia.
- Usuários internacionais.
- Várias resoluções e formatos.
- Criptografia.
- Vídeos de até 1 GB.
- **Usar serviços de nuvem.**

### Estimativas

- 10% dos usuários enviam 1 vídeo por dia, com média de 300 MB, o que dá **150 TB/dia** de armazenamento.
- **Custo de CDN:** 5 M × 5 vídeos × 0,3 GB × US$ 0,02/GB ≈ **US$ 150 mil por dia**, usando o preço de CDN dos EUA citado no livro (~2020; confira a tabela atual do provedor). A lição: **banda de saída é o custo dominante.**

## Desenho de alto nível

- **Por que usar serviços prontos (object storage, CDN):** construir de forma escalável é caro e complexo. Nem gigantes constroem tudo (o SDI1 cita Netflix na AWS e o Facebook usando a Akamai).
- **Componentes:**
  - cliente;
  - **CDN** (streaming);
  - **servidores de API** (todo o resto: recomendações, URL de upload, metadados, cadastro).

### Fluxo de upload (dois fluxos em paralelo)

**a) O vídeo:**
1. upload para o **armazenamento de originais** (blob);
2. **servidores de transcodificação** processam;
3. em paralelo, o resultado vai para o **armazenamento de transcodificados** → CDN, e um **evento de conclusão** vai para a fila;
4. o **handler de conclusão** atualiza o banco e o cache de metadados;
5. a API avisa o cliente que o vídeo está pronto.

**b) Metadados** (nome, tamanho, formato): o cliente envia à API, que atualiza o banco (com sharding e réplicas) e o cache.

### Streaming

- **Streaming × download:** o cliente recebe pedaços continuamente e começa a tocar logo.
- **Protocolos de streaming adaptativo sobre HTTP:** MPEG-DASH e Apple HLS (entre outros).
- O vídeo sai da **borda da CDN mais próxima**.

## Aprofundamentos

- **Transcodificação, por que é necessária:**
  - o vídeo bruto é enorme (centenas de GB por hora em alta definição);
  - compatibilidade entre dispositivos;
  - **bitrate adaptativo** (qualidade conforme a banda, trocando dinamicamente).
  
  Um formato tem **contêiner** (.mp4, .mov) e **codec** (H.264, VP9, HEVC).
- **Modelo DAG:** tarefas configuráveis em estágios, sequenciais ou paralelas (inspiração citada: o pipeline de vídeo do Facebook). O vídeo é separado em vídeo, áudio e metadados, e passa por inspeção, codificações em várias resoluções e bitrates, miniatura e marca d'água.
- **Arquitetura de transcodificação:**
  - **pré-processador:** divide em **GOPs** (grupos de quadros independentes, de poucos segundos); gera o DAG a partir de configuração; guarda segmentos e metadados em armazenamento temporário para retry;
  - **agendador do DAG:** transforma o DAG em estágios e tarefas;
  - **gerenciador de recursos:** fila de tarefas (com prioridade), fila de workers (utilização), fila de execução, agendador que casa tarefa e worker;
  - **workers:** executam as tarefas;
  - **armazenamento temporário:** metadados em memória, mídia em blob, liberado ao terminar;
  - **vídeo codificado** como saída.
- **Otimizações:**
  - **velocidade:**
    - upload em **pedaços** alinhados por GOP (paralelo e **retomável**), divididos no cliente;
    - **centros de upload** perto do usuário (via CDN);
    - **filas entre as etapas** para desacoplar e paralelizar;
  - **segurança:**
    - **URL pré-assinada** (o cliente pede à API uma URL temporária com permissão só para aquele objeto e envia direto ao storage);
    - proteção de conteúdo: **DRM** (FairPlay, Widevine, PlayReady), criptografia AES com política de autorização, marca d'água visual;
  - **custo** (a distribuição de visualizações tem **cauda longa**):
    - CDN só para os populares, e o resto de servidores próprios de alta capacidade;
    - menos versões codificadas para conteúdo pouco visto (codificar sob demanda os curtos);
    - distribuir só nas regiões onde o vídeo é popular;
    - no extremo, **CDN própria com parceria com provedores de internet** (modelo do Netflix).
    - **Analise o padrão histórico antes de otimizar.**
- **Erros:**
  - **recuperáveis:** retry algumas vezes e depois erro;
  - **não recuperáveis:** vídeo malformado, parar e devolver erro;
  - "manual" por componente: upload (retry), divisão (fazer no servidor), transcodificação (retry), pré-processador (regerar o DAG), agendador (reagendar), fila do gerenciador (réplica), worker (retry em outro), API (sem estado, vai para outra instância), cache (réplicas), banco (promover réplica).
- **Extras:**
  - escalar a API (sem estado) e o banco;
  - **transmissão ao vivo** (latência mais rígida, protocolo diferente, menos paralelismo, tratamento de erro mais rápido);
  - **remoção de conteúdo** (direitos autorais, material ilegal, detecção no upload ou por denúncia).

## Conexões com os fundamentos (DDIA1)

- **Pipeline em DAG = motor de dataflow:** o mesmo modelo do Spark e do Flink (operadores em grafo, estágios, retry por tarefa). Tarefas devem ser **determinísticas e idempotentes**: retransformar um GOP gera o mesmo arquivo, e reprocessar é seguro. Ver [processamento em lote](../04-dados-derivados/processamento-em-lote.md).
- **Entrada imutável e saída derivada:** o original fica guardado. As versões transcodificadas são **dados derivados** e podem ser regeradas (novo codec, nova resolução) reprocessando os originais, como a evolução por reprocessamento do DDIA1 cap. 12.
- **Fila entre etapas:** desacoplamento, buffer e escala independente. Ver [processamento de fluxo](../04-dados-derivados/processamento-de-fluxo.md).
- **Metadados × blobs:** o banco guarda metadados pequenos (consultados o tempo todo) e o object storage guarda os blobs. É a separação clássica de padrões de acesso.
- **Eventos de conclusão e consistência:** a interface precisa lidar com o estado intermediário "em processamento" (o vídeo existe nos metadados, mas não está pronto). Ler as próprias escritas e estados explícitos evitam confusão.

## Aplicação no mundo real

- **Serviços gerenciados** (ex.: AWS Elemental MediaConvert, Transcoder API do Google Cloud, Mux, Cloudflare Stream) e **FFmpeg** para pipelines próprios. Confirme preços e recursos atuais.
- **Upload multipart e retomável** em object storage (S3 multipart, protocolo tus). URLs pré-assinadas com expiração curta e restrição de tamanho e tipo.
- **Custos:** estime a banda de saída **antes** de lançar. Use camadas de armazenamento (frio para o pouco acessado) e ciclo de vida dos objetos.
- **Moderação e direitos autorais:** impressões digitais de conteúdo e denúncias.

## No Projeto prático (encurtador de links)

- **Pré-visualização de páginas de destino:** um worker gera uma miniatura (captura de tela) da URL de destino. É um mini-pipeline assíncrono com fila, worker, armazenamento em object storage (ex.: MinIO local, compatível com S3) e evento de conclusão que atualiza os metadados do link.
- **Exercício:** upload de avatar do usuário via **URL pré-assinada** direto ao MinIO, sem passar pela API.

## Armadilhas comuns

- Receber arquivos grandes pela API em vez de URL pré-assinada.
- Upload sem retomada.
- Transcodificar de forma síncrona na requisição.
- Servir tudo pela CDN sem olhar a cauda longa (custo).
- Não guardar o original, o que impede reprocessar.

## Perguntas de verificação

1. Por que o custo de CDN domina, e quais três otimizações o reduzem?
2. O que é uma URL pré-assinada e que problema resolve?
3. Por que dividir o vídeo em GOPs ajuda no upload e na transcodificação?
4. Como o modelo DAG se relaciona com os motores de dataflow?
5. O que muda para transmissão ao vivo?

## Referências

- [SDI1, cap. 14, "Design YouTube"]: escopo, estimativas de armazenamento e custo de CDN, serviços de nuvem, fluxos de upload e streaming, protocolos, transcodificação, DAG, arquitetura de transcodificação, otimizações de velocidade, segurança e custo, tratamento de erros, ao vivo, remoção de conteúdo.
- [DDIA1, cap. 10, "Dataflow engines" e "Fault tolerance", pp. 421–423].
- [DDIA1, cap. 12, "Reprocessing data for application evolution", pp. 496–497].
