# Roteiro de Testes Manuais — Boletim Inteligente

Cobre o estado do projeto até a **Fase 6** (Fundação, domínio Aluno/Nota, MS3 serverless, GraphQL MS1↔MS2, Resilience4j, Observabilidade).

## 0. Subindo tudo, na ordem certa

```bash
cd docs && docker compose up -d              # Postgres (5433), Zipkin (9411), Prometheus (9090), Grafana (3000)
cd ../config-server && mvn spring-boot:run    # 8888 — espera "Started ConfigServerApplication"
cd ../eureka-server && mvn spring-boot:run    # 8761
cd ../ms3-serverless && mvn spring-boot:run -Dspring-boot.run.arguments="--server.port=8083"
cd ../ms1-coordenador && mvn spring-boot:run  # 8081
cd ../ms2-ai-powered && export $(grep -v '^#' .env | xargs) && mvn spring-boot:run  # 8082 — precisa de OPENAI_API_KEY real
cd ../api-gateway && mvn spring-boot:run      # 8080
```

Cada um em seu próprio terminal (é polyrepo, não tem módulo pai pra subir tudo com um comando só). O `ms2-ai-powered` precisa da variável `OPENAI_API_KEY` de verdade no ambiente — sem ela, o serviço nem termina de subir (a montagem do RAG chama a API de embeddings da OpenAI durante o boot).

---

## 1. Docker / Backing services

| O quê | Como | Espera |
|---|---|---|
| Containers de pé | `docker compose -f docs/docker-compose.yml ps` | 4 containers `Up`/`healthy` |
| Postgres | `docker exec boletim-postgres psql -U boletim -d boletim -c "\dt"` | tabelas `aluno`, `nota` |
| Zipkin (navegador) | http://localhost:9411 | UI abre; depois de gerar tráfego (seção 8), busque por serviço e veja os traces |
| Prometheus (navegador) | http://localhost:9090/targets | os 6 targets `/actuator/prometheus` aparecem **up** |
| Grafana (navegador) | http://localhost:3000 (admin/admin) | dashboard "Resilience4j" já aparece na lista, sem precisar importar nada manualmente |

## 2. Config Server (8888)

```bash
curl -s http://localhost:8888/actuator/health          # {"status":"UP"}
curl -s http://localhost:8888/ms1-coordenador/default   # deve trazer datasource, actuator etc.
curl -s http://localhost:8888/api-gateway/default       # NÃO deve ter bloco "routes" estático
```

## 3. Eureka Server (8761)

- Navegador: http://localhost:8761 — dashboard. Depois de subir todo mundo, "Instances currently registered" deve listar `API-GATEWAY`, `MS1-COORDENADOR`, `MS2-AI-POWERED`, `MS3-SERVERLESS`.

## 4. MS3 Serverless (8083) — Spring Cloud Function isolado

```bash
curl -s -X POST localhost:8083/validarNota -H "Content-Type: application/json" -d '{"valor":8.5,"frequencia":90}'
# {"situacao":"APROVADO","aprovado":true, ...}

curl -s -X POST localhost:8083/validarNota -H "Content-Type: application/json" -d '{"valor":6.0,"frequencia":90}'
# RECUPERACAO

curl -s -X POST localhost:8083/validarNota -H "Content-Type: application/json" -d '{"valor":9.0,"frequencia":50}'
# REPROVADO_POR_FALTA (frequência < 75%, ignora a nota boa)

curl -s -X POST localhost:8083/validarNota -H "Content-Type: application/json" -d '{"valor":15,"frequencia":90}'
# NOTA_INVALIDA

curl -s -o /dev/null -w "%{http_code}\n" localhost:8083/actuator/health   # 200
```

## 5. MS1 Coordenador (8081) — CRUD Aluno/Nota

