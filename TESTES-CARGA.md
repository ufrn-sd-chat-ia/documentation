# Roteiro de Teste de Carga (Fase 8) — JMeter

Plano de teste pronto em [`docs/jmeter/boletim-inteligente.jmx`](jmeter/boletim-inteligente.jmx) (+ [`docs/jmeter/perguntas.csv`](jmeter/perguntas.csv)).
Este documento explica **o que o plano faz, como rodar pela interface gráfica, como ler o resultado** e,
para quem preferir montar tudo do zero na GUI, **item a item o que adicionar** para chegar no mesmo plano.

Todo o tráfego passa pelo **API Gateway (`localhost:8080`)** — nunca bate direto no MS1/MS2/MS3 — porque
é o roteamento dinâmico via Eureka que estamos, também, testando sob carga. Tudo no `.jmx` está com
valores fixos (host, porta, número de threads) direto nos campos — sem variáveis, sem parâmetros de
linha de comando — pra abrir na GUI e já rodar, e pra editar qualquer número direto na tela do elemento.

## 0. Por que o plano é dividido em dois grupos de carga

- **Grupo 1 (CRUD Aluno/Nota)**: é o candidato certo para achar o **Knee Capacity** e a **Usable
  Capacity** do sistema — é barato (Postgres + validação síncrona no MS3), então dá pra escalar threads
  de verdade (10, 20, 30, 40...) sem gastar dinheiro nem levar rate-limit de terceiro.
- **Grupo 2 (`/perguntar`, o caminho de IA)**: propositalmente **pequeno** (3 threads, rodando pra sempre).
  Cada chamada aqui vira uma chamada de verdade pra API da OpenAI (custa dinheiro e tem sua própria
  latência/rate-limit externo) e, além disso, o MS1 já limita essa rota a **3 req/s** por decisão de
  design (`RateLimiter` do Resilience4j, Fase 5) — ou seja, a "capacidade" desse caminho é
  **artificialmente limitada por configuração**, não pelo hardware. Rodar dezenas de threads nele só
  provaria que o rate limiter funciona (o que já validamos manualmente na Fase 5), sem revelar nada novo
  sobre a capacidade real do sistema, e custaria caro. Por isso ele fica separado, em escala pequena, e
  serve para provar que **o caminho de IA continua saudável (RAG + Chat Memory + MCP + GraphQL) mesmo
  sob alguma concorrência**, não para medir capacidade.

## 1. Como rodar (pela GUI)

Pré-requisito: todo o ambiente no ar (ver ordem de subida em `docs/TESTES-MANUAIS.md`, seção 0) — Config
Server, os dois nós do Eureka, Gateway, MS1, MS2, MS3, mcp-server-academico, e os backing services do
`docker-compose.yml` (Postgres, Zipkin, Prometheus, Grafana).

Instalar o JMeter (se ainda não tiver): `sudo apt install jmeter` ou baixar de `jmeter.apache.org`.

Abrir o plano:

```bash
cd docs/jmeter
jmeter
```

E, dentro da GUI: `File → Open` → selecione `boletim-inteligente.jmx`. Aperte o botão verde ▶ (Start) na
barra de ferramentas. Os resultados aparecem ao vivo em três listeners já incluídos, na árvore à
esquerda:

- **View Results Tree** — cada requisição individual, corpo da resposta, código HTTP. Bom pra depurar.
- **Summary Report** — throughput e % de erro agregados por sampler.
- **Aggregate Report** — o mesmo, mas com percentis (p90/p95/p99), que é o que importa pra achar o joelho
  da curva de capacidade.

Para limpar os resultados de uma rodada anterior antes de rodar de novo: ícone da vassourinha
("Clear All") na barra de ferramentas, ou `Ctrl+E`.

> Nota: para números finais "de verdade" (sem o overhead da própria interface do JMeter distorcendo o
> tempo medido), o ideal tecnicamente seria rodar em modo `jmeter -n -t boletim-inteligente.jmx -l
> resultado.jtl`. Não é obrigatório para o que a Fase 8 pede — rodar pela GUI já é suficiente para
> visualizar throughput, erro e latência — mas vale saber que essa opção existe se os números pela GUI
> parecerem inconsistentes com cargas maiores.

## 2. O que o plano faz, passo a passo

