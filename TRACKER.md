# 🗺️ Tracker do Projeto: Chat IA (Sistemas Distribuídos)

## Fase 1: Infraestrutura e Configuração Base (Twelve-Factor App)
- [x] Criar Organização no GitHub (Polyrepo).
- [x] Criar repositório público `chat-configs` para configurações centralizadas.
- [x] Implementar **Config Server** (Porta 8888) integrado com Git.
- [x] Implementar **Eureka Server** (Porta 8761) para Service Discovery.
- [x] Implementar **API Gateway** (Porta 8080) com Cloud LoadBalancer.
- [x] Configurar rotas dinâmicas e ativar o Actuator no Gateway.

## Fase 2: Microserviço 1 - Coordenador do Chat (A Porta de Entrada)
- [x] Criar o projeto `ms1-coordenador` (Porta 8081).
- [x] Registar o MS1 no Eureka e validar o roteamento via Gateway (`/api/ms1/**`).
- [ ] Integrar **Resilience4j** (Exigência de nota).
- [ ] Implementar padrão **Circuit Breaker** nas chamadas externas.
- [ ] Implementar **Fallback Method** (resposta de segurança caso a IA falhe).
- [ ] Expor o status do Circuit Breaker via Actuator para observabilidade.

## Fase 3: Microserviço 2 - Cérebro AI (Spring AI)
- [ ] Criar o projeto `ms2-ai-powered` e registar no Eureka.
- [ ] Configurar conexão com o provedor de LLM (ex: OpenAI / Gemini).
- [ ] Implementar funcionalidade de Chat Básico.
- [ ] Implementar **Chat Memory** (para a IA lembrar do contexto).
- [ ] Implementar **RAG** (Retrieval-Augmented Generation) com leitura de documentos.
- [ ] Implementar **MCP / Tools** (permitir que a IA chame funções ou pesquise na web).

## Fase 4: Microserviço 3 - Função Serverless
- [ ] Criar projeto `ms3-serverless`.
- [ ] Implementar Spring Cloud Function (ex: para sumarizar mensagens longas ou filtrar palavras).
- [ ] Integrar chamada da função no fluxo do MS1.

## Fase 5: Integração e Comunicação
- [ ] Implementar API **GraphQL** (O MS1 deve consultar o MS2 usando GraphQL, conforme slide de requisitos).

## Fase 6: Observabilidade e Testes de Carga (Validação Final)
- [ ] Integrar **Micrometer** e **Zipkin** (Distributed Tracing) em todos os microserviços.
- [ ] Subir painel do **Grafana/Prometheus** para monitorizar a saúde e os Circuit Breakers.
- [ ] Criar script de testes no **JMeter**.
- [ ] Identificar a *Knee Capacity* (ponto de saturação).
- [ ] Validar o fluxo de falha e recuperação: desligar o MS2 e ver o Circuit Breaker abrir no MS1 sem gerar erros no JMeter.