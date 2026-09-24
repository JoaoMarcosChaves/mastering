---
titulo: Estimativas de capacidade (back-of-the-envelope)
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) com princípio de medição do DDIA1; aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Estimativas de capacidade (cálculo de guardanapo)

> Antes de desenhar, estime **ordens de grandeza**: requisições por segundo (QPS), pico, armazenamento, banda, memória de cache, número de servidores. O valor está no **processo** (premissas explícitas, unidades, arredondamento), não na precisão. A estimativa diz quais projetos são plausíveis. A **medição** confirma.

## Por que importa

Uma estimativa de dois minutos evita projetar um cluster para 10 requisições por segundo, ou uma máquina só para 100 mil. Ela também transforma requisitos vagos ("muitos usuários") em números que orientam escolhas: cabe em memória? precisa de sharding? a banda da CDN custa quanto?

## Conceitos-chave

### Potências de dois e unidades

| Aproximação | Nome | Unidade |
|---|---|---|
| 2^10 ≈ mil | kilo | KB |
| 2^20 ≈ milhão | mega | MB |
| 2^30 ≈ bilhão | giga | GB |
| 2^40 ≈ trilhão | tera | TB |
| 2^50 ≈ quatrilhão | peta | PB |

- Um caractere ASCII ocupa 1 byte. Um inteiro de 64 bits, 8 bytes. Um UUID, 16 bytes binário (36 como texto).
- Um dia tem ~86.400 s. Arredondar para **~10^5 s** facilita as contas. Um mês tem ~2,5 × 10^6 s.

### Latências: ordens de grandeza, não tabela sagrada

O SDI1 reproduz os "números de latência que todo programador deveria saber", que circulam a partir de uma apresentação de Jeff Dean (Google) com valores de ~2010. As conclusões qualitativas que o livro tira:

- **memória é rápida, disco é lento** (e busca aleatória em disco magnético é muito lenta);
- **compressão simples é rápida**: comprimir antes de mandar pela rede costuma compensar;
- **idas e voltas entre regiões custam dezenas a centenas de milissegundos.**

Escada de intuição (ordens de grandeza aproximadas; confira valores atualizados e **meça no seu hardware**):

| Operação | Ordem de grandeza |
|---|---|
| Referência ao cache da CPU | nanossegundos |
| Referência à memória principal | ~100 ns |
| Comprimir ~1 KB com algoritmo rápido | microssegundos |
| Ida e volta dentro do datacenter | ~0,5 ms |
| Ler 1 MB sequencial da memória / SSD | micro a poucos milissegundos |
| Busca em disco magnético | ~10 ms |
| Ida e volta entre continentes | ~100–150 ms |

**Posição da trilha:** o bootstrap (§1) rejeita "tabelas estáticas de latência" como verdade. Use a escada para **intuição e plausibilidade**. Para decidir, **meça**: a variação e a cauda (DDIA1, cap. 1 e 8) importam mais do que a média de uma tabela.

### Disponibilidade em "noves"

- **Disponibilidade** é a fração do tempo em que o serviço está operacional. **SLA** é o contrato com o cliente sobre ela. Provedores de nuvem costumam oferecer SLAs de 99,9% ou mais em muitos serviços (confira cada SLA).
- Indisponibilidade permitida (cálculo direto):

| Disponibilidade | Por ano | Por mês (~30 dias) | Por dia |
|---|---|---|---|
| 99% | ~3,65 dias | ~7,2 h | ~14,4 min |
| 99,9% | ~8,8 h | ~43 min | ~1,4 min |
| 99,99% | ~53 min | ~4,3 min | ~8,6 s |
| 99,999% | ~5,3 min | ~26 s | ~0,9 s |

- **Componentes em série multiplicam** as disponibilidades: dois serviços de 99,9% em série dão ~99,8%. **Componentes redundantes em paralelo** reduzem a indisponibilidade: 1 − (1 − A)². É por isso que redundância em todas as camadas importa.
- Para SLO, SLI e orçamento de erros, a Referência primária é o capítulo de SLOs do **Google SRE** (lista de validação da trilha).

### O roteiro de uma estimativa

