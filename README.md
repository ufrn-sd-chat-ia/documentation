# Sistema Distribuído: Boletim Inteligente (Gestão Acadêmica com IA)

Projeto desenvolvido para a disciplina de Sistemas Distribuídos, aplicando os princípios do *Twelve-Factor App* numa arquitetura Cloud Native orientada a microserviços.

## Domínio da Aplicação

O sistema permite que um **professor cadastre alunos, lance notas e faça perguntas em linguagem natural** sobre o desempenho da turma (ex.: "quais alunos estão em recuperação?", "qual a média da turma?"). A infraestrutura de microserviços, resiliência e observabilidade exigida pela disciplina é a mesma independentemente do domínio — só o conteúdo de negócio mudou (era um mini-chat estilo WhatsApp, agora é um boletim acadêmico assistido por IA).

## Estrutura de Repositórios (GitHub Organization: `ufrn-sd-chat-ia`)

A arquitetura adota a abordagem *Polyrepo*, com um repositório dedicado à configuração externa. Os nomes técnicos dos repositórios foram mantidos (rebrand leve) — apenas o domínio de negócio dentro de cada um muda.

1. `chat-configs`: Arquivos `.yml` globais e de roteamento (config-server + rotas do gateway).
2. `config-server`: Central de configuração distribuída.
3. `eureka-server`: Registry de descoberta de serviços (em evolução para modo cluster).
4. `api-gateway`: Porta de entrada única e roteador reativo.
5. `ms1-coordenador`: Serviço acadêmico — CRUD de Aluno/Nota, orquestração e resiliência.
6. `ms2-ai-powered`: Motor de IA (RAG, MCP, Chat Memory, GraphQL server) para perguntas sobre a turma.
7. `ms3-serverless`: *(a criar)* Função Spring Cloud Function para validar/computar uma nota lançada.
8. `docs`: Este repositório — documentação e tracker de progresso.

## Fluxo de Informação (Arquitetura Alvo)

1. O Cliente (professor via REST, ou JMeter no teste de carga) faz uma requisição HTTP para o **API Gateway**.
2. O Gateway consulta o **Eureka** para resolver o endereço do serviço de destino e roteia via *LoadBalancer* — todas as rotas são resolvidas por *discovery* (nenhuma rota manual hard-coded).
3. O **MS1 (Coordenador Acadêmico)**:
   - Faz o CRUD de Aluno/Nota direto no **PostgreSQL** próprio (um banco por serviço).
   - Ao lançar uma nota, invoca o **MS3 (Serverless)** para validar o valor e computar a situação (aprovado/reprovado/recuperação).
   - Quando a requisição é uma pergunta em linguagem natural, encaminha para o **MS2 (Cérebro AI)** usando **GraphQL**.
   - Ambas as chamadas (MS2 e MS3) são protegidas por **Circuit Breaker + Retry** (Resilience4j). Se o serviço de destino falhar, o MS1 devolve uma resposta de fallback graciosamente.
4. O **MS2 (Cérebro AI)** expõe uma API **GraphQL**, usa **RAG** sobre o regulamento de avaliação da disciplina, mantém **Chat Memory** por conversa, e usa **MCP** para invocar tanto um servidor MCP de terceiros quanto um servidor MCP próprio (`mcp-server-academico`).
5. Toda a stack emite traces para o **Zipkin** e métricas (incluindo estado dos circuit breakers) para **Prometheus/Grafana**.

## Mapeamento de Portas e Serviços (Alvo)

| Serviço | Porta | Descrição |
| :--- | :--- | :--- |
| **Config Server** | `8888` | Fornece as propriedades do GitHub. |
| **Eureka Server** | `8761` (+`8762` no 2º nó do cluster) | Registry dos microserviços. |
| **API Gateway** | `8080` | Ponto de entrada público do sistema. |
| **MS1-Coordenador** | `8081` | CRUD de Aluno/Nota + orquestração/resiliência. |
| **MS2-Cérebro AI** | `8082` | GraphQL + RAG + Chat Memory + MCP. |
| **MS3-Serverless** | `8083` | Spring Cloud Function (validação de nota). |
| **mcp-server-academico** | `8084` (ou STDIO) | Servidor MCP próprio, consumido pelo MS2. |
| **PostgreSQL** | `5433` (mapeado do `5432` do container — evita conflito com Postgres nativo do host) | Banco do MS1 (via Docker). |
| **Zipkin** | `9411` | Distributed tracing. |
| **Prometheus** | `9090` | Coleta de métricas. |
| **Grafana** | `3000` | Dashboards (inclui estado dos circuit breakers). |

## Rotas e Endpoints para Teste Rápido

1. **Dashboard do Eureka:** `http://localhost:8761/`
2. **Validação do Config Server:** `http://localhost:8888/ms1-coordenador/default`
3. **Rotas do Gateway (Actuator):** `http://localhost:8080/actuator/gateway/routes`
4. **Health / Circuit Breakers do MS1:** `http://localhost:8081/actuator/health`
5. **Grafana:** `http://localhost:3000/`

> Ver `docs/TRACKER.md` para o checklist detalhado do que já está pronto e o que falta.
