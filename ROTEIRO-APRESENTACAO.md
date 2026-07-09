# Roteiro do Dia da Apresentação

Sequência linear pra seguir ao vivo na frente do professor — cada passo tem o comando exato, o que
esperar, e por quê. Assume que você já fez o levantamento de Knee Capacity (`docs/TESTES-CARGA.md`,
seção 3) antes da apresentação e já sabe o número. Todos os valores de threads/rate limiter abaixo
refletem o estado atual do projeto (rate limiter `notas-lancamento` em 50/s, `pergunta` em 3/s).

## 0. Antes de começar (pré-voo, uns 10-15 min antes)

**Suba tudo, na ordem** (`docs/TESTES-MANUAIS.md`, seção 0):
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

**Confirme que todo mundo respondeu** (copia e cola, espera um `200` em cada linha):
```bash
for p in 8888 8761 8762 8080 8081 8082 8083 8084; do
  curl -s -o /dev/null -w "porta $p -> %{http_code}\n" http://localhost:$p/actuator/health
done
```

**Garanta que existe pelo menos 1 aluno** (senão `POST /notas` dá 404 "aluno não encontrado"):
```bash
curl -s -X POST http://localhost:8081/alunos -H "Content-Type: application/json" \
  -d '{"nome":"Aluno Carga","matricula":"LOAD1","email":"aluno.carga@teste.com"}'
```
Se já existir (matrícula duplicada, 409), ignora — já tem aluno.

**Abas do navegador, já abertas, nesta ordem** (não perca tempo abrindo ao vivo):
1. Zipkin — `http://localhost:9411`
2. Grafana — `http://localhost:3000`, dashboard **Resilience4j** já aberto
3. Eureka peer1 — `http://localhost:8761`
4. JMeter — plano `docs/jmeter/boletim-inteligente.jmx` já carregado

**JMeter — conferir o estado dos grupos antes de começar:**
- `Setup: Criar 1 Aluno` → pode deixar **desabilitado** (já tem aluno).
- `Carga CRUD (Aluno/Nota)` (`GET /alunos`) → **habilitado**, ajuste "Number of Threads" pra um valor
  entre 5 e o Knee Capacity que você levantou antes (ex.: se o knee for ~80, use algo como 30-40).
- `POTS /notas` (`POST /notas`) → **habilitado**, o Constant Throughput Timer dentro dele decide a
  taxa real — deixe em `2400`/min (40 req/s, abaixo do limite de 50/s) pro baseline limpo.
- `Carga IA (/perguntar)` → deixe **desabilitado** até o passo 6 (custa chamada real de LLM).

---

## Passo 1 — Baseline: carga contínua, zero erros

**O que fazer:** no JMeter, aperte ▶ (Start). Deixe rodando uns 20-30s.

**O que olhar:** no `Summary Report`, as linhas `GET /alunos` e `POST /notas` devem fechar em **0% Error**.

**Explicação pro professor:** "estou com N usuários simultâneos, número escolhido entre 5 e o Knee
Capacity que levantei antes — o sistema aguenta essa carga indefinidamente sem erro, é o baseline
saudável." Se aparecer erro aqui, **pare e investigue antes de continuar** — não é esperado.

**Não pare o JMeter** — ele fica rodando (Loop = Forever) durante todos os próximos passos.

---

## Passo 2 — Configuração em tempo de execução (Config Server refresh)

Com a carga ainda rodando ao fundo, mostre que dá pra mudar uma regra de negócio sem reiniciar nada.

**Comando 1 — mostrar o comportamento atual:**
```bash
curl -s -X POST http://localhost:8081/notas -H "Content-Type: application/json" \
  -d '{"alunoId":1,"avaliacao":"Demo Refresh","valor":6.5,"data":"2026-07-08","frequencia":90}'
```
**Esperado:** `"situacao":"RECUPERACAO"` (nota 6,5 está entre 5,0 e 7,0, o corte atual).