1. **`00 - Setup: Criar 20 Alunos`** (`setUp Thread Group`, roda uma vez, antes de tudo, sempre com 1
   thread): cadastra 20 alunos via `POST /alunos` (matrícula `LOAD0001`, `LOAD0002`, ...) para existir
   massa de dados real antes da carga principal. Sem isso, todo `POST /notas` da carga principal falharia
   com 404 (aluno inexistente) em vez de medir o caminho feliz.
2. **`01 - Carga CRUD (Aluno/Nota) - Knee Capacity`** (10 threads, ramp-up 10s, **Loop = Forever** — são
   os números que você muda direto na tela do Thread Group pra escalar a carga): cada iteração faz
   `GET /alunos` (leitura) e depois `POST /notas` (escrita, que dispara MS1 → MS3 síncrono para
   validar/calcular a situação). O `alunoId` (1 a 20) e os valores de nota/frequência são sorteados a
   cada chamada (funções `__Random` / `__groovy`), então cada requisição é levemente diferente — mais
   realista do que repetir sempre o mesmo payload.
3. **`02 - Carga IA (/perguntar)`** (3 threads, ramp-up 5s, **Loop = Forever**): cada thread lê uma
   pergunta de `perguntas.csv` (round-robin) e chama `GET /perguntar?mensagem=...&chatId=carga-<thread>`
   — exercita GraphQL (MS1→MS2), RAG, Chat Memory (um `chatId` fixo por thread, então a conversa acumula
   contexto de verdade) e os dois clients MCP.

Os dois grupos rodam **indefinidamente**, até você clicar no botão vermelho (Stop) — foi assim que os
professores pediram nas outras unidades, justamente pra dar tempo de acompanhar ao vivo, derrubar
instância, subir de novo, e ver o sistema se recuperar sem precisar reiniciar o teste do zero. Não há
nenhum "think time" (pausa) entre requisições — cada thread dispara a próxima chamada assim que a anterior
responde.

## 3. Como achar o Knee Capacity e a Usable Capacity

Como o Grupo 1 agora roda pra sempre, faça isso **antes** da apresentação, uma rodada por linha da
tabela: suba o **"Number of Threads"** (clique no elemento `01 - Carga CRUD...` na árvore e edite o
campo), aperte Start, deixe estabilizar uns 20-30s, anote os números do `Aggregate Report`
(**throughput**, coluna "Throughput", e **tempo médio**, coluna "Average"), aperte Stop, limpe os
resultados (`Ctrl+E`) e repita com o próximo valor de threads. Exemplo de tabela pra preencher:

| threads | throughput (req/s) | tempo médio (ms) | p95 (ms) | erros |
| :--- | :--- | :--- | :--- | :--- |
| 5 | | | | |
| 10 | | | | |
| 20 | | | | |
| 30 | | | | |
| 40 | | | | |
| 50 | | | | |

- **Knee Capacity** = o ponto em que o throughput para de crescer proporcionalmente aos threads, mas o
  tempo de resposta ainda está razoável (o "joelho" da curva). É a carga a partir da qual cada usuário
  adicional deixa de agregar throughput de graça e começa a custar latência.
- **Usable Capacity** = a maior carga em que o tempo de resposta ainda fica dentro de um limite aceitável
  pro usuário (defina um SLA, ex.: p95 < 1s) — normalmente um pouco **abaixo** do knee.
- Se aparecerem erros/429 no `GET /alunos`/`POST /notas` em volumes baixos (5-10 threads), isso não é
  knee capacity — é bug (não há rate limiter nessas rotas) e deve ser investigado antes de continuar.

## 4. Validar "zero erros em condição normal"

Rode com uma carga moderada e realista, entre 5 e o Knee Capacity encontrado na seção 3 (ex.: 10
threads, ramp-up 10s — os valores já deixados como padrão no `.jmx`). No `Summary Report`, olhe a coluna
**Error %** linha a linha:

- **`GET /alunos`** deve fechar em **0%** sempre — essa rota não tem rate limiter, então qualquer erro
  aqui é um problema real. É **esta a linha que representa o "zero erros em condição normal" pedido pelo
  professor.**
