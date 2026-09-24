# SQLite, com log de eventos e colunas JSON, para o estado do aprendiz

Perfil, tentativas de Pílulas, agenda de revisões, Probabilidade de domínio, níveis de competência e XP ficam num único arquivo SQLite: um log de eventos imutável mais tabelas derivadas, com colunas JSON para o que varia por tipo de Pílula. Há um único aprendiz, e as consultas típicas ("revisões vencidas hoje", "taxa de erro por tema em 30 dias", "XP por semana") são cruzamentos e agregações, o ponto forte de SQL.

## Considered Options

- **Banco de documentos (NoSQL)**: atraente pela flexibilidade, mas as colunas JSON já a oferecem; traria um servidor para operar e perderia joins e agregações simples, com o esquema continuando implícito no código.
- **Postgres**: servidor desnecessário para um único usuário.

## Consequences

- Migrar do Mac para a VM significa copiar um arquivo.
