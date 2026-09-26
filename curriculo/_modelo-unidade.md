# Modelo de Unidade

Use este formato para cada Unidade na passada 2. Salve como `curriculo/MNN-slug/UN-slug.md` (ex.: `M04-escalar-leituras/U3-estrategias-de-atualizacao.md`) e ligue o arquivo na tabela de Unidades do `modulo.md`. Copie o bloco abaixo e preencha todas as seções. Quando uma seção não se aplicar, escreva "Não se aplica" com o motivo. A Unidade está pronta para `rascunho` quando todas as seções estiverem preenchidas e cada afirmação relevante tiver fonte.

```markdown
---
id: M04.U3
modulo: M04
titulo: "Estratégias de atualização do cache"
competencias: [M04.C2]
nivel-alvo: 2                 # nível da escala 0–4 que a Unidade leva o aprendiz a atingir
lente: [L04, L05]             # perguntas de curriculo/lente.md que a Unidade responde
status: rascunho              # esqueleto | rascunho | validado
referencias-primarias: []     # ≥2 por conceito, autores diferentes, nenhuma do Kleppmann
conferido-em: 2026-09-26      # última conferência do Conteúdo volátil
---

# M04.U3 · Estratégias de atualização do cache

## Objetivo
Uma frase observável: o que o aprendiz faz ao final. "Escolher entre cache-aside e write-through para o redirecionamento e explicar a inconsistência que cada um aceita."

## Por que importa
Duas ou três frases ligando o tema ao encurtador ou a um sistema real.

## Conceitos
Síntese própria, curta, com fonte ao fim de cada parágrafo. Aponte para `knowledge/` quando o tema já estiver lá.

## Trade-offs
| Opção | Ganha | Paga | Escolha quando |
|---|---|---|---|

## Divergências entre fontes
O que as fontes dizem de diferente e como a trilha apresenta isso como trade-off. "Nenhuma encontrada" também é resposta.

## Respostas da Lente
Uma resposta, com fonte, para cada pergunta listada em `lente:`.

## Sementes de exercício

### Decisões
Contexto com números → opção ruim / boa / ótima → consequência de cada uma.

### Armadilhas
Erro comum → por que acontece → como perceber.

### Perguntas de verificação
Pergunta → resposta esperada em uma ou duas frases.

### Critérios de rubrica
- Nível 1: ...
- Nível 2: ...
- Nível 3: ...

## Leituras
| Leitura (link) | Trecho | Tempo | Tipo | Por que ler |
|---|---|---|---|---|
Tipo: primária, secundária ou documentação oficial. Leituras de 5–10 minutos, com o trecho delimitado.

## Conteúdo volátil
Fatos de ferramenta (versão, cota gratuita, preço, comando), cada um com link para a documentação oficial e data de conferência.

## Fontes
Lista com a chave de cada fonte e o trecho usado. Chaves dos livros: `knowledge/fontes.md`.
```

## Exemplo de semente de decisão

> **Contexto:** o redirecionamento recebe 2.000 leituras por segundo e 5 escritas por segundo; um link apagado por abuso pode redirecionar por no máximo 1 minuto.
> - **Ruim:** write-behind, que confirma a escrita antes de gravar no banco. Consequência: uma queda do Redis perde links recém-criados.
> - **Boa:** cache-aside com TTL de 1 minuto. Consequência: cumpre o limite, mas cada link expira a cada minuto e o banco recebe a releitura.
> - **Ótima:** cache-aside com invalidação na remoção e TTL longo como rede de segurança. Consequência: taxa de acerto alta e remoção quase imediata, com o custo de tratar a corrida entre a invalidação e uma leitura concorrente.
