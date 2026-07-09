## Visão geral do fluxo

```
Cliente (professor / JMeter)
       │
       ▼
┌──────────────────┐        ┌───────────────────┐
│   API Gateway    │◄──────►│  Eureka (cluster) │
│      :8080       │        │  peer1:8761       │
└────────┬─────────┘        │  peer2:8762       │
         │ roteia via       └───────────────────┘
         │ discovery (lb://)              ▲
         ▼                                │            todos os serviços
┌──────────────────┐   REST    ┌────────────────────┐  se registram e
│ MS1-Coordenador  │──────────►│  MS3-Serverless    │  se descobrem aqui
│      :8081       │◄──────────│      :8083         │
│ (Aluno/Nota/JPA) │  situação │ (valida nota, FaaS)│
└────────┬─────────┘           └────────────────────┘
         │ GraphQL
         ▼
┌────────────────────┐   MCP/SSE  ┌──────────────────────┐
│  MS2-AI-Powered    │───────────►│ mcp-server-academico │
│      :8082         │            │        :8084         │
│ (RAG, Chat Memory, │            │ (tools de regras de  │
│  Tool, MCP client) │            │  avaliação)          │
└────────┬───────────┘            └──────────────────────┘
         │ MCP/stdio
         ▼
  mcp-server-fetch (terceiro, via uvx)

Todos os serviços ────► Zipkin (traces) + Prometheus (métricas) ────► Grafana (dashboards)
Todos os serviços ────► Config Server (:8888) ────► chat-configs (Git)
```

Todos os 7 componentes rodam em processos separados (polyrepo — cada um é um repositório git independente), simulando microsserviços de verdade rodando em máquinas/containers diferentes.

## Os 12 Fatores — como o projeto atende

| Fator | Como atendemos |
|---|---|
| **I. Codebase** | 	Polyrepo: 1 repositório por serviço (api-gateway, config-server, eureka-server, ms1, ms2, ms3, mcp-server-academico), todos sob a mesma organização |
| **II. Dependencies** | Toda dependência tem <version> explícita no pom.xml de cada serviço, nada resolvido silenciosamente por BOM |
| **III. Config** | Porta, datasource, chave da OpenAI, regras de negócio (RegrasAvaliacaoProperties) vêm do Config Server (chat-configs, repositório git separado) ou de variáveis de ambiente — nunca hardcoded. @RefreshScope aplicado onde a config pode mudar em runtime (ex: regras de aprovação no MS3) |
| **IV. Backing services** | Postgres (MS1), Zipkin, Prometheus, Grafana são serviços "anexados" via Docker Compose, todos trocáveis por config sem mudar código |
| **V. Build, release, run** | Maven builda o jar; roda com `spring-boot:run`/`java -jar`; config injetada em runtime, não embutida no build |
| **VI. Processes** | MS1/MS3/Gateway são stateless. Exceção conhecida: Chat Memory do MS2 é InMemoryChatMemoryRepository (RAM do processo), keyed por chatId: funciona bem com 1 instância, mas não seria compartilhado se o MS2 escalasse horizontalmente. É uma limitação documentada, não escondida |
| **VII. Port binding** | Cada serviço expõe sua própria porta via Tomcat embutido (Spring Boot), sem depender de um container externo pra existir |
| **VIII. Concurrency** | Escalabilidade horizontal via múltiplas instâncias registradas no mesmo Eureka, balanceadas pelo Gateway — testado nos ciclos de kill/restart do JMeter |
| **IX. Disposability** | Boot rápido, shutdown limpo; testado repetidamente subindo/derrubando MS1 e MS3 durante os testes de resiliência sem efeitos colaterais |
| **X. Dev/prod parity** | Mesmos serviços de apoio (Postgres, Zipkin, Prometheus, Grafana) via Docker Compose em qualquer ambiente — só variáveis de ambiente mudam |
| **XI. Logs** | Logs vão para stdout (padrão Spring Boot), correlacionados por trace ID via Zipkin/Micrometer Tracing |
| **XII. Admin processes** | Tarefas administrativas via Actuator: /actuator/refresh (recarrega config), /actuator/health, /actuator/circuitbreakers, /actuator/ratelimiters, /actuator/prometheus — nenhum script ad-hoc |

