# 🗺️ Tracker do Projeto: Boletim Inteligente (Sistemas Distribuídos — Trabalho 3)

> Atualizado após auditoria completa dos 7 repositórios (chat-configs, config-server, eureka-server,
> api-gateway, ms1-coordenador, ms2-ai-powered, docs) cruzada com a descrição oficial do trabalho e
> com o checklist levantado em sala. Reflete a mudança de tema: de "mini-chat" para "professor
> cadastra alunos e lança notas, com um assistente de IA para consultas sobre a turma".
>
> Critérios de avaliação (peso): 1) 12-Factor (1,0) — 2) Gateway/Config/Eureka/GraphQL/Serverless (3,0)
> — 3) Resiliência + Observabilidade (3,0) — 4) Spring AI: Chat Memory/RAG/MCP (3,0).

## Fase 0 — Diagnóstico (já feito, para referência)

O que **já existe e funciona** hoje:
- [x] Organização GitHub (Polyrepo) com 7 repositórios.
- [x] `config-server` (8888) lendo `chat-configs` do GitHub.
- [x] `eureka-server` (8761) — **standalone**, ainda não é cluster.
- [x] `api-gateway` (8080) — discovery locator ligado + **1 rota manual** (`RewritePath` para ms1), Actuator exposto.
- [x] `ms1-coordenador` (8081) — registrado no Eureka, mas só tem um endpoint de teste (`/chat/teste`), **sem lógica de negócio**.
- [x] `ms2-ai-powered` — Spring AI + OpenAI, **RAG** (SimpleVectorStore + TextReader + TokenTextSplitter sobre `conhecimento.txt`), **Chat Memory** (`MessageWindowChatMemory` in-memory), **1 Tool** (`consultarNota`, hoje mockada), **1 MCP client** de terceiro (`@modelcontextprotocol/server-fetch` via stdio/npx).
- [x] Downgrade para Spring Boot 3.3.0/Java 21/Spring Cloud 2023.0.2 em api-gateway, config-server e eureka-server — commitado na Fase 1.
- [x] `docker-compose.yml` (repo `docs`) com PostgreSQL (5433→5432), Zipkin, Prometheus e Grafana, todos validados sobem e ficam `healthy`.
- [x] Todas as dependências de todos os `pom.xml` com `<version>` explícita, conferida via `mvn dependency:tree` (sem mudança de versão resolvida).

O que **ainda não existe** no projeto (Fases 2 em diante):
- Resilience4j (nenhum pom.xml tem a dependência).
- GraphQL (client ou server).
- MS3 / Spring Cloud Function.
- Microserviços containerizados (só as *backing services* — Postgres/Zipkin/Prometheus/Grafana — têm Dockerfile/compose por enquanto).
- Zipkin / micrometer-tracing / Prometheus / Grafana **integrados aos serviços** (a infra já sobe, mas nenhum serviço ainda emite traces/métricas).
- Plano de teste JMeter (`.jmx`).
- Persistência real (Postgres sobe via Docker, mas o MS1 ainda não tem JPA/driver nem entidades).

---

## Fase 1 — Fundação: 12-Factor, versionamento e infraestrutura de apoio (crit. 1) ✅

- [x] Fixar (commitar) o downgrade Boot 3.3.0 / Java 21 / Spring Cloud 2023.0.2 já iniciado. Commitado em api-gateway, config-server e eureka-server (+ mudanças pendentes do ms2: tool calling, MCP client, webflux/actuator).
- [x] Adicionar `<version>` explícita em **todas** as dependências de todos os `pom.xml`. Versões conferidas com `mvn dependency:tree` em cada módulo (compile+test) para garantir que o valor fixado é exatamente o que já vinha sendo resolvido pelas BOMs — zero mudança de comportamento, só remove a ambiguidade.
- [x] Criar `docker-compose.yml` no repo `docs` com PostgreSQL, Zipkin, Prometheus, Grafana. Subido e validado localmente: os 4 containers ficam `healthy`/`Up`. Postgres exposto em `5433` (não `5432`) por já haver um Postgres nativo do host ocupando a porta padrão nesta máquina.
  - Microserviços containerizados ficou **fora** do escopo desta fase (opcional, mencionado no TRACKER original) — os serviços Java continuam rodando via `mvn spring-boot:run` no host por enquanto.
