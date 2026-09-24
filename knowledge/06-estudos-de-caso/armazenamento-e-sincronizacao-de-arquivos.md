---
titulo: "Estudo de caso: armazenamento e sincronização de arquivos (estilo Google Drive/Dropbox)"
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) confrontada com primária (DDIA1); contém divergência registrada sobre consistência
atualizado: 2026-09-24
---

# Estudo de caso: armazenamento e sincronização de arquivos

> Enviar, baixar e **sincronizar** arquivos entre vários dispositivos, com histórico de versões, compartilhamento e notificações. Os requisitos que dominam o desenho são **nunca perder dados**, **sincronizar rápido** e **economizar banda**. O desenho separa **metadados** (banco relacional) de **conteúdo** (blocos em object storage), usa **sincronização delta** por blocos, **notifica** mudanças e resolve **conflitos** de edição concorrente.

## Por que importa

Backups, anexos, documentos compartilhados, sincronização offline de apps. O caso junta durabilidade, deduplicação por hash de conteúdo, versionamento imutável, estados intermediários (upload "pendente"), notificação de mudanças e resolução de conflitos, que é o problema da replicação multilíder quando cada dispositivo é um "líder".

## Escopo do exercício (SDI1)

- **Funcionalidades:** adicionar (arrastar e soltar), baixar, sincronizar entre dispositivos, ver revisões, compartilhar, notificar edição, exclusão e compartilhamento.
- **Fora do escopo:** edição colaborativa em tempo real.
- **Outros requisitos:** qualquer tipo de arquivo, criptografado em repouso, até 10 GB por arquivo, 10 milhões de DAU.
- **Não funcionais:** **confiabilidade** (perder dados é inaceitável), **sincronização rápida**, **economia de banda** (planos móveis), escalabilidade e alta disponibilidade.

### Estimativas

- 50 milhões de usuários × 10 GB grátis = **500 PB alocados**.
- 2 uploads por dia de ~500 KB, leitura:escrita 1:1: upload ≈ **~240 QPS** (pico ≈ 480).

## Evolução do desenho

1. **Um servidor:** servidor web + MySQL + diretório `drive/` com um *namespace* por usuário.
   - **APIs** (todas autenticadas e com HTTPS):
     - upload **simples** ou **retomável** (pede uma URL retomável, envia, retoma em caso de falha);
     - download por caminho;
     - listar revisões.
2. **Disco cheio:** sharding por usuário resolve o espaço, mas não o risco de perda. Solução: migrar para **object storage** (S3) com **replicação na mesma região e entre regiões**.
3. **Desacoplar:**
   - balanceador;
   - servidores web escaláveis;
   - banco de metadados separado, com replicação e sharding;
   - arquivos no object storage replicado em duas regiões.
4. **Conflitos de sincronização:** quando dois usuários editam o mesmo arquivo, **a primeira versão processada vence**, e a segunda recebe um conflito. O sistema mostra as duas cópias (local e do servidor), e o usuário mescla ou escolhe uma.

## Desenho de alto nível

- **Servidores de blocos:** dividem o arquivo em **blocos** (referência do Dropbox: até 4 MB), cada um com hash, e enviam ao armazenamento. O arquivo é reconstruído juntando os blocos em ordem.
- **Armazenamento em nuvem** (blocos) + **armazenamento frio** para dados inativos.
- **Servidores de API:** autenticação, perfil, metadados.
- **Banco e cache de metadados:** usuários, arquivos, blocos, versões.
- **Serviço de notificação:** pub/sub que avisa os clientes sobre mudanças feitas em outro lugar.
- **Fila de backup offline:** guarda mudanças para clientes offline sincronizarem depois.

## Aprofundamentos

- **Servidores de blocos:**
  - **sincronização delta:** envia só os blocos alterados (algoritmos no estilo rsync);
  - **compressão** por tipo de arquivo;
  - **criptografia** de cada bloco antes do envio.
- **Consistência forte (segundo o SDI1):** um arquivo não pode aparecer diferente em clientes diferentes ao mesmo tempo. Para isso, o livro propõe cache consistente com o primário e **invalidação do cache na escrita**, e **banco relacional pelo ACID nativo**.
- **Esquema:**
  - usuário;
  - dispositivo (`push_id`);
  - namespace (raiz do usuário);
  - arquivo (versão mais recente);
  - **versão do arquivo** (**linhas somente leitura**, para preservar o histórico);
  - bloco.
- **Fluxo de upload** (duas requisições em paralelo):
  - **metadados:** grava com status **"pendente"** e notifica os outros clientes;
  - **conteúdo:** servidores de blocos dividem, comprimem, criptografam e enviam. O armazenamento chama um **callback de conclusão**, o status passa a **"enviado"** e os clientes são notificados.
- **Fluxo de download:** o cliente é notificado (online) ou busca ao reconectar (offline) → pede metadados → baixa os blocos novos → reconstrói.
- **Notificação:** **long polling** (como o Dropbox, segundo o SDI1) em vez de WebSocket, porque a comunicação é só servidor → cliente e as notificações são infrequentes. Ao detectar mudança, o servidor fecha a conexão e o cliente busca os metadados.
- **Economizar espaço:**
  - **deduplicar blocos** por hash (no nível da conta);
  - **política de versões:** limite de versões e manter só as "valiosas", com mais peso às recentes;
  - **armazenamento frio** (ex.: classes de arquivamento do S3) para o que não é acessado há meses.