## Comandos no terminal
- Ordem para subir os serviços e fazer setup

```bash
cd docs && docker compose up -d
cd ../config-server && mvn spring-boot:run
cd ../eureka-server && mvn spring-boot:run -Dspring-boot.run.profiles=peer1
cd ../eureka-server && mvn spring-boot:run -Dspring-boot.run.profiles=peer2
cd ../mcp-server-academico && mvn spring-boot:run
cd ../ms3-serverless && mvn spring-boot:run -Dspring-boot.run.arguments="--server.port=8083"
cd ../ms1-coordenador && mvn spring-boot:run
cd ../ms2-ai-powered && export $(grep -v '^#' .env | xargs) && mvn spring-boot:run
cd ../api-gateway && mvn spring-boot:run
```

```bash
for p in 8888 8761 8762 8080 8081 8082 8083 8084; do
  curl -s -o /dev/null -w "porta $p -> %{http_code}\n" http://localhost:$p/actuator/health
done
```

```bash
curl -s -X POST http://localhost:8081/alunos -H "Content-Type: application/json" \
  -d '{"nome":"Aluno Carga","matricula":"LOAD1","email":"aluno.carga@teste.com"}'
```

Url das ferramentas para abrir no navegador (Zen - ZIPKIN não funciona bem no meu chrome)
1. Zipkin — `http://localhost:9411`
2. Grafana — `http://localhost:3000`, dashboard **Resilience4j** já aberto
3. Eureka peer1 — `http://localhost:8761`

- Atualizar média no chat-configs/application.yml
Testar antes e depois de mudar
```bash
curl -s -X POST http://localhost:8081/notas -H "Content-Type: application/json" \
  -d '{"alunoId":1,"avaliacao":"Demo Refresh","valor":6.5,"data":"2026-07-08","frequencia":90}'
```

```bash
curl -s -X POST http://localhost:8083/actuator/refresh   # ms3-serverless
curl -s -X POST http://localhost:8084/actuator/refresh   # mcp-server-academico
```

- Verificar resultado do MS3 e fallback
Testar antes, durante e depois de derrubar
```bash
curl -s -X POST http://localhost:8081/notas -H "Content-Type: application/json" \
  -d '{"alunoId":1,"avaliacao":"Teste CB","valor":8,"data":"2026-07-08","frequencia":90}'
```

- Verificar resultado ao derrubar MS1
Verificar a query no Grafana/Explore/Prometheus antes e depois
```promql
up{job="ms1-coordenador"}
```

- Testes MS2 - IA
  - Testar separadamente o mcp externo para buscar uma url
```bash
curl -s "http://localhost:8080/MS1-COORDENADOR/perguntar?mensagem=Acesse%20https%3A%2F%2Fhttpbin.org%2Fuuid%20e%20me%20diga%20o%20UUID%20que%20apareceu&chatId=teste-mcp-fetch"
```
  - Testar MCP próprio
```bash
curl -s -X POST localhost:8082/graphql -H "Content-Type: application/json" \
  -d '{"query":"query { perguntar(mensagem: \"o que significa a situação REPROVADO_POR_FALTA?\", chatId: \"mcp-academico-2\") { mensagem } }"}'
```

  - Testar rag
```bash
curl -s -X POST localhost:8082/graphql -H "Content-Type: application/json" \
  -d '{"query":"query { perguntar(mensagem: \"qual é a palavra-chave secreta mencionada no regulamento para pedir revisão de nota?\", chatId: \"rag3\") { mensagem } }"}'
  ```
# esperado: menciona "BOLETIM2026" -- esse texto só existe no seu conhecimento.txt

- Testar chatmemory
1.
```bash
curl -s -X POST localhost:8082/graphql -H "Content-Type: application/json" \
  -d '{"query":"query { perguntar(mensagem: \"Meu nome é Professora Ana. Guarde isso.\", chatId: \"memoria1\") { mensagem } }"}'
```

2.
```bash
curl -s -X POST localhost:8082/graphql -H "Content-Type: application/json" \
  -d '{"query":"query { perguntar(mensagem: \"Qual é o meu nome?\", chatId: \"memoria1\") { mensagem } }"}'
```
# esperado: responde "Ana" -- prova que lembrou do turno anterior
