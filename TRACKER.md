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

## Fase 2 — Domínio Acadêmico no MS1 (rebrand leve + persistência) ✅

- [x] Modelar entidades `Aluno` e `Nota` (JPA) no `ms1-coordenador`.
- [x] Adicionar PostgreSQL via Spring Data JPA (driver `postgresql:42.7.3` + `spring.datasource.*`, já preparado desde a Fase 1).
- [x] Criar `AlunoController` / `NotaController` REST (CRUD: cadastrar aluno, lançar nota, listar/consultar). Validação na borda: matrícula duplicada (409), aluno/nota inexistente (404), campos obrigatórios (400).
- [x] Substituir `ChatController` (endpoint de teste) pela lógica real de orquestração.
- [ ] ~~Endpoint de pergunta em linguagem natural que delega para o MS2~~ — adiado pra Fase 4 de propósito: a integração real é via GraphQL (ainda não existe), então criar uma versão REST provisória aqui seria trabalho jogado fora.

## Fase 3 — MS3 Serverless (crit. 2) ✅

- [x] Criado módulo `ms3-serverless` (ainda sem repositório Git próprio — decisão pendente, ver nota abaixo) com Spring Cloud Function.
- [x] Implementado `Function<NotaInput, NotaResultado>` **validarNota**: valida faixa 0–10, frequência mínima (75%) e calcula situação (APROVADO/RECUPERACAO/REPROVADO/REPROVADO_POR_FALTA/NOTA_INVALIDA). Função pura e sem estado. Limiares (`frequencia-minima`, `media-aprovacao`, `media-recuperacao`) vêm de `@ConfigurationProperties` alimentadas pelo `chat-configs/ms3-serverless.yml` — já prontos pra virar `@RefreshScope` na Fase 7.
- [x] Exposto via `spring-cloud-function-web` em `POST /validarNota` (exigiu adicionar `spring-boot-starter-web` também — `spring-cloud-function-web` sozinho não traz servidor HTTP embutido).
- [x] Integrado ao fluxo de "lançar nota" do MS1: `NotaService` chama o MS3 via `RestClient` `@LoadBalanced` (Eureka) a cada nota lançada. Sem o MS3, a nota ainda é salva com `situacao: PENDENTE_VALIDACAO` (paraquedas manual via try/catch — o Circuit Breaker de verdade é da Fase 5).

## Fase 4 — GraphQL entre MS1 e MS2 (crit. 2) ✅

- [x] MS2: adicionado `spring-boot-starter-graphql`, schema (`graphql/schema.graphqls`) com `perguntar(mensagem: String!, chatId: String): String`, ligado ao `AiChatService` já existente via `PerguntaGraphQlController` (`@QueryMapping`). Exposto em `POST /graphql`.
- [x] MS1: client GraphQL via `spring-graphql` (`HttpGraphQlClient` sobre um `WebClient` `@LoadBalanced`, resolvendo `ms2-ai-powered` via Eureka). Novo endpoint `GET /perguntar?mensagem=...&chatId=...` delega pro MS2 — fecha o item que tinha ficado em aberto da Fase 2.
- [x] RAG do MS2: `conhecimento.txt` trocado do regulamento de Sistemas Distribuídos para o regulamento de avaliação (frequência mínima 75%, critérios de aprovação/recuperação/reprovação — os mesmos números do MS3, pra IA e a validação não divergirem).
- [x] Tool `consultarNota` → `consultarNotas`: deixou de ser mockada, agora busca aluno por matrícula e notas reais no MS1 via `WebClient` `@LoadBalanced` (`GET /alunos/matricula/{matricula}` + `GET /notas?alunoId=`). MS2 nunca acessa o Postgres do MS1 diretamente — mantém *database per service*.
- Validado de ponta a ponta: MS2 GraphQL isolado, MS1→MS2 via GraphQL, tool MS2→MS1 trazendo dado real do Postgres, e o caminho completo Cliente→Gateway→MS1→MS2.

## Fase 5 — Resiliência com Resilience4j (crit. 3) ✅