**Comando 2 — edite o valor no `chat-configs/application.yml`** (repo separado, fora do editor se
quiser, ou local): troque `media-aprovacao: 7.0` por `media-aprovacao: 6.0`. Se estiver usando o
Config Server em modo `native` local, salve o arquivo; se for GitHub, faça commit+push antes.

**Comando 3 — dispare o refresh nos dois serviços que têm essa propriedade:**
```bash
curl -s -X POST http://localhost:8083/actuator/refresh   # ms3-serverless
curl -s -X POST http://localhost:8084/actuator/refresh   # mcp-server-academico
```
**Esperado:** cada um responde uma lista JSON com `["boletim.avaliacao.media-aprovacao"]` — é a
confirmação de que a property realmente mudou.

**Comando 4 — repita a mesma nota 6,5, sem reiniciar nada:**
```bash
curl -s -X POST http://localhost:8081/notas -H "Content-Type: application/json" \
  -d '{"alunoId":1,"avaliacao":"Demo Refresh","valor":6.5,"data":"2026-07-08","frequencia":90}'
```
**Esperado:** `"situacao":"APROVADO"` — mesma nota, resultado diferente, zero restart.

**Explicação pro professor:** "`@RefreshScope` no bean de propriedades + `/actuator/refresh` — o Spring
Cloud Context relê o Config Server e substitui só esse bean, não o processo inteiro. Só funciona nos
serviços que têm essa anotação: `ms3-serverless` e `mcp-server-academico` (os dois compartilham a mesma
regra `boletim.avaliacao.*`, por isso preciso dar refresh nos dois)."

**Reverta depois** (`media-aprovacao: 7.0` de novo + refresh nos dois), pra não bagunçar os próximos
passos.

---

## Passo 3 — Circuit Breaker abrindo (derrubar o MS3) — visível no Grafana

Com o JMeter ainda rodando (o `POST /notas` já está gerando chamadas MS1→MS3 continuamente):

**Comando — derrube o MS3** (Ctrl+C no terminal dele, ou):
```bash
# no terminal onde o ms3-serverless está rodando, aperte Ctrl+C
```

**O que olhar no Grafana** (dashboard Resilience4j, variável `circuitbreaker_name = ms3-validacao`):
- Painel **"Failure Rate"**: sobe rápido, de 0% pra próximo de 100%.
- Painel **"CircuitBreaker States"**: depois de 5 chamadas falhadas (`minimum-number-of-calls: 5`),
  muda de `closed` pra `open`.

**O que olhar no JMeter** (`Summary Report`, linha `POST /notas`): continua em **sucesso** (não vira
erro!) — cada resposta agora vem com `"situacao":"PENDENTE_VALIDACAO"`. Confirme com:
```bash
curl -s -X POST http://localhost:8081/notas -H "Content-Type: application/json" \
  -d '{"alunoId":1,"avaliacao":"Teste CB","valor":8,"data":"2026-07-08","frequencia":90}'
```
**Esperado:** HTTP 201, `"situacao":"PENDENTE_VALIDACAO"`.

**Explicação pro professor — o ponto mais importante desse passo:** "o cliente não vê erro nenhum
porque o Circuit Breaker + fallback existem exatamente pra isso: isolar a falha de uma dependência
interna, sem propagar pro usuário final. Quem mostra a falha de verdade é o Grafana, olhando o estado
interno do circuito — essa é a diferença entre 'uma dependência cair' e 'o usuário perceber'."

---

## Passo 4 — Recuperação do Circuit Breaker

**Comando — suba o MS3 de novo:**
```bash
cd ms3-serverless && mvn spring-boot:run -Dspring-boot.run.arguments="--server.port=8083"
```