- [x] Padronizar variáveis de ambiente dos backing services via Config Server: `ms2-ai-powered.yml` já usava `${OPENAI_API_KEY:...}`; adicionado o mesmo padrão em `ms1-coordenador.yml` (`POSTGRES_URL`/`POSTGRES_USER`/`POSTGRES_PASSWORD`, com defaults apontando pro compose local). As propriedades `spring.datasource.*`/`spring.jpa.*` ficam inertes até a Fase 2 adicionar `spring-boot-starter-data-jpa` + driver ao pom do MS1 — preparar a convenção agora evita retrabalho.

## Fase 2 — Domínio Acadêmico no MS1 (rebrand leve + persistência)

- [ ] Modelar entidades `Aluno` e `Nota` (JPA) no `ms1-coordenador`.
- [ ] Adicionar PostgreSQL via Spring Data JPA (driver + `spring.datasource.*` no `chat-configs/ms1-coordenador.yml`).
- [ ] Criar `AlunoController` / `NotaController` REST (CRUD: cadastrar aluno, lançar nota, listar/consultar).
- [ ] Substituir `ChatController` (endpoint de teste) pela lógica real de orquestração.
- [ ] Endpoint de pergunta em linguagem natural (`/ms1/perguntar` ou similar) que delega para o MS2.

## Fase 3 — MS3 Serverless (crit. 2)

- [ ] Criar repositório/módulo `ms3-serverless` com Spring Cloud Function.
- [ ] Implementar `Function<NotaInput, NotaResultado>` **validarNota**: valida range 0–10, calcula situação (aprovado/reprovado/recuperação) e frequência mínima. Função pura, sem estado — ideal para demonstrar o conceito de FaaS.
- [ ] Expor via `spring-cloud-function-web` (HTTP) para ser chamada pelo MS1.
- [ ] Integrar a chamada no fluxo de "lançar nota" do MS1 (com Circuit Breaker — ver Fase 5).

## Fase 4 — GraphQL entre MS1 e MS2 (crit. 2)

- [ ] MS2: adicionar `spring-boot-starter-graphql`, definir schema (`.graphqls`) com uma query tipo `perguntar(mensagem: String, chatId: String): String`, ligar ao `AiChatService` já existente.
- [ ] MS1: adicionar client GraphQL (`spring-graphql` `HttpGraphQlClient` ou `WebClient` + `graphql-java` client) para consumir o MS2.
- [ ] Atualizar RAG do MS2: trocar `conhecimento.txt` de "regulamento de Sistemas Distribuídos" para o **regulamento de avaliação da turma** (critérios de aprovação, pesos de prova, frequência mínima) — conteúdo do novo domínio.
- [ ] Tool `consultarNota` do MS2 deixa de ser mockada: chama o MS1 via `WebClient` (REST) para buscar dado real — mantém o princípio de *database per service* (MS2 nunca acessa o Postgres do MS1 diretamente).

## Fase 5 — Resiliência com Resilience4j (crit. 3)

- [ ] Adicionar `resilience4j-spring-boot3` + `spring-boot-starter-aop` no MS1 (e no Gateway, onde aplicável).
- [ ] `@CircuitBreaker` + `fallbackMethod` na chamada MS1→MS2 (GraphQL) e MS1→MS3 (validação de nota).
- [ ] `@Retry` nas mesmas chamadas (cuidado com *retry storm* — configurar backoff exponencial).
- [ ] Rate Limiting: `RequestRateLimiter` no Gateway (server-side, protege o sistema de excesso de requisições) e/ou `@RateLimiter` client-side no MS1.
- [ ] Retry configurado também no **Gateway** (filtro `Retry` nas rotas, conforme pedido no checklist).
- [ ] Expor estado dos circuit breakers via Actuator (`management.health.circuitbreakers.enabled=true`, `management.endpoint.health.show-details=always`) em todos os serviços que os usam.
- [ ] (Opcional, se der tempo) Bulkhead separando a rota de IA (mais lenta) de outras rotas do MS1.