- [x] `resilience4j-spring-boot3` (2.1.0, gerido pelo BOM do spring-cloud-dependencies) + `spring-boot-starter-aop` no MS1.
- [x] `@CircuitBreaker` na chamada MS1→MS2 (GraphQL, `Ms2GraphQlClient`) e MS1→MS3 (`Ms3ValidacaoClient`), com `fallbackMethod` **só no `@Retry`** (não duplicado no `@CircuitBreaker`) — descoberto na prática que colocar o mesmo fallback nos dois faz o CB engolir a exceção antes do Retry conseguir agir, zerando os retries.
- [x] `@Retry` com backoff exponencial (200ms/300ms, multiplicador 2) nas mesmas chamadas, com `ignore-exceptions` para `CallNotPermittedException` — sem isso o Retry insistia mesmo com o circuito já aberto, anulando o fail-fast.
- [x] Rate limiting via `@RateLimiter` (Resilience4j) em `POST /notas` (5/s) e `GET /perguntar` (3/s) no MS1, com handler dedicado devolvendo 429. Optou-se por isso em vez de `RequestRateLimiter` no Gateway pra não precisar adicionar Redis como nova infra só pra essa funcionalidade.
- [x] Retry no **Gateway**: filtro nativo do Spring Cloud Gateway (`default-filters`), restrito a métodos `GET` — POST (lançar nota, cadastrar aluno) não é idempotente, então retry automático aí criaria duplicatas.
- [x] Actuator expondo `circuitbreakers`, `circuitbreakerevents`, `ratelimiters`, `ratelimiterevents`; `management.health.circuitbreakers.enabled`/`ratelimiters.enabled` ligados.
- [x] Bulkhead (semáforo, 5 chamadas concorrentes) isolando a rota de IA (`Ms2GraphQlClient`) do resto do MS1.
- Validado de ponta a ponta: ciclo completo do circuit breaker (CLOSED→OPEN→fail-fast→HALF_OPEN→CLOSED) derrubando e religando o MS3; retry confirmado por tempo de resposta (~0,6-0,8s com MS3 fora vs ~20ms após o circuito abrir); rate limiter confirmado (excesso de requisições retorna 429). Bulkhead confirmado também: como o rate limiter de `/perguntar` (3/s) é mais restritivo e intervém antes, isolei o teste subindo temporariamente o `limit-for-period` do rate limiter pra 50 só durante o teste (revertido logo depois, não ficou commitado) e disparei 10 chamadas concorrentes — exatamente 5 passaram e 5 voltaram com `"Bulkhead 'ms2-pergunta' is full and does not permit further calls"`, batendo com `max-concurrent-calls: 5`.

## Fase 6 — Observabilidade (crit. 3) ✅

- [x] `micrometer-tracing-bridge-brave` + `zipkin-reporter-brave` nos 6 serviços; Zipkin já estava de pé desde a Fase 1.
- [x] `micrometer-registry-prometheus` nos 6 serviços + `prometheus` na exposição do actuator de cada um (`config-server` e `eureka-server` ganharam exposição própria — o primeiro no `application.yml` local, já que não pode se auto-configurar via Config Server; o segundo em `chat-configs/eureka-server.yml`, novo).
- [x] Sampling de tracing em 100% (`management.tracing.sampling.probability: 1.0`) e endpoint do Zipkin, ambos no `chat-configs/application.yml` (globais, valem pra todos os serviços).
- [x] Dashboard **oficial** do Resilience4j importado de verdade: baixado direto de `github.com/resilience4j/resilience4j/blob/master/grafana_dashboard.json` (o mesmo referenciado em `resilience4j.readme.io/docs/grafana-1`), com `${DS_PROMETHEUS}` substituído pelo uid fixo do datasource provisionado. Provisionamento automático via `docker-compose` (datasource + dashboard sobem sozinhos, sem clique manual no Grafana).
- [x] **Bug real encontrado e corrigido**: os traces se quebravam entre MS1→MS3 e MS1→MS2 (cada chamada virava um trace novo, sem correlação). Causa: os beans `RestClient.Builder`/`WebClient.Builder` `@LoadBalanced` eram criados via `RestClient.builder()`/`WebClient.builder()` puro, sem passar pelas customizações automáticas do Boot (`RestClientBuilderConfigurer`/`WebClientCustomizer`) que injetam o `ObservationRestClientCustomizer`/`ObservationWebClientCustomizer` responsáveis por propagar o contexto de trace. Corrigido injetando e aplicando esses configurers/customizers nos 3 beans manuais (`ms1-coordenador` × 2, `ms2-ai-powered` × 1). Depois da correção, um único `traceId` cobre Gateway → MS1 → MS3 e também Gateway → MS1 → MS2 (incluindo sub-spans do Spring AI: RAG, tool call, chat, embeddings).
- [x] Roteiro de demonstração validado de verdade nas métricas do Prometheus (as mesmas que alimentam o Grafana): MS3 no ar → `resilience4j_circuitbreaker_state{state="closed"}=1`; derrubei o MS3 e gerei tráfego → `state="open"=1`, `failure_rate=80%`; religuei o MS3 → depois do `wait-duration-in-open-state` + cache do Eureka atualizar, `half_open=1`; tráfego de sucesso → `closed=1` de novo, `failure_rate` resetado.

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