```bash
# Cadastrar aluno
curl -s -X POST localhost:8081/alunos -H "Content-Type: application/json" \
  -d '{"nome":"Ana Souza","matricula":"20260001","email":"ana@ufrn.edu.br"}'
# 201, id retornado (anote pra usar abaixo)

curl -s localhost:8081/alunos                      # lista
curl -s localhost:8081/alunos/1                     # busca por id
curl -s localhost:8081/alunos/matricula/20260001    # busca por matrícula (usado pelo MS2)

# Erros esperados
curl -i -X POST localhost:8081/alunos -H "Content-Type: application/json" -d '{"nome":"X","matricula":"20260001"}'  # 409 (duplicada)
curl -i localhost:8081/alunos/9999                                                                                   # 404

# Lançar nota (chama o MS3 por baixo pra calcular a situação)
curl -s -X POST localhost:8081/notas -H "Content-Type: application/json" \
  -d '{"alunoId":1,"avaliacao":"Prova 1","valor":8.5,"frequencia":90,"data":"2026-07-01"}'
# 201, "situacao":"APROVADO"

curl -s "localhost:8081/notas?alunoId=1"    # lista notas do aluno
curl -s localhost:8081/actuator/health      # "db":{"status":"UP", ...}
```

**Teste de fallback (importante)**: derrube o `ms3-serverless` (Ctrl+C no terminal dele) e repita o `POST /notas`. Espera **201** (não falha!) com `"situacao":"PENDENTE_VALIDACAO"`. Suba o MS3 de novo depois.

## 6. MS2 AI-Powered (8082) — REST, GraphQL, RAG, Tool, MCP

**REST (como já existia):**
```bash
curl -s "localhost:8082/ia/perguntar?mensagem=qual+a+frequ%C3%AAncia+m%C3%ADnima%3F&chatId=t1"
```

**GraphQL via GraphiQL (navegador, interface visual)**:
http://localhost:8082/graphiql (o navegador segue um redirect automático pra `?path=/graphql`, é normal) — cole no editor:
```graphql
query {
  perguntar(mensagem: "qual a frequência mínima para aprovação?", chatId: "navegador1")
}
```
Clique em "Run" (▶). Deve responder citando os 75%.

**GraphQL via curl:**
```bash
curl -s -X POST localhost:8082/graphql -H "Content-Type: application/json" \
  -d '{"query":"query { perguntar(mensagem: \"qual a nota mínima para aprovação direta?\", chatId: \"t2\") }"}'
# deve responder "7,0" (ou "7.0")
```

**Tool `consultarNotas` (aciona chamada real ao MS1)**:
```bash
curl -s -X POST localhost:8082/graphql -H "Content-Type: application/json" \
  -d '{"query":"query { perguntar(mensagem: \"quais as notas do aluno de matrícula 20260001?\", chatId: \"t3\") }"}'
# a resposta deve listar as notas reais que você cadastrou no passo 5 (prova de que bateu no Postgres via MS1)
```

**MCP (client de terceiro)**: sobe junto com o serviço — confirme nos logs do `mvn spring-boot:run` a linha `Server response with Protocol ... Info: Implementation[name=mcp-fetch ...]`. Não tem endpoint HTTP próprio pra testar via curl (o MCP client fala stdio com o processo `uvx mcp-server-fetch`).

## 7. API Gateway (8080) — roteamento dinâmico e ponto de entrada único

```bash
curl -s localhost:8080/actuator/gateway/routes | python3 -m json.tool
# só rotas dinâmicas: /API-GATEWAY/**, /MS1-COORDENADOR/**, /MS2-AI-POWERED/**, /MS3-SERVERLESS/** -- nenhuma estática

curl -s localhost:8080/MS1-COORDENADOR/alunos                 # rota até o MS1 via gateway
curl -s -X POST localhost:8080/MS3-SERVERLESS/validarNota -H "Content-Type: application/json" -d '{"valor":8,"frequencia":90}'

curl -s -o /dev/null -w "%{http_code}\n" localhost:8080/actuator/env   # 404 esperado (não exposto)
```

## 8. Ponta a ponta: Cliente → Gateway → MS1 → GraphQL → MS2

```bash
curl -s "http://localhost:8080/MS1-COORDENADOR/perguntar?mensagem=quais+alunos+est%C3%A3o+em+recupera%C3%A7%C3%A3o%3F&chatId=e2e1"
```
Isso passa pelas 4 camadas: Gateway resolve via Eureka → MS1 recebe REST → MS1 conversa GraphQL com o MS2 → MS2 roda RAG/Tool/LLM → resposta volta pelo mesmo caminho.

## 9. Resiliência (Resilience4j) — MS1

