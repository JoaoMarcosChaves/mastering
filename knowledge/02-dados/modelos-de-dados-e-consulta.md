---
titulo: Modelos de dados e linguagens de consulta
fontes: [DDIA1]
status-validacao: base primária única (DDIA1); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Modelos de dados e linguagens de consulta

> O modelo de dados muda não só o código, mas também a forma de pensar o problema. Relacional, documento e grafo são bons em domínios diferentes. O que decide a escolha é o **tipo de relacionamento** entre os dados: um-para-muitos (árvore), muitos-para-um ou muitos-para-muitos.

## Por que importa

Escolher o modelo errado não quebra o sistema no primeiro dia. O custo aparece aos poucos: joins emulados na aplicação, dados duplicados divergindo, consultas que exigem reescrever o código. Escolher bem começa por olhar como os dados se relacionam e como serão lidos.

## Conceitos-chave

### Camadas de modelo

Toda aplicação empilha modelos: objetos da aplicação → modelo genérico (tabelas, documentos JSON, grafo) → bytes em memória e disco → sinais físicos. Cada camada esconde a complexidade da de baixo. Todo modelo embute premissas sobre o uso: algumas operações ficam naturais e rápidas, outras desajeitadas.

### Relacional

- Dados em relações (tabelas): coleções não ordenadas de tuplas (linhas). Proposto por Codd em 1970, dominante desde meados dos anos 1980.
- O grande ganho histórico foi **esconder o caminho de acesso**. O otimizador de consultas escolhe índices e ordem de execução. Criar um índice novo acelera consultas existentes sem reescrevê-las.
- Excelente para **muitos-para-um e muitos-para-muitos** via chaves estrangeiras e joins.
- **Normalização** evita duplicar informação legível por humanos: guarda-se um ID, e o nome fica em um lugar só. Isso facilita atualizações, evita ambiguidade e habilita localização e busca melhor. IDs não precisam mudar porque não têm significado para humanos.

### Documento

- Registros autocontidos (JSON, XML, BSON), aninhando relações um-para-muitos dentro do pai. É o formato natural para estruturas em árvore, como um currículo com empregos, formação e contatos.
- **Localidade:** tudo em um lugar, uma leitura só. A vantagem só existe se você costuma precisar de grande parte do documento de uma vez. Atualizar geralmente reescreve o documento inteiro, então a recomendação é manter documentos pequenos.
- **Joins fracos ou ausentes.** Relações muitos-para-um e muitos-para-muitos viram consultas extras na aplicação ou dados duplicados que você precisa manter coerentes.
- **Os dados tendem a ficar mais interligados com o tempo.** Um modelo que começa sem joins costuma ganhar referências a entidades conforme o produto cresce (ex.: empresa vira entidade com página própria; recomendações referenciam o autor).
- **"Sem esquema" é impreciso.** O código que lê sempre assume uma estrutura. O correto é:
  - **schema-on-read:** esquema implícito, interpretado na leitura, análogo à tipagem dinâmica;
  - **schema-on-write:** esquema explícito, validado na escrita, análogo à tipagem estática.
- **Quando schema-on-read ajuda:** dados heterogêneos (muitos tipos de objeto) ou estrutura ditada por sistemas externos que mudam sem aviso.
- **Mudança de formato nos dois mundos:**
  - com documentos, grava-se o formato novo e o código trata os documentos antigos na leitura;
  - no relacional, faz-se uma migração (`ALTER TABLE` + `UPDATE`). O `ALTER` costuma ser rápido na maioria dos bancos; reescrever uma tabela grande é lento em qualquer banco. O MySQL da época copiava a tabela inteira no `ALTER` (panorama de 2017).

### A lição histórica

- O **modelo hierárquico** (IMS, anos 1960–70) era uma árvore de registros, muito parecida com documentos JSON. Era bom para um-para-muitos, ruim para muitos-para-muitos e sem joins.
- O **modelo em rede** (CODASYL) permitia vários pais, mas o acesso era por **caminhos de acesso** navegados manualmente. O código ficava complexo e mudar o modelo exigia reescrever consultas.
- O **relacional venceu** por expor os dados de forma plana e delegar o caminho de acesso ao otimizador.
- Bancos de documento voltaram ao aninhamento para um-para-muitos. Para muitos-para-um e muitos-para-muitos, usam referências por ID resolvidas na leitura, exatamente como chaves estrangeiras.

### Convergência

Bancos relacionais passaram a suportar JSON: PostgreSQL desde a 9.3, MySQL desde a 5.7 (panorama de 2017). Bancos de documento passaram a suportar joins ou resolução de referências. O caminho saudável é o **híbrido**: usar o modelo que se encaixa em cada parte dos dados.

### Grafos