**O que olhar no Grafana:** depois de `wait-duration-in-open-state` (10s) + o Eureka/LoadBalancer
perceberem a instância de volta (pode levar mais alguns segundos), o estado migra `open → half_open →
closed`. Narre esse intervalo em vez de ficar em silêncio ("está esperando os 10 segundos do
wait-duration, e mais um pouco pro load balancer atualizar a lista de instâncias").

**Confirme com uma chamada:**
```bash
curl -s -X POST http://localhost:8081/notas -H "Content-Type: application/json" \
  -d '{"alunoId":1,"avaliacao":"Pos Recovery","valor":8,"data":"2026-07-08","frequencia":90}'
```
**Esperado:** `"situacao":"APROVADO"` de novo (validação real, não mais fallback).

---

## Passo 5 — Erro de verdade no JMeter (derrubar/subir o MS1)

Esse é o passo que satisfaz literalmente "a taxa de erro deve subir quando desligar uma instância, e
cair quando subir de novo" **no `Summary Report` do JMeter** — diferente do Passo 3 (onde o JMeter não
mostrava erro nenhum, de propósito). A diferença: o MS1 é o próprio serviço que o Gateway chama
diretamente, sem nenhum Circuit Breaker/fallback o protegendo.

**Comando — derrube o MS1** (Ctrl+C no terminal dele).

**O que olhar no JMeter:** `Summary Report`, linhas `GET /alunos` e `POST /notas` — o **Error %** sobe
quase instantaneamente (conexão recusada, não depende de o Eureka perceber a queda).

**O que olhar como observabilidade** (já que não é sobre circuit breaker aqui): Prometheus/Grafana —
vá em **Explore** e rode:
```promql
up{job="ms1-coordenador"}
```
**Esperado:** cai de `1` pra `0` assim que o MS1 cai. Alternativa visual: dashboard do Eureka
(`http://localhost:8761`) — a instância de `MS1-COORDENADOR` some da lista "Instances currently
registered with Eureka".

**Comando — suba o MS1 de novo:**
```bash
cd ms1-coordenador && mvn spring-boot:run
```

**O que esperar:** o erro no JMeter cai de volta a 0%, mas **não instantaneamente** — o Gateway (como
qualquer client Eureka) só atualiza sua lista de instâncias a cada ~30s por padrão
(`eureka.client.registry-fetch-interval-seconds`, não foi ajustado neste projeto). Narre isso
("aguardando o Gateway atualizar a lista de instâncias do Eureka") em vez de parecer que travou. O
`up{job="ms1-coordenador"}` no Prometheus volta a `1` bem mais rápido (nos ~5s do scrape interval) —
pode mostrar essa métrica primeiro, pra provar que o MS1 já está de pé, enquanto o Gateway ainda não
percebeu.

---

## Passo 6 — Rate Limiter (mostrar rejeição de verdade)

**No JMeter:** abra `POTS /notas` → `POST /notas` → o Constant Throughput Timer dentro dele. Mude
**"Target throughput (in samples per minute)"** de `2400` pra `3600` (60 req/s — acima do limite de
50/s do servidor).

**O que olhar no JMeter:** `Summary Report`, linha `POST /notas` — passa a ter uma fatia de erro (os
`429`).

**O que olhar no Grafana (Explore, Prometheus):**
```promql
rate(http_server_requests_seconds_count{uri="/notas", status="429"}[30s])
```
**Esperado:** linha bem acima de zero, mostrando os `429`/segundo de verdade.
```promql
resilience4j_ratelimiter_available_permissions{name="notas-lancamento"}
```
**Esperado:** oscila entre 0 e 50 — quando bate 0, a próxima chamada naquele segundo é rejeitada.

**Explicação pro professor:** "o rate limiter tem `timeout-duration: 0s` — não tem fila, é rejeição
imediata. Isso é intencional: numa API de escrita, preferimos falhar rápido a deixar requisições
acumulando e derrubando o serviço."

**Volte o target pra `2400`** depois, pra não deixar o resto da demo com erro de fundo.