Estado inicial dos circuit breakers:
```bash
curl -s localhost:8081/actuator/circuitbreakers | python3 -m json.tool
# ms3-validacao e ms2-pergunta, ambos "state":"CLOSED"
```

**Circuit Breaker + Retry (MS3)** — derrube o `ms3-serverless` (Ctrl+C) e:
```bash
# As 2 primeiras chamadas devem demorar ~0.6-0.9s (retry com backoff tentando de verdade)
time curl -s -X POST localhost:8081/notas -H "Content-Type: application/json" \
  -d '{"alunoId":1,"avaliacao":"Teste CB","valor":7,"frequencia":90,"data":"2026-07-05"}'

# Repita mais 3-4 vezes -- a partir da 5a chamada o circuito abre:
curl -s localhost:8081/actuator/circuitbreakers | python3 -c "import sys,json; print(json.load(sys.stdin)['circuitBreakers']['ms3-validacao']['state'])"
# "OPEN"

# Com o circuito OPEN, as chamadas devem ser quase instantâneas (fail-fast, sem retry desperdiçado)
time curl -s -X POST localhost:8081/notas -H "Content-Type: application/json" \
  -d '{"alunoId":1,"avaliacao":"Teste CB 2","valor":7,"frequencia":90,"data":"2026-07-05"}'
# situacao:"PENDENTE_VALIDACAO", tempo ~20ms
```

Suba o `ms3-serverless` de novo, espere uns 30-40s (o Eureka do MS1 atualizar o cache + `wait-duration-in-open-state`), e repita o `POST /notas` 3 vezes — o circuito deve fechar de novo (`HALF_OPEN` → `CLOSED`) e a `situacao` volta a ser calculada de verdade.

**Rate Limiter** — dispare rajadas rápidas (mais que o limite configurado: 5/s em `/notas`, 3/s em `/perguntar`):
```bash
for i in $(seq 1 8); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST localhost:8081/notas -H "Content-Type: application/json" \
    -d "{\"alunoId\":1,\"avaliacao\":\"RL $i\",\"valor\":7,\"frequencia\":90,\"data\":\"2026-07-05\"}"
done
# as primeiras ~5 dão 201, o resto dá 429 com {"erro":"Muitas requisições..."}
```

**Retry no Gateway**: o filtro é nativo do Spring Cloud Gateway, restrito a `GET` (POST não é idempotente — retry automático em `POST /notas` criaria notas duplicadas). Não há um jeito simples de forçar isso via curl sem derrubar uma instância no meio de uma requisição GET; a configuração está em `chat-configs/api-gateway.yml` (`spring.cloud.gateway.default-filters`).

**Bulkhead**: protege `Ms2GraphQlClient` (máx. 5 chamadas concorrentes). Como o rate limiter de `/perguntar` (3/s) é mais restritivo e intervém primeiro, pra isolar o teste suba temporariamente o limite do rate limiter:

```bash
# em chat-configs/ms1-coordenador.yml, troque só pra teste:
#   resilience4j.ratelimiter.instances.pergunta.limit-for-period: 3  ->  50
# reinicie o ms1, depois dispare 10 chamadas concorrentes:
for i in $(seq 1 10); do
  curl -s "localhost:8081/perguntar?mensagem=explique+o+regulamento+numero+$i&chatId=bh$i" > /tmp/bh$i.json &
done
wait
for i in $(seq 1 10); do echo "chamada $i: $(cat /tmp/bh$i.json | head -c 80)"; done
# esperado: exatamente 5 respondem de verdade (ou tentam), 5 voltam com a
# mensagem de fallback "assistente de IA está temporariamente indisponível"
# -- confira o log do ms1 pra ver "Bulkhead 'ms2-pergunta' is full ..." exatamente 5 vezes
grep "Bulkhead" <log do ms1>

# não esqueça de reverter o limit-for-period pra 3 depois do teste
```

## 10. Observabilidade (Zipkin + Prometheus + Grafana)

**Prometheus — todos os targets `up`:**
```bash
curl -s http://localhost:9090/api/v1/targets | python3 -c "
import sys, json
d = json.load(sys.stdin)
for t in d['data']['activeTargets']: print(t['scrapeUrl'], t['health'])
"
```

