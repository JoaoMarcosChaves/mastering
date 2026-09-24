# Fontes

Catálogo das fontes citadas em `knowledge/`. Cada arquivo de conhecimento cita as fontes pela **chave** abaixo, seguida de capítulo, seção e página impressa. Exemplo: `[DDIA1, cap. 5, "Leaders and Followers", p. 152]`.

Os livros em si ficam **somente na máquina local** (`knowledge-resources/`, fora do git). Nada aqui reproduz trechos, figuras ou tabelas desses livros: o conteúdo é síntese própria, organizada por tema, com citação da origem.

| Chave | Obra | Edição | Papel na trilha |
|---|---|---|---|
| `DDIA1` | Martin Kleppmann, *Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems*. O'Reilly. | 1ª edição, março de 2017 | **Referência primária** |
| `SDI1` | Alex Xu, *System Design Interview – An Insider's Guide*, volume 1. Publicação independente. | 2ª edição, 2020 | **Referência secundária** (estrutura cenários; não substitui medições) |

## Observações sobre as edições

- **DDIA1:** a lista de validação da trilha cita "Kleppmann e Riccomini", que é a 2ª edição. O exemplar local é a 1ª edição (2017). Os princípios continuam válidos. Exemplos de ferramentas e o panorama de tecnologias refletem 2017: onde isso importa, o arquivo de conhecimento sinaliza **"(panorama de 2017)"**.
- **SDI1:** livro orientado a entrevista. Os números de exemplo (usuários, QPS, armazenamento) são hipóteses didáticas do autor, não medições. Por isso ele é Referência secundária (Q46/Q57 da sessão de refinamento).

## Como citar

- **DDIA1:** capítulo + título da seção + **página impressa** (a que aparece no rodapé), não o índice do arquivo PDF. No PDF local, a página impressa *p* corresponde aproximadamente à página *p + 22* do arquivo.
- **SDI1:** capítulo + título da seção. A edição não tem numeração impressa estável no PDF local.

## Erros e inconsistências encontrados nas fontes

A validação cruzada encontrou problemas que ficam registrados nos arquivos correspondentes:

- **SDI1, cap. 8:** o cálculo de armazenamento do encurtador multiplica por 10 anos duas vezes (365 TB, quando o correto é ~36,5 TB). Ver [encurtador de URLs](06-estudos-de-caso/encurtador-de-urls.md).
- **SDI1, caps. 7 e 8:** o encurtador usa o gerador Snowflake (64 bits) e ao mesmo tempo assume códigos de 7 caracteres, mas IDs Snowflake em base62 têm ~11. Mesmo arquivo.
- **SDI1 × DDIA1:** divergências conceituais sobre CAP, quóruns, hashing consistente e chaves quentes, e "consistência forte" com cache. Registradas na seção "Divergências" de cada arquivo e resumidas no [README](README.md).