- **Vértices e arestas** servem quando "tudo pode se relacionar com tudo": redes sociais, a web, malhas viárias, ou um grafo heterogêneo com muitos tipos de entidade.
- **Grafo de propriedades** (ex.: Neo4j): vértices e arestas com ID, rótulo e propriedades. É possível percorrer nos dois sentidos, e qualquer vértice pode se ligar a qualquer outro. Isso dá boa evolutividade.
- **Triple-store** (sujeito, predicado, objeto), usado com RDF: modelo praticamente equivalente, com outra terminologia.
- **Consultas de travessia de tamanho variável** (ex.: "nasceu em algum lugar dentro dos EUA", seguindo a cadeia cidade → estado → país) são naturais em linguagens de grafo e desajeitadas em SQL, onde exigem CTEs recursivas.
- **Grafos não são o CODASYL de volta:** não há restrição de quais tipos se ligam, é possível acessar vértices diretamente por ID ou índice, não há ordem obrigatória, e há linguagens declarativas.

### Linguagens de consulta

- **Declarativa × imperativa.** Na declarativa (SQL, Cypher, SPARQL, CSS), você diz *o que* quer, e o motor decide *como*. Ganhos: concisão, liberdade para o banco otimizar e reorganizar dados, e paralelização.
- **Imperativa:** você dita a ordem das operações. O banco não sabe no que o código depende (ex.: ordem dos registros), então não pode otimizar livremente.
- **MapReduce:** meio-termo, com funções puras `map` e `reduce` chamadas pelo framework. É de baixo nível e exige coordenar duas funções. Por isso sistemas NoSQL acabam "reinventando o SQL" com linguagens declarativas (ex.: o aggregation pipeline do MongoDB).
- **Datalog:** base teórica das linguagens de grafo. Constrói consultas complexas com regras reutilizáveis e recursivas.

## Trade-offs e decisões

| Forma dos dados | Modelo mais natural | Por quê |
|---|---|---|
| Árvore de um-para-muitos, carregada inteira | Documento | Localidade, sem joins, código mais simples |
| Entidades com muitos-para-um e muitos-para-muitos moderados | Relacional | Joins eficientes, normalização, otimizador |
| Dados altamente interligados, travessias de profundidade variável | Grafo | Travessia natural, flexibilidade de relacionamentos |
| Dados heterogêneos ou ditados por terceiros | Documento (ou colunas JSON) | Schema-on-read evita esquemas que atrapalham |

- **Desnormalizar** reduz joins, mas transfere para a aplicação o trabalho de manter as cópias coerentes.
- **Emular joins na aplicação** funciona, mas costuma ser mais lento e mais complexo do que o join no banco.

## Aplicação no mundo real

- **Comece pela forma das consultas:** liste as telas e APIs e veja se cada uma lê uma árvore inteira ou cruza entidades.
- **Relacional com colunas JSON** (ex.: PostgreSQL `jsonb`, SQLite com funções JSON) atende a maioria das aplicações: núcleo normalizado e partes variáveis em JSON. É o padrão adotado pelo App de estudos (ver ADR 0002).
- **Banco de documento** brilha em agregados autocontidos com acesso por chave (catálogos, perfis, eventos) e sem necessidade de relacionar entidades.
- **Banco de grafo** vale a pena quando as consultas centrais são travessias (recomendação, fraude, permissões hierárquicas). Antes disso, CTEs recursivas no relacional costumam bastar.
- **Persistência poliglota:** usar mais de um modelo no mesmo sistema é comum. O custo é manter os dados sincronizados (ver [integração de dados e correção](../04-dados-derivados/integracao-de-dados-e-correcao.md)).

## No Projeto prático (encurtador de links)

- **Link** (código → URL de destino, dono, data de criação) é uma entidade simples com acesso por chave. Serve tanto em relacional quanto em chave-valor.
- **Cliques** são eventos imutáveis, com relacionamento muitos-para-um com o link. Consultas de analytics agregam por link e por período.
- **Decisão sugerida para a primeira versão:** PostgreSQL relacional, com cliques em uma tabela de eventos. Rever quando as medições mostrarem gargalo.

## Armadilhas comuns

- Escolher documento porque "não tem esquema" e descobrir que o esquema está espalhado pelo código.
- Aninhar dados que depois precisam ser referenciados por outras entidades.
- Documentos que crescem sem limite, com reescrita cara a cada atualização.
- Ir para um banco de grafo sem consultas de travessia de verdade.

## Perguntas de verificação

1. Por que a presença de relações muitos-para-muitos pesa contra o modelo de documento?
2. Explique schema-on-read × schema-on-write com a analogia de tipagem.
3. O que o modelo relacional resolveu em relação ao CODASYL?
4. Por que uma linguagem declarativa dá mais liberdade ao banco para otimizar?
5. Em que situação uma CTE recursiva em SQL substitui um banco de grafo?

## Referências

- [DDIA1, cap. 2, "Relational Model Versus Document Model", pp. 28–42]: NoSQL, impedância objeto-relacional, relações muitos-para-um e muitos-para-muitos, IMS e CODASYL, flexibilidade de esquema, localidade, convergência.
- [DDIA1, cap. 2, "Query Languages for Data", pp. 42–48]: declarativo × imperativo, MapReduce, aggregation pipeline.
- [DDIA1, cap. 2, "Graph-Like Data Models", pp. 49–63]: grafo de propriedades, Cypher, SQL recursivo, triple-stores, SPARQL, Datalog.