- **`POST /notas`** tem um `RateLimiter` de propósito (5 requisições/segundo, Fase 5) — com mais de ~5
  threads disparando sem pausa, é esperado (e correto) que uma fração dessas chamadas volte `429`. **Isso
  não é falha do sistema, é o rate limiter fazendo o trabalho dele.** Se você quer que essa linha também
  feche 100% limpa durante a demonstração (por exemplo, se o professor perguntar sobre ela), duas opções:
  1. **Recomendado, mais simples:** foque a narrativa no `GET /alunos` para o item "zero erros" e "kill de
     instância", e explique o `POST /notas` como uma demonstração *separada e deliberada* do rate limiter
     (a mesma da Fase 5) caso ele pergunte.
  2. **Se preferir que ambas as rotas fechem 100% limpas nesta rodada específica:** suba temporariamente
     `resilience4j.ratelimiter.instances.notas-lancamento.limit-for-period` no `chat-configs` (de `5` para,
     digamos, `50`) e reinicie o MS1 antes da apresentação — igual já foi feito pra testar o Bulkhead na
     Fase 5. Lembre de reverter depois.

> Nota sobre o Grupo 2 (`/perguntar`): com 3 threads e cada chamada real de LLM levando 1-3s pra
> responder, a taxa efetiva já fica naturalmente abaixo de 3 req/s (o limite configurado), então também
> deve ficar perto de 0% de erro sem precisar de nenhum ajuste. Um `429` isolado aqui e ali não é bug.

## 5. Roteiro ao vivo pedido pelo professor: derrubar e recriar instância, com observabilidade

Esse é o roteiro que a apresentação exige literalmente: JMeter rodando pra sempre (Grupo 1 já com Loop =
Forever, threads entre 5 e o Knee Capacity da seção 3) enquanto você/o professor derruba uma instância,
espera a taxa de erro subir, sobe uma instância nova, espera a taxa de erro cair — tudo com uma ferramenta
de observabilidade mostrando o estado do sistema o tempo todo.

**Ponto importante antes de rodar isso pela primeira vez:** a nossa arquitetura tem **dois
comportamentos bem diferentes** dependendo de qual serviço for derrubado, e vale a pena entender os dois
antes da apresentação — inclusive é um ótimo ponto pra citar de próprio punho se o professor perguntar
"por que não subiu erro no JMeter":

### Variante A — derrubar o MS1 (recomendada pra "o Summary Report do JMeter mostra erro subindo")

O MS1 é quem o Gateway chama diretamente — **não existe nenhum Circuit Breaker/fallback entre o Gateway e
o MS1** (o CB só protege as chamadas que *saem* do MS1 pra MS2/MS3). Então, se o MS1 cair, o erro aparece
de verdade no cliente.

1. Com o Grupo 1 rodando (threads entre 5 e o Knee Capacity), deixe uns 15-20s estável — `Summary Report`
   com `GET /alunos` em 0% de erro.
2. Derrube o processo do `ms1-coordenador` (Ctrl+C). O erro aparece **quase instantaneamente** no
   `Summary Report` (conexão recusada — não depende do Eureka perceber a queda, é a tentativa de conexão
   TCP que falha na hora).
3. Suba uma **nova instância** do `ms1-coordenador` (mesmo comando, mesma porta 8081 — `mvn
   spring-boot:run` de novo). Ela se registra no Eureka automaticamente.
4. A taxa de erro no `Summary Report` volta a cair — leve em conta que o Gateway (como qualquer client
   Eureka) só atualiza sua lista de instâncias a cada ~30s por padrão (`eureka.client.registry-fetch-
   interval-seconds`, não foi ajustado neste projeto), então pode levar até meio minuto pra normalizar
   depois do MS1 voltar. Vale narrar isso ("o gateway ainda não atualizou a lista de instâncias") em vez
   de parecer que travou.

**Observabilidade pra essa variante:** como não é sobre Circuit Breaker (o MS1 não tem CB pra si mesmo),
mostre no **Grafana/Prometheus** a métrica padrão de scrape `up{job="ms1-coordenador"}` — ela cai pra `0`
assim que o Prometheus falha em coletar `/actuator/prometheus` do MS1 (a cada `scrape_interval: 5s`) e
volta pra `1` quando a nova instância sobe. Alternativa mais visual: o próprio **dashboard do Eureka**
(`http://localhost:8761`), que mostra a instância de `ms1-coordenador` sumir e reaparecer na lista de
`Instances currently registered with Eureka`.

### Variante B — derrubar o MS3 ou o MS2 (a que o professor citou como exemplo: "status dos circuit breakers")