- **Falhas:**
  - balanceador secundário com heartbeat;
  - servidor de blocos (outros assumem os pendentes);
  - storage (réplicas em outras regiões);
  - API sem estado;
  - cache replicado;
  - banco (promover réplica);
  - **servidor de notificação** (milhares ou milhões de conexões perdidas, e reconexão lenta);
  - fila offline replicada.
- **Alternativas discutidas:**
  - **upload direto do cliente para o storage:** mais rápido, mas exige lógica de blocos, compressão e criptografia em cada plataforma, e cliente manipulável é ruim para criptografia;
  - separar um **serviço de presença**.

## Divergências entre as fontes

- **"Consistência forte é fácil em banco relacional porque ele é ACID" (SDI1):** o DDIA1 mostra que isso é bem mais sutil:
  - o C do ACID é **da aplicação**;
  - a maioria dos bancos relacionais roda com **isolamento fraco** por padrão (read committed);
  - **réplicas assíncronas não são linearizáveis**;
  - "invalidar o cache na escrita" é uma **gravação dupla** sujeita a condições de corrida (duas escritas concorrentes podem deixar o cache com o valor antigo).
  
  Leitura correta: use o banco com líder único como **sistema de registro** para metadados, leia do líder ou com garantias de ler as próprias escritas quando necessário, e trate o cache como **dado derivado** (invalidação via CDC ou versão, ou TTL curto). Ver [consistência e consenso](../03-sistemas-distribuidos/consistencia-e-consenso.md) e [integração de dados e correção](../04-dados-derivados/integracao-de-dados-e-correcao.md).
- **Conflitos:** "a primeira processada vence, a outra vira cópia de conflito" é uma estratégia de **não perder dados** (equivalente a guardar irmãos e deixar o usuário mesclar), coerente com o DDIA1. Para edição colaborativa em tempo real, o DDIA1 aponta CRDTs e transformação operacional.

## Conexões com os fundamentos (DDIA1)

- **Clientes offline = replicação multilíder:** cada dispositivo aceita escritas localmente e sincroniza depois. Conflitos são inevitáveis e precisam de estratégia. Ver [replicação](../03-sistemas-distribuidos/replicacao.md).
- **Versões imutáveis:** o histórico somente leitura dá auditabilidade e recuperação de erros. É o mesmo princípio de eventos imutáveis do DDIA1 cap. 11 e 12.
- **Endereçamento por conteúdo:** blocos identificados pelo hash permitem deduplicação e verificação de integridade de ponta a ponta (checksums).
- **Sincronização por log:** a fila de backup offline funciona como offset de consumidor ("tudo desde a última versão que o dispositivo viu").
- **Estados intermediários:** "pendente" → "enviado" evita que outros clientes baixem um arquivo incompleto. O callback de conclusão deve ser **idempotente**.

## Aplicação no mundo real

- **Object storage** (S3, GCS, Azure Blob, MinIO local) com versionamento, replicação, regras de ciclo de vida (mover para classes frias) e **criptografia** gerenciada ou do cliente.
- **Upload retomável:** multipart do S3 ou protocolo tus. **URLs pré-assinadas** para o cliente enviar direto ao storage sem expor credenciais.
- **Sincronização delta:** algoritmos no estilo rsync, com *content-defined chunking* (blocos com fronteiras definidas pelo conteúdo, que resistem a inserções no meio do arquivo).
- **Integridade:** checksums por bloco verificados no download.
- **Segurança e privacidade:** controle de acesso por namespace, links de compartilhamento com expiração, auditoria de acessos, obrigações de exclusão (LGPD/GDPR) que também valem para versões e backups.

## No Projeto prático (encurtador de links)

- **Exportação de relatórios:** o dono pede um CSV com todos os cliques de um link. Um job gera o arquivo, grava no object storage (MinIO), marca "pendente → pronto" nos metadados e notifica (SSE ou e-mail) com uma **URL pré-assinada** de download que expira.
- **Exercício:** tornar o job idempotente (repetir o pedido não gera dois arquivos) e testar o estado "pendente" quando o worker cai no meio.

## Armadilhas comuns

- Guardar arquivos no disco do servidor da aplicação.
- Enviar o arquivo inteiro a cada pequena alteração.
- Não ter estado intermediário ("pendente"), o que deixa clientes baixarem arquivos incompletos.
- Invalidação de cache por gravação dupla, achando que isso dá consistência forte.
- Guardar versões sem limite nem política de armazenamento frio.
- Sobrescrever silenciosamente em conflito (perda de dados).

## Perguntas de verificação

1. Por que separar metadados (banco) de conteúdo (object storage)?
2. Como a sincronização delta economiza banda? Por que os blocos têm hash?
3. Qual a estratégia de conflito proposta e por que ela não perde dados?
4. Por que long polling foi preferido a WebSocket neste caso?
5. Onde o SDI1 simplifica demais a "consistência forte", segundo o DDIA1?

## Referências

- [SDI1, cap. 15, "Design Google Drive"]: escopo, estimativas, evolução de um servidor a S3, APIs de upload retomável, conflitos, desenho de alto nível, servidores de blocos, sincronização delta, consistência, esquema, fluxos de upload e download, notificação por long polling, economia de espaço, falhas, alternativas.
- [DDIA1, cap. 5, "Clients with offline operation" e "Handling Write Conflicts", pp. 170–177].
- [DDIA1, cap. 7, "The Meaning of ACID", pp. 223–227].
- [DDIA1, cap. 11, "Keeping Systems in Sync", pp. 452–454].
- [SDI1, cap. 16, "The Learning Continues"]: lista de arquiteturas reais e blogs de engenharia para aprofundamento (links do livro podem estar desatualizados).
