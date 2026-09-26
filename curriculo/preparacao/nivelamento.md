---
id: nivelamento
status: esqueleto
depende-de: [triagem]
---

# Nivelamento

**Pergunta:** que ferramentas e fundamentos de infraestrutura faltam para você fazer os labs?

## Objetivo

Deixar o aprendiz capaz de operar sozinho o ambiente dos labs: terminal, git, Docker, rede básica e uma VM Linux remota. Cada Unidade só é cursada se a Triagem indicar.

## Competências

| ID | Competência |
|---|---|
| N.C1 | Usar o terminal para navegar, editar arquivos, inspecionar processos e ler logs. |
| N.C2 | Usar git no fluxo da trilha: branch, commit, pull request e conflito simples. |
| N.C3 | Subir, inspecionar e derrubar serviços com Docker e Docker Compose. |
| N.C4 | Explicar o caminho de uma requisição (IP, porta, DNS, TCP, HTTP) e inspecioná-lo com `curl` e `dig`. |
| N.C5 | Operar uma VM Linux remota com segurança (SSH, usuários, serviços, firewall) e acessá-la por Tailscale. |

## Unidades

| ID | Unidade | Objetivo | Competência |
|---|---|---|---|
| N.U1 | Terminal no dia a dia | Navegar, editar, encadear comandos e achar um erro no log de um processo. | N.C1 |
| N.U2 | Git para os labs | Criar branch, fazer commits pequenos, abrir PR e resolver um conflito simples. | N.C2 |
| N.U3 | Docker e Compose | Subir o PostgreSQL e uma API com Compose, ver logs e entrar num container. | N.C3 |
| N.U4 | Redes na prática | Rastrear uma requisição com `dig` e `curl -v` e explicar cada etapa. | N.C4 |
| N.U5 | VM remota com Tailscale | Criar a VM, endurecer o acesso SSH e acessar um serviço dela pela rede Tailscale. | N.C5 |

## No Projeto prático e nos labs

Lab do N.U5: levar o App de estudos e o Hermes para a VM e2-micro do GCP, acessada por Tailscale. É a primeira implantação fora do Mac.

## Fontes de partida

- **Estrutura (Hello Interview):** Core Concepts › Networking Essentials.
- **Texto de base (Primer):** Communication (HTTP, TCP, UDP); Domain name system.
- **Documentação oficial:** git, Docker e Docker Compose, Linux (páginas `man`), nível gratuito do GCP, Tailscale.