---

## Passo 7 — Rota de IA: rate limiter, custo da OpenAI, e "não gasta chave quando rejeita"

**No JMeter:** habilite o grupo `Carga IA (/perguntar)` (3 threads, roda junto dos outros).

**Deixe rodando uns 15-20s**, depois olhe o `Summary Report` — deve fechar perto de 0% de erro (a
latência real do LLM, ~1-3s por chamada, já mantém a taxa natural abaixo do limite de 3 req/s).

**Pra mostrar o rate limiter rejeitando** (sem gastar mais chamadas de LLM que o necessário): suba
temporariamente o "Number of Threads" desse grupo pra uns 10-15 — o excesso acima de 3/s começa a
tomar `429`, e — ponto importante — **isso acontece antes de qualquer chamada à OpenAI**, então as
chamadas rejeitadas não custam nada. Explique isso citando o código: o `@RateLimiter(name = "pergunta")`
está anotado direto no método do `PerguntaController` (MS1), que roda antes de qualquer chamada de
rede sair em direção ao MS2/OpenAI.

**Depois, volte pra 3 threads** e desabilite o grupo de novo (economizar chamadas de LLM pro resto da
apresentação).

---

## Passo 8 — Tracing distribuído no Zipkin

Com qualquer carga rodando (ideal: os grupos CRUD, já que são mais rápidos de achar um trace completo):

1. Abra `http://localhost:9411`, **Service Name = ms1-coordenador**, **Run Query**.
2. Clique num trace da lista.
3. **Esperado:** árvore `Gateway → MS1 → MS3` (ou `→ MS2`, se a IA estiver ligada), um único `traceId`
   do início ao fim, com o tempo gasto em cada span.

**Explicação pro professor:** "cada requisição ganha um `traceId` na entrada do Gateway, propagado via
headers B3/`traceparent` em cada chamada HTTP subsequente — dá pra ver exatamente onde o tempo foi
gasto numa requisição que atravessa 3 serviços."

---

## Passo 9 — Rotas do Gateway: dinâmicas, e só o MS1 é público

```bash
curl -s http://localhost:8080/actuator/gateway/routes | python3 -m json.tool
```
**Esperado:** só uma rota, `/MS1-COORDENADOR/**`.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/MS2-AI-POWERED/ia/perguntar
```
**Esperado:** `404` — confirma que não dá mais pra chamar o MS2 direto pelo Gateway.

**Explicação pro professor:** "as rotas são 100% dinâmicas via `discovery.locator` do Eureka — mas
restringi com `include-expression` só ao MS1, porque MS2/MS3 não têm proteção própria (nem
Resilience4j no `pom.xml`); sem essa restrição, dava pra chamar a rota de IA direto no MS2 e furar o
rate limiter que protege o custo da OpenAI."

---

## Fechamento — checklist rápido dos 12 Fatores

Ver tabela completa em `docs/APRESENTACAO.md`, seção 10. Se sobrar tempo, é um bom jeito de encerrar
("resumindo os 12 fatores...") em vez de terminar abruptamente no último `curl`.

---

## Se algo der errado no meio da apresentação

- **`POST /notas` retorna 404 "aluno não encontrado"** → o Postgres foi resetado; rode o comando de
  criar aluno da seção 0 de novo.
- **JMeter mostrando erro em tudo, do nada** → confira se o Zipkin não crashou (`docker logs
  boletim-zipkin --tail 20`, procure `OutOfMemoryError`) — já corrigimos isso com `JAVA_OPTS` no
  `docker-compose.yml`, mas se ainda acontecer, `docker compose restart zipkin` resolve em segundos e
  não afeta o resto do sistema.
- **Erro "Connection refused" vindo do Eureka peer2** → é só o outro nó do cluster não estar de pé;
  suba ele (`-Dspring-boot.run.profiles=peer2`) se for demonstrar o cluster também.
