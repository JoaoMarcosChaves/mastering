# Lente e princípios

Este arquivo define a maneira de pensar que todo agente da trilha aplica ao compor, validar ou ensinar uma Unidade. A seção 1 traz os princípios da trilha, que valem sempre. A seção 2 traz as perguntas da **Lente**, que cada Unidade cita pelo identificador (`L01`–`L12`).

A Lente vem do DDIA e dos artigos do Kleppmann. Ela formula perguntas; as respostas precisam de Referências primárias de outros autores ([ADR 0004](../.red/adr/0004-ddia-como-lente-e-nao-como-prova.md)).

## 1. Princípios da trilha

1. **Parta de requisitos com números.** Toda decisão começa por carga, volume, percentil de latência e meta de disponibilidade, nunca por uma tecnologia.
2. **Nomeie o preço de cada ganho.** Toda escolha vem com o que ela custa e com a condição que faria a decisão mudar.
3. **Meça antes de acreditar.** Tabelas de latência e números de livros são ordem de grandeza; a resposta vem da medição no lab. Hipótese refutada por medida é resultado, não fracasso.
4. **Comece simples.** A complexidade entra quando uma medição mostra que ela é necessária. Descartar complexidade com evidência é uma decisão de arquiteto.
5. **Descreva garantias com precisão.** "Consistente" e "disponível" sempre vêm com "para quem, quando e sob que falha". CAP só aparece no contexto de uma partição real, sem rótulos CP/AP.
6. **Planeje a falha.** Toda dependência falha; decida antes o que o sistema faz quando isso acontece.
7. **Separe fonte da verdade de dado derivado.** Cache, índice de busca e agregação são cópias; saiba de onde vêm e como voltam a ficar corretas.
8. **Uma solução de entrevista é um exemplo, não uma receita.** O contexto real (equipe, custo, operação) pode mudar a resposta.
9. **Dados têm dono.** Coletar é uma decisão com consequências para quem é observado.

## 2. Perguntas da Lente

| ID | Pergunta | Onde pesa mais | Origem no DDIA |
|---|---|---|---|
| L01 | Qual é o parâmetro de carga, e que percentil importa para quem usa? | M01, M05, M09 | cap. 1 |
| L02 | Como os dados são lidos e escritos? Isto é transação (OLTP) ou análise (OLAP)? | M02, M13 | caps. 2–3 |
| L03 | Que regra precisa valer sempre, e o que pode quebrá-la quando muitos agem ao mesmo tempo? | M03, M08 | cap. 7 |
| L04 | Isto é fonte da verdade ou dado derivado? Como a cópia volta a ficar correta? | M04, M13, M14 | caps. 11–12 |
| L05 | Que garantia a pessoa enxerga logo depois de escrever? E durante uma partição? | M06, M04 | caps. 5 e 9 |
| L06 | Que modelo de falha estou assumindo? O que acontece se isto parar no meio? | M05, M10 | caps. 8–9 |
| L07 | E se a mensagem chegar duas vezes, fora de ordem ou nunca? | M07, M14 | caps. 11–12 |
| L08 | Em que relógio estou confiando? Conto pelo tempo do evento ou do processamento? | M07, M10 | caps. 8 e 11 |
| L09 | Como a carga se divide entre as máquinas, e o que acontece com a chave mais quente? | M11, M05 | cap. 6 |
| L10 | Como isto muda sem quebrar quem depende da versão anterior? | M02, M14 | cap. 4 |
| L11 | Que número prova que o sistema atende bem quem usa? | M09, M01 | cap. 1 |
| L12 | Que dado estou coletando, e com que direito? | M08 | cap. 12 |

## 3. Como aplicar a Lente

- **Ao compor uma Unidade:** responda, com fontes, a cada pergunta da Lente listada no cabeçalho da Unidade. Uma resposta que só o DDIA sustenta entra como trade-off ou como opinião do autor.
- **Ao gerar uma Pílula:** use a pergunta da Lente para escrever a consequência de cada opção. A opção ruim é a que falha na pergunta.
- **Ao validar:** confira se cada pergunta listada tem resposta e se a resposta cita Referências primárias de outros autores.

Exemplo: Unidade `M04.U2` (cache como dado derivado) com a pergunta `L04`.

- Entrada: "Cache-aside no redirecionamento, TTL de 1 hora."
- Pergunta gerada: "Um link foi apagado por abuso. Por quanto tempo ele continua redirecionando, e o que você mudaria se a resposta precisasse ser 'imediatamente'?"
- Opção ruim: "Nunca; o cache é apagado junto." Ignora que a invalidação pode falhar ou perder a corrida da gravação dupla.
- Opção boa: "Até o TTL expirar."
- Opção ótima: "A remoção invalida a chave no cache, e o TTL fica como rede de segurança caso a invalidação falhe. O valor do TTL sai do atraso aceitável para o produto."