Aqui o comportamento é **diferente de propósito**: o MS1 tem Circuit Breaker + Retry + fallback gracioso
protegendo as chamadas pra MS3 (`Ms3ValidacaoClient`) e MS2 (`Ms2GraphQlClient`). Isso significa que,
quando o MS3 ou o MS2 caem, **o cliente final (JMeter) continua recebendo respostas de sucesso** (a nota
é salva com `situacao: PENDENTE_VALIDACAO`, ou a pergunta responde com uma mensagem de fallback) — então
o `Summary Report` do JMeter **não mostra a falha**. Isso não é um problema, é o próprio objetivo da
resiliência que a Fase 5 implementou: o usuário final não percebe a queda de uma dependência.

Quem mostra a falha, nesse caso, é o **Grafana** (dashboard Resilience4j), olhando o estado do circuit
breaker correspondente (`ms3-validacao` ou `ms2-pergunta`):

1. Estável: `circuitbreaker_state{state="closed"}=1`.
2. Derrube o `ms3-serverless` (ou `ms2-ai-powered`). Continue gerando carga (Grupo 1 pra MS3, ou Grupo 2
   pra MS2). Depois de 5 chamadas (`minimum-number-of-calls: 5`) com falha ≥ 50%, o estado migra pra
   `open` — visível no painel, junto com `failure_rate` subindo.
3. Suba a instância de volta. Depois do `wait-duration-in-open-state` (10s pro MS3, 15s pro MS2), o
   circuito tenta `half_open` e, com chamadas de sucesso, fecha de novo (`closed=1`).

**Se o professor perguntar por que o JMeter não mostra erro nessa variante**, essa é a resposta certa (e
uma ótima resposta): "essa é exatamente a diferença entre uma dependência falhar e o **usuário perceber**
— o Circuit Breaker com fallback existe pra evitar que a falha de uma dependência interna vire erro pro
cliente; quem quisermos ver o erro subir de verdade no cliente, é o cenário da Variante A (derrubar o
próprio MS1, que não tem ninguém protegendo-o)."

### Sobre o Grupo 2 (IA) nesse roteiro

O Grupo 2 já fica rodando pra sempre em paralelo, com uma escala menor (3 threads) — não porque a demo de
falha/recuperação seja diferente, mas porque **a carga em si precisa ser menor** (rate limiter de 3 req/s
e custo real de chamada de LLM, ver seção 0). Se o professor pedir pra derrubar o `mcp-server-academico`
ou o `ms2-ai-powered` enquanto olha o caminho de IA, é exatamente a Variante B acima (o MS1 protege a
chamada pra MS2 com CB+fallback, então o sinal visível é o Grafana, não o `Summary Report`).

### Recomendação prática

Prepare (e ensaie ao menos uma vez antes da apresentação) **as duas variantes** — elas respondem a leituras
diferentes da mesma instrução do professor, e ter as duas prontas cobre qualquer serviço que ele peça pra
derrubar. Deixe o Grafana e o dashboard do Eureka abertos em abas diferentes, prontos, antes de começar.

Esse roteiro reaproveita e estende o de falha/recuperação da Fase 5/6 (`docs/TESTES-MANUAIS.md`, seções
9-10), agora **sob carga concorrente contínua** (JMeter rodando pra sempre) em vez de chamadas manuais
isoladas — é essa combinação (carga contínua + falha + observabilidade) que a Fase 8 pede.

## 6. Revisão final dos 12 fatores (antes da entrega)

Checklist rápido, um a um — já cobertos ao longo do projeto, aqui é só a conferência final:

| Fator | Onde está |
| :--- | :--- |
| I. Codebase | Um repo Git por serviço (`ufrn-sd-chat-ia`), todos versionados a partir de um único codebase cada. |
| II. Dependencies | Todo `pom.xml` com `<version>` explícita em cada dependência (Fase 1). |
| III. Config | `chat-configs` + Config Server; nada de config hard-coded; `${VAR:default}` em tudo (URLs, senhas, chaves). |
| IV. Backing services | Postgres, Zipkin, Prometheus, Grafana — todos "anexáveis" via URL/porta, trocáveis sem mudar código. |
| V. Build/Release/Run | Maven build separado do `run` (`mvn spring-boot:run` / jar); config injetada em runtime, não no build. |
| VI. Processes | Serviços stateless (estado de sessão de chat fica em memória por instância — ok para o escopo do trabalho, seria Redis em produção real). |
| VII. Port binding | Cada serviço embute seu próprio servidor (Tomcat/Netty), nenhum depende de container externo pra existir. |
| VIII. Concurrency | Escala por processo/instância (múltiplas instâncias de MS1 registradas no Eureka, balanceadas) — testado na Fase 8. |
| IX. Disposability | Start rápido (Spring Boot), shutdown gracioso; Circuit Breaker garante que matar uma dependência não trava o resto. |
| X. Dev/prod parity | Mesmos backing services (Postgres real, não H2) em dev e "prod" local via Docker Compose. |
| XI. Logs | Logs vão para stdout (padrão Spring Boot) — tratados como event stream, sem gerenciamento de arquivo pelo app. |
| XII. Admin processes | `/actuator/refresh`, `/validarNota` (MS3) e as tools MCP rodam como processos one-off contra o mesmo código/config do serviço principal. |

