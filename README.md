# Sistema Distribuído: Chat AI-Powered

Projeto desenvolvido para a disciplina de Sistemas Distribuídos, aplicando os princípios do *Twelve-Factor App* numa arquitetura Cloud Native orientada a microserviços.

## Estrutura de Repositórios (GitHub Organization)

A arquitetura adota a abordagem *Polyrepo*, com um repositório dedicado à configuração externa:
1. `chat-configs`: Arquivos `.yml` globais e de roteamento.
2. `config-server`: Central de configuração distribuída.
3. `eureka-server`: Registry de descoberta de serviços.
4. `api-gateway`: Porta de entrada única e roteador reativo.
5. `ms1-coordenador`: Serviço principal de orquestração de mensagens.
6. `ms2-ai-powered`: (*Em breve*) Motor de IA (RAG, MCP, Memory).
7. `ms3-serverless`: (*Em breve*) Funções auxiliares sob demanda.

## Fluxo de Informação (Arquitetura)

1. O Cliente (ou JMeter) faz uma requisição HTTP para o **API Gateway**.
2. O Gateway consulta o **Eureka** para resolver o endereço do **MS1**.
3. O Gateway utiliza o *LoadBalancer* interno para encaminhar a requisição para o **MS1**.
4. O **MS1 (Coordenador)** recebe a mensagem e:
   - Se a mensagem for muito longa, invoca o **MS3 (Serverless)** para sumarização.
   - Envia a mensagem processada para o **MS2 (Cérebro AI)** utilizando **GraphQL**.
   - É protegido por um **Circuit Breaker**. Se o MS2 falhar, o MS1 devolve uma resposta de fallback graciosamente.

## Mapeamento de Portas e Serviços

| Serviço | Porta | Descrição |
| :--- | :--- | :--- |
| **Config Server** | `8888` | Fornece as propriedades do GitHub. |
| **Eureka Server** | `8761` | Lista telefónica dos microserviços (Registry). |
| **API Gateway** | `8080` | Ponto de entrada público do sistema. |
| **MS1-Coordenador** | `8081` | Recebe mensagens e gere a resiliência. |
| **MS2-Cérebro AI** | `(a definir)` | Processamento com Spring AI. |
| **MS3-Serverless** | `(a definir)` | Cloud function. |

## Rotas e Endpoints para Teste Rápido

Para garantir que a infraestrutura base está funcional, execute os serviços pela ordem abaixo e aceda às seguintes rotas:

1. **Dashboard do Eureka:**
 `http://localhost:8761/`
   *(Verifique se API-GATEWAY e MS1-COORDENADOR estão registados).*

2. **Validação do Config Server:**
  `http://localhost:8888/api-gateway/default`
   *(Deve retornar o JSON com as propriedades lidas do GitHub).*

3. **Observabilidade das Rotas (Gateway Actuator):**
 `http://localhost:8080/actuator/gateway/routes`
   *(Deve listar as regras de redirecionamento ativas).*

4. **Teste de Fluxo Ponta-a-Ponta (Client -> Gateway -> MS1):**
 `http://localhost:8080/api/ms1/teste`
   *(Deve retornar a mensagem de sucesso originada no Controller do MS1).*