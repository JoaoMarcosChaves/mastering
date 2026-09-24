---
titulo: CDN (rede de distribuição de conteúdo)
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1, caps. 1 e 14); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# CDN (rede de distribuição de conteúdo)

> Uma CDN é uma rede de servidores geograficamente distribuídos que guarda cópias de conteúdo (sobretudo estático: imagens, vídeo, CSS, JS) **perto do usuário**. Ela reduz latência e carga na origem, mas custa por transferência e precisa de estratégia de expiração e invalidação. Em sistemas de mídia, é o **maior item de custo**.

## Por que importa

A latência entre continentes é de dezenas a centenas de milissegundos. Servir da borda muda a experiência percebida e protege a origem de picos. Para vídeo e downloads, a banda de saída da CDN domina a conta, então a arquitetura de distribuição é também uma decisão financeira.

## Conceitos-chave (SDI1)

- **Funcionamento:**
  1. o usuário pede um recurso a um domínio da CDN;
  2. em caso de *miss*, a borda busca na **origem** (servidor web ou object storage);
  3. a origem responde com um **TTL** (quanto tempo pode ficar em cache);
  4. a borda guarda e serve os pedidos seguintes até o TTL expirar.
- **Considerações:**
  - **custo:** cobrança por transferência. Não coloque na CDN o que é pouco acessado;
  - **TTL adequado:** longo demais serve conteúdo velho, curto demais sobrecarrega a origem;
  - **fallback:** se a CDN cair, o cliente deve conseguir buscar na origem;
  - **invalidação:** pela API do provedor, ou por **versionamento de URL** (ex.: `logo.png?v=2`, ou hash no nome do arquivo).
- **Conteúdo dinâmico:** também pode ser cacheado (por caminho, query, cookies e cabeçalhos). O SDI1 deixa isso fora do escopo.
- **Vídeo (SDI1 cap. 14):**
  - streaming a partir da borda mais próxima;
  - **estimativa de custo:** 5 milhões de DAU × 5 vídeos × 300 MB × US$ 0,02/GB ≈ **US$ 150 mil por dia** (preço de referência de ~2020);
  - **otimizações pela cauda longa:** CDN só para os populares, e o resto de servidores próprios; menos versões codificadas para os pouco vistos; distribuir só nas regiões onde há audiência; no extremo, **CDN própria com parceria com provedores de internet** (modelo do Netflix);
  - CDN como **centro de upload** perto do usuário;
  - tries de autocomplete por país servidas pela CDN (cap. 13).

## Conexões com os fundamentos (DDIA1)

- **CDN é cache, e cache é dado derivado:** vale a mesma disciplina de invalidação. O versionamento na URL é a forma mais robusta, porque a versão antiga nunca é "atualizada", só deixa de ser referenciada. É o princípio da imutabilidade do DDIA1 cap. 11. Ver [cache](cache.md).
- **Hashing consistente** nasceu para distribuir carga entre caches de uma CDN (DDIA1 cap. 6, Karger et al.). Ver [particionamento](../03-sistemas-distribuidos/particionamento.md).
- **Latência geográfica:** a replicação para perto do usuário é um dos três motivos para replicar (DDIA1 cap. 5).

## Aplicação no mundo real

- **Provedores:** CloudFront, Cloud CDN, Azure Front Door, Cloudflare, Fastly, Akamai. Muitos oferecem **computação na borda** (funções), WAF, proteção DDoS e rate limiting. Confirme recursos e preços atuais.
- **Cache-busting:** nomes de arquivos com hash do conteúdo (padrão dos bundlers de front-end) + `Cache-Control: public, max-age=31536000, immutable` para estáticos versionados. HTML com TTL curto ou revalidação.
- **Origem protegida:** acesso à origem só via CDN (ex.: object storage privado com permissão para a CDN).
- **Observabilidade:** hit ratio da CDN, tráfego de retorno à origem, erros por localização.
- **Custo:** a banda de saída da origem para a CDN e da CDN para o usuário têm preços diferentes. Modele os dois.

## No Projeto prático (encurtador de links)

- **Estáticos** (página inicial, JS, CSS) com nomes versionados em CDN, ou localmente com Nginx simulando a borda.
- **Redirecionamento na borda (opcional, avançado):** servir `GET /{código}` de uma função na borda com o mapa código → URL em um key-value distribuído. Discuta a invalidação quando um link é desativado por abuso (propagação global leva tempo, então é pontualidade × integridade) e o impacto no analytics (cliques precisam ser registrados mesmo servidos na borda).
- **Exercício de custo:** estimar a banda mensal do redirecionamento (resposta pequena) × a de páginas de pré-visualização com imagens, e comparar.

## Armadilhas comuns

- Colocar na CDN conteúdo pouco acessado e pagar sem benefício.
- Invalidar por API a cada deploy em vez de versionar URLs.
- Cachear respostas personalizadas como públicas (vazamento de dados entre usuários).
- Não ter fallback para a origem.
- Origem aberta ao público, contornando a CDN.

## Perguntas de verificação

1. Descreva o fluxo de *miss* e *hit* em uma CDN.
2. Por que versionar a URL é melhor do que invalidar por API?
3. Como a cauda longa de popularidade orienta a redução de custo com CDN?
4. Que risco existe ao cachear respostas com cookies ou dados do usuário?

## Referências

- [SDI1, cap. 1, seções "Content delivery network (CDN)" e "Considerations of using a CDN"].
- [SDI1, cap. 14, "Design YouTube"]: streaming pela CDN, custo, otimizações de custo, centros de upload.
- [SDI1, cap. 13, "Design a Search Autocomplete System"]: tries por país na CDN.
- [DDIA1, cap. 6, "Consistent Hashing", p. 204]: origem do hashing consistente em caches de CDN.
- [DDIA1, cap. 5, introdução, p. 151]: replicação para reduzir latência geográfica.