**Zipkin — trace distribuído completo:** faça uma pergunta via MS1 (`curl "localhost:8081/perguntar?mensagem=...&chatId=x"`), espere uns 3s, e busque em http://localhost:9411 pelo serviço `ms1-coordenador`. Um único trace deve mostrar `ms1-coordenador` → `ms2-ai-powered` com sub-spans de RAG (`embedding`), tool calling (`tool_call consultar-notas`) e chat (`chat gpt-3.5-turbo`) — e se a pergunta disparar a tool, o trace volta pro `ms1-coordenador` (`GET /alunos/matricula/{matricula}`).

**Grafana — dashboard Resilience4j com dado real:**
1. Abra http://localhost:3000 (admin/admin) → dashboard "Resilience4j" já deve estar na lista (provisionado automaticamente, não precisa importar).
2. Selecione `application=ms1-coordenador` e `circuitbreaker_name=ms3-validacao` (ou `ms2-pergunta`) nas variáveis do topo.
3. Gere tráfego (lance notas, derrube o MS3) e veja os painéis mudarem: estado do circuito, taxa de falha, chamadas em buffer.

**Roteiro de falha/recuperação (o que o professor vai cobrar na apresentação):**
```bash
# 1. Estado normal
curl -s -G localhost:9090/api/v1/query --data-urlencode 'query=resilience4j_circuitbreaker_state{name="ms3-validacao",state="closed"}'

# 2. Derrube o ms3-serverless (Ctrl+C) e gere ~6 falhas
for i in $(seq 1 6); do curl -s -X POST localhost:8081/notas -H "Content-Type: application/json" -d "{\"alunoId\":1,\"avaliacao\":\"F$i\",\"valor\":7,\"frequencia\":90,\"data\":\"2026-07-06\"}" >/dev/null; done

# 3. Confirme que abriu (visível também no painel do Grafana)
curl -s -G localhost:9090/api/v1/query --data-urlencode 'query=resilience4j_circuitbreaker_state{name="ms3-validacao",state="open"}'

# 4. Suba o ms3-serverless de novo, espere ~40s (wait-duration + cache do Eureka)
# 5. Gere 3 chamadas de sucesso -- confirme que fechou o circuito de novo
curl -s -G localhost:9090/api/v1/query --data-urlencode 'query=resilience4j_circuitbreaker_state{name="ms3-validacao",state="closed"}'
```

---

## Checklist rápido (o que "tudo OK" significa)

- [ ] Eureka lista os 4 serviços (gateway, ms1, ms2, ms3) como `UP`.
- [ ] Gateway só tem rotas dinâmicas (`/actuator/gateway/routes`).
- [ ] CRUD de Aluno/Nota funciona, incluindo os erros esperados (409/404/400).
- [ ] Nota lançada aparece com `situacao` calculada (MS1→MS3 funcionando).
- [ ] Derrubar o MS3 não quebra o lançamento de nota (fallback `PENDENTE_VALIDACAO`).
- [ ] GraphiQL abre no navegador e responde perguntas de regulamento.
- [ ] Pergunta sobre aluno específico traz nota real do Postgres (tool funcionando).
- [ ] Fluxo completo via Gateway (`/MS1-COORDENADOR/perguntar`) responde.
- [ ] `/actuator/env` dá 404 no Gateway (exposição não vazando).
- [ ] Circuit breaker do MS3 abre depois de ~5 falhas seguidas e volta a fechar quando o MS3 volta.
- [ ] Com o circuito aberto, as respostas são quase instantâneas (fail-fast, sem retry gastando tempo à toa).
- [ ] Rate limiter devolve 429 depois do limite configurado (5/s em `/notas`, 3/s em `/perguntar`).
- [ ] Bulkhead rejeita exatamente as chamadas além do limite de 5 concorrentes (com rate limiter temporariamente elevado pro teste).
- [ ] Prometheus mostra os 6 targets `up`.
- [ ] Zipkin mostra um trace único cobrindo Gateway→MS1→MS2 (ou MS3), sem quebras.
- [ ] Grafana já tem o datasource Prometheus e o dashboard "Resilience4j" sem nenhum passo manual.
- [ ] Derrubar/religar o MS3 reflete em tempo real em `resilience4j_circuitbreaker_state` (e portanto no painel do Grafana).
