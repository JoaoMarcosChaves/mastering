# Currículo construído antes da trilha; o curador passa a mantê-lo

O currículo inteiro (Mapa de competências, módulos e Unidades) é construído antes de a trilha começar, pelo Claude Code, e fica em `curriculo/`. A construção tem duas passadas: primeiro o esqueleto com objetivos, Competências e Unidades; depois o conteúdo completo, em lotes de módulos. O Agente curador deixa de escrever conteúdo um módulo à frente do aprendiz e passa a manter o currículo: reconfere o Conteúdo volátil antes de cada módulo e propõe referências novas.

Há três razões. O aprendiz ganha previsibilidade, porque revisa a trilha inteira e recebe estimativas por módulo. A validação é feita uma vez, com calma, em vez de às pressas antes de cada módulo. E os agentes ficam mais simples: escrever currículo é a tarefa mais cara e arriscada para um agente sem supervisão, e disputaria a cota do Codex com o tutor.

## Considered Options

- **Curador escrevendo um módulo à frente** (decisão anterior): mantinha o conteúdo sempre atual e se adaptava ao progresso. Foi rejeitada porque o aprendiz só enxergava o módulo seguinte e a validação virava gargalo antes de cada módulo.

## Consequences

- Cada Unidade separa o conteúdo durável (conceitos, trade-offs, perguntas da Lente) do Conteúdo volátil (versões, cotas gratuitas, preços, links), que registra a data da última conferência.
- As Unidades trazem Sementes de exercício, não exercícios prontos: decisões com opções ruim, boa e ótima, armadilhas, perguntas de verificação e critérios de rubrica. Como os agentes geram Pílulas a partir delas fica para ADRs futuros.
- A validação continua independente: na construção, cada lote é conferido por um agente diferente do que escreveu, e o aprendiz revisa por amostragem.
- Os ADRs seguintes tratam de como o conteúdo é interpretado, gamificado e transformado em exercícios. Mudar o currículo depois disso é manutenção.
- Deixa de valer o combinado anterior para o MVP, em que o Claude Code escreveria só o Módulo 1 e o curador assumiria a partir do Módulo 2.