1. **Premissas explícitas:** usuários ativos mensais (MAU), fração que usa por dia (DAU), ações por usuário por dia, proporção leitura/escrita, tamanho médio dos objetos, retenção.
2. **QPS médio** = DAU × ações por dia ÷ ~10^5 s.
3. **QPS de pico:** um multiplicador (ex.: 2× a 10× o médio, conforme o perfil de tráfego).
4. **Armazenamento** = escritas por dia × tamanho × dias de retenção (× fator de replicação).
5. **Banda** = QPS × tamanho da resposta.
6. **Cache:** a regra 80/20 é comum. Cachear ~20% dos objetos lidos por dia costuma cobrir a maior parte das leituras (heurística; valide com a distribuição real).
7. **Servidores** = QPS de pico ÷ QPS sustentado por servidor (medido).

### Exemplo do SDI1 (números didáticos, não reais)

Rede social com 300 milhões de MAU, 50% diários, 2 posts por usuário por dia, 10% com mídia de 1 MB, retenção de 5 anos:

- DAU = 150 milhões;
- QPS de posts ≈ 150 M × 2 ÷ 86.400 ≈ **3.500**, com pico ≈ **7.000**;
- mídia por dia ≈ 150 M × 2 × 10% × 1 MB = **30 TB/dia**, e em 5 anos ≈ **55 PB**.

### Dicas de processo

- **Arredonde sem medo:** 99.987 / 9,1 ≈ 100.000 / 10.
- **Escreva as premissas** para consultar depois.
- **Rotule as unidades:** "5" é KB ou MB?
- Estimativas comuns: QPS, pico, armazenamento, cache e servidores.

## Trade-offs e decisões

- **Estimativa × medição:** estimar orienta a arquitetura inicial, e medir (teste de carga, produção) confirma e corrige. Uma sem a outra é chute ou é tarde demais.
- **Média × pico × cauda:** dimensione para o pico com folga, e use SLOs com percentis (não médias).

## Aplicação no mundo real

- **Planejamento de capacidade:** estime, provisione com margem, meça em produção e reestime periodicamente.
- **Custo:** converter armazenamento e banda estimados em custo mensal (preços da calculadora do provedor) costuma mudar decisões, como compressão, retenção ou camada de armazenamento frio.
- **Teste de carga para calibrar:** meça a vazão real de uma instância (ex.: com k6) e use esse número no passo 7, não um palpite.

## No Projeto prático (encurtador de links)

Premissas de exercício (ajuste as suas):

- 100 milhões de links novos por mês;
- leitura:escrita = 100:1;
- 500 bytes por link;
- retenção de 5 anos.

Estimativa:

- escrita ≈ 100 M ÷ (30 × 10^5 s) ≈ **~40 QPS**, e leitura ≈ **~4.000 QPS**;
- armazenamento ≈ 100 M × 12 × 5 × 500 B ≈ **~3 TB** em 5 anos.

Conclusão de plausibilidade: um PostgreSQL bem indexado + cache aguenta com folga no começo, e sharding não é necessário cedo.

**Exercício da trilha:** comparar essa estimativa com a vazão medida de uma instância local e anotar a diferença, como sugere a conquista **Investigador** (hipótese refutada por medição).

## Armadilhas comuns

- Esquecer o pico e dimensionar pela média.
- Misturar unidades (bits × bytes, MB × MiB, por segundo × por dia).
- Usar números de latência de tabela como garantia.
- Esquecer o fator de replicação no armazenamento.
- Não escrever as premissas.

## Perguntas de verificação

1. Com 10 milhões de DAU e 20 leituras por usuário por dia, qual o QPS médio de leitura?
2. Quanto tempo de indisponibilidade por mês um SLO de 99,9% permite?
3. Dois componentes de 99,9% em série: qual a disponibilidade composta?
4. Por que as tabelas de latência servem para intuição, mas não para decisão?

## Referências

- [SDI1, cap. 2, "Back-of-the-envelope Estimation"]: potências de dois, números de latência, disponibilidade em noves, exemplo de QPS e armazenamento, dicas.
- [DDIA1, cap. 1, "Describing Performance", pp. 13–17]: percentis, cauda, medir no cliente.
- [DDIA1, cap. 8, "Unreliable Networks", pp. 277–287]: variabilidade de atrasos e por que medir.
