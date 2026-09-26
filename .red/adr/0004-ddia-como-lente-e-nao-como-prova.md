# DDIA como lente, e não como prova

O DDIA deixa de ser Referência primária e passa a ser só a Lente da trilha: as perguntas-guia que o Agente curador usa para aprofundar cada unidade ("que garantia o cliente observa?", "isto é fonte da verdade ou dado derivado?"). Enquanto contava como prova, o DDIA formulava a pergunta, dava a resposta e ainda contava como uma das duas Referências primárias que a validam. Num tema como CAP, bastavam o DDIA e um artigo do próprio Kleppmann para validar a unidade: um único autor concordando consigo mesmo. A Lente pode formular perguntas, mas as respostas precisam de ≥2 Referências primárias de outros autores.

## Considered Options

- **O DDIA continua como prova, mas a segunda Referência primária não pode ser do Kleppmann**: dá menos trabalho de pesquisa e aproveita a fonte mais rigorosa sobre mecanismos. Foi rejeitada porque a visão do Kleppmann ainda definiria a pergunta e contaria como metade da prova.
- **Manter como estava**: DDIA mais um artigo do Kleppmann validariam uma unidade sozinhos.

## Consequences

- Os artigos do Kleppmann (ex.: "Please stop calling databases CP or AP") recebem o mesmo tratamento: servem de Lente, nunca de prova.
- Os 8 módulos em que o DDIA era a fonte principal (M1, M2, M3, M6, M7, M10, M11, M14) precisam de duas Referências primárias de fora. A lista atual (Fowler, Newman, Google SRE, AWS Well-Architected) quase não trata de dados distribuídos. Por isso, na construção do currículo, o curador propõe candidatas para esses temas (ex.: o texto de Brewer sobre CAP de 2012, o artigo do Dynamo, "A Critique of ANSI SQL Isolation Levels") e o aprendiz decide quais promover.
- As divergências entre SDI1 e DDIA (CAP, quóruns, hashing consistente, cache, ACID) deixam de ser decididas a favor do DDIA automaticamente. Sem confirmação de outro autor, entram como trade-off ou como opinião do autor.
- A síntese em `knowledge/` continua útil como material de estudo, mas nenhum arquivo tem hoje uma Referência primária que conte para a validação.
