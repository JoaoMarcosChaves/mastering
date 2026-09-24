# Hermes Agent como runtime dos agentes

O Agente tutor, o Agente curador e o Agente validador rodam no Hermes Agent (Nous Research, self-hosted, Python), autenticado pela assinatura Codex/ChatGPT via OAuth. Mantivemos o Hermes mesmo depois de descartar seus diferenciais de mensageria e agendamento, porque queremos memória persistente, subagentes isolados com ferramentas restritas e skills que evoluem com o uso.

## Considered Options

- **Codex SDK chamado direto pelo backend do App de estudos**: menos peças e mesma stack TypeScript dos labs; rejeitado por não trazer subagentes, sandboxes nem evolução de skills prontos.

## Consequences

- Um processo Python convive com uma stack TypeScript.
- Mudanças que o Hermes propõe nas próprias skills chegam como PR com justificativa e só entram após revisão humana.
- A memória nativa do Hermes fica restrita a preferências de conversa; o perfil do aprendiz vive no SQLite do App de estudos, que o entrega ao Hermes como resumo compacto ao abrir a sessão.
- O quanto o Hermes consome dos limites do plano Codex não está documentado; o tutor tem prioridade sobre o curador na cota.