## Fase 6 — Observabilidade (crit. 3)

- [ ] Adicionar `micrometer-tracing-bridge-brave` + `zipkin-reporter-brave` em todos os serviços; subir Zipkin via Docker Compose.
- [ ] Adicionar `micrometer-registry-prometheus` em todos os serviços; expor `/actuator/prometheus`.
- [ ] Subir Prometheus (scrape configs apontando pros `/actuator/prometheus` de cada serviço) e Grafana via Docker Compose.
- [ ] Importar o dashboard oficial do Resilience4j no Grafana (JSON disponível em `resilience4j.readme.io/docs/grafana-1`) para visualizar estado dos circuit breakers em tempo real.
- [ ] Validar o roteiro de demonstração pedido pelo professor: derrubar uma instância → taxa de erro sobe (visível no Grafana) → subir instância nova → taxa de erro cai.

## Fase 7 — MCP próprio + configuração em runtime (crit. 4 + 12-Factor)

- [ ] Criar `mcp-server-academico`: pequeno servidor MCP próprio (Spring AI `spring-ai-starter-mcp-server`) expondo tools do domínio acadêmico (ex.: `listarProximasAvaliacoes`, `buscarRegrasDeAprovacao`).
- [ ] MS2 passa a ter **dois** MCP clients: o de terceiro (`fetch`, já existente) e o próprio (`mcp-server-academico`) — atende literalmente "MCP Server criado pelo aluno e MCP Server de Terceiro".
- [ ] Configuração alterável em tempo de execução: expor `/actuator/refresh` (ou Spring Cloud Bus, se houver tempo) nos serviços com `@RefreshScope`, permitindo alterar uma propriedade no `chat-configs` e refletir sem reiniciar o serviço.
- [ ] Eureka em modo cluster: 2 instâncias peer-aware (via `--spring.profiles.active` com `application-peer1.yml`/`application-peer2.yml`, cada uma apontando `defaultZone` para a outra).
- [ ] Gateway: remover a rota manual (`RewritePath`) e configurar **apenas** via `discovery.locator` + regras no `application.yml` do config-server (nada hard-coded no `api-gateway`), conforme pedido no checklist.

## Fase 8 — Testes de carga e validação final

- [ ] Criar plano JMeter (`.jmx`) cobrindo o fluxo ponta-a-ponta (Gateway → MS1 → MS2/MS3).
- [ ] Rodar o teste com usuários simultâneos > 5 e identificar o **Knee Capacity** e a **Usable Capacity** do sistema.
- [ ] Validar que o Summary Report do JMeter fecha com zero erros em condição normal.
- [ ] Ensaiar o roteiro de falha/recuperação (derrubar instância → religar) com o Grafana aberto, conforme será cobrado na apresentação.
- [ ] Revisar todos os 12 fatores um a um antes da entrega (log como event stream, disposability, etc.).

---

## Notas de decisão (para não esquecer o "porquê")

- **Persistência:** PostgreSQL via Docker no MS1 (não H2, não mock) — decisão tomada para ter um cenário mais realista de "backing service" (fator 12 e critério de resiliência quando o banco também for testado).
- **MS3 Serverless:** função de **validação/cálculo de nota** (range 0–10, situação, frequência) — escolhida por ser uma função pura e stateless, ideal para demonstrar o conceito de FaaS sem ambiguidade com o que o MS2 (IA) já faz.
- **Rebrand:** leve — nomes de repositórios/módulos/pacotes Java permanecem (`ms1-coordenador`, `ms2-ai-powered`), mas classes de domínio (controllers/services/tools) passam a falar de Aluno/Nota em vez de mensagens de chat, para não quebrar as configs de rota/Eureka já validadas.
