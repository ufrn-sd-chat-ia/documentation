# GraphQL no Boletim Inteligente

## O que é GraphQL

GraphQL é uma linguagem de consulta para APIs (não é um banco de dados, nem um protocolo de transporte — normalmente roda sobre HTTP, como REST). A diferença central pra REST:

- **REST**: o servidor define endpoints fixos (`/alunos`, `/alunos/{id}`, `/notas`), cada um devolvendo uma forma fixa de dado. Se o cliente precisa de menos campos, recebe tudo mesmo assim (*over-fetching*); se precisa de dados de duas entidades relacionadas, faz duas chamadas (*under-fetching* — ex.: buscar aluno, depois buscar as notas dele).
- **GraphQL**: existe **um único endpoint** (normalmente `POST /graphql`). O cliente manda uma *query* dizendo exatamente quais campos quer, e o servidor devolve só isso, numa única resposta, ainda que os dados venham de fontes/objetos diferentes por trás.

Três peças centrais:

1. **Schema (SDL — Schema Definition Language)**: um contrato explícito, escrito num arquivo `.graphqls`, que declara os tipos e operações disponíveis. É a "assinatura" da API, versionada como código.
2. **Query** (ou Mutation, para escritas): a pergunta que o cliente manda, escrita na mesma linguagem do schema.
3. **Resolver**: o método no servidor que sabe *como* buscar o dado de cada campo da query (no Spring GraphQL, um método anotado com `@QueryMapping`/`@Argument`).

## Por que o trabalho pede isso

A descrição do trabalho exige que o **MS1 (coordenador)** seja **cliente GraphQL** e o **MS2 (IA)** seja **servidor GraphQL** — ou seja, a comunicação MS1→MS2 especificamente (não a comunicação Gateway→MS1, que continua REST) tem que passar por GraphQL. Isso é pedido pra demonstrar, na prática, que o grupo sabe **consumir e expor** os dois lados do protocolo dentro de uma arquitetura de microserviços — não é uma exigência de performance ou de modelagem de dados complexa (nosso schema tem só um campo!), é uma exigência de **demonstrar o padrão**.

## Como implementamos

### MS2 — servidor GraphQL

**Schema** (`ms2-ai-powered/src/main/resources/graphql/schema.graphqls`):
```graphql
type Query {
    perguntar(mensagem: String!, chatId: String): String
}
```

**Resolver** (`PerguntaGraphQlController`):
```java
@Controller
public class PerguntaGraphQlController {
    private final AiChatService aiChatService;

    @QueryMapping
    public String perguntar(@Argument String mensagem, @Argument String chatId) {
        return aiChatService.processarMensagem(chatId != null ? chatId : "chat-padrao", mensagem);
    }
}
```

O resolver **não contém lógica de IA nenhuma** — só repassa pro `AiChatService` que já existia (o mesmo que atende o endpoint REST `/ia/perguntar`). GraphQL aqui é uma **porta de entrada nova** pro mesmo serviço, não uma reimplementação.

A dependência `spring-boot-starter-graphql` faz o resto: lê o `.graphqls`, casa o `Query.perguntar` com o método `@QueryMapping`, e expõe tudo em `POST http://localhost:8082/graphql`, aceitando um corpo JSON `{"query": "..."}`.

### MS1 — cliente GraphQL

```java
@Component
public class Ms2GraphQlClient {
    private static final String QUERY = """
            query Perguntar($mensagem: String!, $chatId: String) {
                perguntar(mensagem: $mensagem, chatId: $chatId)
            }
            """;

    private final HttpGraphQlClient graphQlClient;

    public Ms2GraphQlClient(WebClient.Builder loadBalancedWebClientBuilder) {
        WebClient webClient = loadBalancedWebClientBuilder
                .baseUrl("http://ms2-ai-powered/graphql")
                .build();
        this.graphQlClient = HttpGraphQlClient.builder(webClient).build();
    }

    public String perguntar(String mensagem, String chatId) {
        return graphQlClient.document(QUERY)
                .variable("mensagem", mensagem)
                .variable("chatId", chatId)
                .retrieve("perguntar")
                .toEntity(String.class)
                .block();
    }
}
```

Pontos importantes:

- `WebClient.Builder` é `@LoadBalanced` (bean definido em `WebClientConfig`) — o host `ms2-ai-powered` na URL não é DNS nem IP, é o **nome do serviço no Eureka**. O `spring-cloud-starter-loadbalancer` intercepta a chamada e resolve pro endereço real da instância registrada. Isso é o mesmo princípio já usado pelo Gateway (`lb://MS2-AI-POWERED`) e pela chamada MS1→MS3 — nada de host/porta fixo em código.
- `HttpGraphQlClient` é reativo (`WebClient` por baixo), então a chamada devolve um `Mono`. Como o MS1 é uma aplicação servlet (bloqueante, Tomcat), chamamos `.block()` no fim — um padrão comum quando uma app blocking precisa de um client que só existe em versão reativa, sem precisar reescrever o MS1 inteiro pra WebFlux.
- O endpoint REST `GET /perguntar` (em `PerguntaController`) é a porta pública desse fluxo: o Gateway/cliente externo conversa **REST** com o MS1, e só a conversa interna MS1→MS2 é que vira **GraphQL** — exatamente o que o enunciado pede.

### Fluxo completo

```
Cliente/Gateway --REST--> MS1 (/perguntar) --GraphQL--> MS2 (/graphql) --> AiChatService
                                                                              |
                                                                    RAG (conhecimento.txt)
                                                                    Tool consultarNotas --REST--> MS1 (/alunos, /notas)
                                                                    MCP (fetch de terceiro)
```

Repare que o MS2, pra responder, pode voltar a chamar o MS1 — mas aí via **REST** (a tool `consultarNotas` usa `WebClient` comum), não GraphQL. O requisito de GraphQL é só na direção MS1→MS2.

### Onde isso cresce depois (não implementado agora)

Hoje o schema tem um campo só, então o "ganho" clássico do GraphQL (pedir só os campos que precisa, combinar dados de fontes diferentes numa query só) não aparece com força — ele aparece se, no futuro, alguém adicionar mais queries no MS2 (ex.: `explicarRegra(topico: String): String`, `resumoConversa(chatId: String): [String]`) sem precisar criar rotas novas nem versionar endpoints REST: o mesmo `/graphql` absorve tudo, e o cliente pede exatamente os campos que quer numa única ida.