Se algum fator acima não estiver 100% verificável ao vivo, esse é o momento de ajustar antes da entrega —
não durante a apresentação.

---

## 7. Se preferir montar manualmente na GUI do JMeter

Segue a lista exata de elementos, na ordem, com os valores usados no `.jmx` gerado — útil tanto para
montar do zero quanto para **conferir** que o arquivo gerado está correto. Tudo com valores fixos, nada
de variável — edite os números direto nos campos quando quiser mudar a carga.

### 7.1 Elementos de configuração (direto sob o Test Plan, valem para tudo abaixo)
1. **HTTP Request Defaults** (`Add → Config Element → HTTP Request Defaults`): `Server Name = localhost`,
   `Port = 8080`, `Protocol = http`. Isso evita repetir host/porta em cada sampler.
2. **HTTP Header Manager** (`Add → Config Element → HTTP Header Manager`): um header,
   `Content-Type = application/json`.

### 7.2 Grupo "Setup" (dados de teste)
1. `Add → Threads → setUp Thread Group`. Number of Threads = `1`, Loop Count = `20`.
2. Dentro dele, `Add → Config Element → Counter`: Start = `1`, Increment = `1`, Reference Name =
   `contador`, Format = `0000` (fica `0001`, `0002`, ...).
3. `Add → Sampler → HTTP Request`: Method = `POST`, Path = `/alunos`, Body Data (aba "Body Data"):
   ```json
   {"nome":"Aluno Carga ${contador}","matricula":"LOAD${contador}","email":"aluno.carga.${contador}@teste.com"}
   ```
4. Sob o sampler, `Add → Assertions → Response Assertion`: campo "Response Code", padrão `201`.

### 7.3 Grupo "01 - Carga CRUD" (o que você escala pra achar a capacidade)
1. `Add → Threads → Thread Group`. Number of Threads = `10`, Ramp-up Period = `10`, marque **"Infinite"**
   (Loop Count vira irrelevante — o grupo roda até você clicar Stop).
2. `Add → Sampler → HTTP Request`: Method = `GET`, Path = `/alunos`.
3. Outro `Add → Sampler → HTTP Request`: Method = `POST`, Path = `/notas`, Body Data:
   ```json
   {"alunoId":${__Random(1,20,)},"avaliacao":"Prova de Carga","valor":${__groovy(new Random().nextInt(101)/10.0d,)},"data":"2026-07-07","frequencia":${__groovy(70 + new Random().nextInt(31),)}}
   ```
   (o `__groovy` funciona nativo, sem plugin — o JMeter já embute o motor Groovy desde a versão 3.1; é só
   pra variar a nota/frequência sorteada a cada chamada, não precisa entender Groovy pra usar).

### 7.4 Grupo "02 - Carga IA"
1. `Add → Threads → Thread Group`. Number of Threads = `3`, Ramp-up Period = `5`, marque **"Infinite"**.
2. `Add → Config Element → CSV Data Set Config`: Filename = `perguntas.csv`, Variable Names = `mensagem`,
   Delimiter = `;`, Recycle on EOF = `True`.
3. `Add → Sampler → HTTP Request`: Method = `GET`, Path = `/perguntar`, parâmetros (aba "Parameters"):
   `mensagem = ${mensagem}`, `chatId = carga-${__threadNum}`.

### 7.5 Listeners (direto sob o Test Plan, fora dos grupos — assim capturam tudo)
1. `Add → Listener → View Results Tree` (ver cada requisição individual — útil rodando pela GUI).
2. `Add → Listener → Summary Report`.
3. `Add → Listener → Aggregate Report` (tem os percentis p90/p95/p99 que faltam no Summary Report).
