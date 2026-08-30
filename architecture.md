# Arquitetura — Dia D Simulation Platform

## Sumário

1. [Objetivo da arquitetura](#1---objetivo-da-arquitetura)
2. [Visão arquitetural](#2---visão-arquitetural)
3. [Contextos da plataforma](#3---contextos-da-plataforma)
4. [Fluxo arquitetural geral](#4---fluxo-arquitetural-geral)
5. [Papel do Orchestrator](#5---papel-do-orchestrator)
6. [Orquestração e coreografia](#6---orquestração-e-coreografia)
7. [Fluxo pós-prova orquestrado](#7---fluxo-pós-prova-orquestrado)
8. [Arquitetura interna dos microservices](#8---arquitetura-interna-dos-microservices)
9. [Arquitetura interna do Orchestrator](#9---arquitetura-interna-do-orchestrator)
10. [Comunicação síncrona](#10---comunicação-síncrona)
11. [Comunicação assíncrona](#11---comunicação-assíncrona)
12. [Eventos, comandos e workflows](#12---eventos-comandos-e-workflows)
13. [Estado e consistência distribuída](#13---estado-e-consistência-distribuída)
14. [Outbox e Inbox](#14---outbox-e-inbox)
15. [Idempotência](#15---idempotência)
16. [Autoridade de domínio](#16---autoridade-de-domínio)
17. [Autoridade de tempo](#17---autoridade-de-tempo)
18. [Ciclo de vida da aplicação](#18---ciclo-de-vida-da-aplicação)
19. [Políticas e estratégias](#19---políticas-e-estratégias)
20. [Persistência e isolamento](#20---persistência-e-isolamento)
21. [Redis](#21---redis)
22. [Segurança arquitetural](#22---segurança-arquitetural)
23. [Resiliência e tratamento de falhas](#23---resiliência-e-tratamento-de-falhas)
24. [Escalabilidade e concorrência](#24---escalabilidade-e-concorrência)
25. [Observabilidade distribuída](#25---observabilidade-distribuída)
26. [Contratos](#26---contratos)
27. [Configuração](#27---configuração)
28. [Infraestrutura e implantação](#28---infraestrutura-e-implantação)
29. [Regras arquiteturais obrigatórias](#29---regras-arquiteturais-obrigatórias)
30. [Decisões arquiteturais](#30---decisões-arquiteturais)

---

## 1 - Objetivo da arquitetura

A arquitetura da **Dia D Simulation Platform** deve sustentar uma plataforma capaz de executar aplicações de prova com comportamento próximo ao de um evento real, mantendo isolamento entre domínios, rastreabilidade, escalabilidade e capacidade de evolução.

A arquitetura deve permitir que diferentes tipos de exame sejam suportados sem transformar o sistema em uma implementação específica do ENEM.

Os principais direcionadores arquiteturais são:

```text
Microservices
Arquitetura Hexagonal
Event-Driven Architecture
API Gateway
Orquestração de workflows distribuídos
Database per Service
Kafka
RabbitMQ
Redis
Outbox / Inbox
Idempotência
Observabilidade distribuída
Backend como autoridade da aplicação
Políticas configuráveis por tipo de prova
Escalabilidade horizontal
```

A inclusão do `diad-orchestrator-service` introduz uma fronteira específica para coordenação de processos distribuídos.

Ele não substitui os serviços de domínio.

Sua responsabilidade é acompanhar e coordenar workflows que atravessam vários contextos.

---

## 2 - Visão arquitetural

A plataforma será composta por frontend, gateway, serviços independentes, infraestrutura de mensageria, cache, persistência e componentes de observabilidade.

## 2.1 Visão Geral totalmente amplificada:
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/93d24172-763a-43d6-8abd-f7ea84a55e3e" />

## 2.1 Visão simplificada:

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/1b641866-daf7-4980-b750-e44b4dfac909" />

Essa visão não significa que toda comunicação passe obrigatoriamente pelo Orchestrator.

O Orchestrator participa somente dos workflows que exigem coordenação explícita.

---

## 3 - Contextos da plataforma

A arquitetura separa responsabilidades em contextos especializados.

### Identidade

```text
diad-auth-service
diad-candidate-service
```

O primeiro controla identidade técnica e autorização.

O segundo controla o perfil de negócio do candidato.

### Definição da prova

```text
diad-exam-service
diad-question-service
```

O `exam-service` define a estrutura e as políticas.

O `question-service` mantém o conteúdo das questões.

### Aplicação

```text
diad-application-service
diad-answer-service
diad-security-service
```

Esses serviços participam diretamente da execução da prova.

### Coordenação

```text
diad-orchestrator-service
```

Mantém o estado de workflows distribuídos e coordena a continuidade entre contextos.

### Pós-prova

```text
diad-scoring-service
diad-performance-service
diad-ranking-service
```

Responsáveis respectivamente por nota, diagnóstico e classificação.

### Comunicação

```text
diad-communication-service
```

Responsável pela entrega de notificações através de provedores externos.

---

## 4 - Fluxo arquitetural geral

O fluxo de alto nível da plataforma é:

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/ddf4640b-1cf3-494e-82c9-e39593b0e218" />

Durante a prova, o `diad-application-service` permanece como autoridade sobre a aplicação.

Após eventos que iniciem processos distribuídos, o Orchestrator pode assumir a coordenação do workflow sem assumir a propriedade das regras dos serviços envolvidos.

---

## 5 - Papel do Orchestrator

O `diad-orchestrator-service` existe para coordenar processos de negócio distribuídos cuja conclusão depende de múltiplos serviços.

Ele funciona como um **Process Manager / Saga Orchestrator**.

Responsabilidades:

- iniciar uma instância de workflow;
- persistir o estado do workflow;
- identificar a etapa atual;
- solicitar a execução da próxima etapa;
- consumir eventos de conclusão;
- controlar timeout;
- controlar tentativas;
- impedir processamento duplicado;
- retomar workflows interrompidos;
- registrar falhas;
- concluir workflows;
- manter rastreabilidade ponta a ponta.

Princípio:

```text
ORCHESTRATOR
     │
     ├── coordena
     ├── acompanha
     ├── correlaciona
     └── decide a próxima etapa

NÃO:
     ├── calcula nota
     ├── calcula ranking
     ├── gera relatório
     ├── corrige questão
     ├── decide permanência mínima
     └── executa regra pertencente a outro domínio
```

A regra arquitetural é:

> **Orchestrator coordena. Serviços de domínio decidem.**

---

## 6 - Orquestração e coreografia

A plataforma utilizará os dois modelos.

### Coreografia

Adequada para eventos independentes
<img width="1183" height="1330" alt="image" src="https://github.com/user-attachments/assets/877db4ab-872a-43aa-b88a-32b400fcd1ec" />

Nenhum serviço central controla todo o fluxo.


### Orquestração

Utilizada quando existe uma sequência de negócio que precisa ser acompanhada.

```text
              Orchestrator
                   │
             comando/solicitação
                   ▼
                Service
                   │
                  evento
                   ▼
              Orchestrator
                   │
             próxima etapa
```

Critério:

```text
Evento independente
→ Coreografia

Workflow com múltiplas etapas dependentes
→ Orquestração
```

O objetivo é evitar dois extremos:

```text
tudo centralizado no Orchestrator
```

e

```text
uma cadeia de eventos impossível de acompanhar
```

---

## 7 - Fluxo pós-prova orquestrado

O principal workflow inicialmente coordenado será o pós-prova.

```text
ApplicationFinished
        │
        ▼
POST_EXAM_WORKFLOW_STARTED
        │
        ▼
ScoreCalculated
        │
        ▼
PerformanceReportGenerated
        │
        ▼
RankingUpdated
        │
        ▼
AnswerKeyAccessEvaluated
        │
        ▼
ResultAvailable
        │
        ▼
CommunicationRequested
        │
        ▼
POST_EXAM_WORKFLOW_COMPLETED
```

Visão por serviços:

```text
Application Service
       │
       │ ApplicationFinished
       ▼
      Kafka
       │
       ▼
Orchestrator Service
       │
       ├──► Scoring Service
       │       │
       │       └── ScoreCalculated
       │
       ├──► Performance Service
       │       │
       │       └── PerformanceReportGenerated
       │
       ├──► Ranking Service
       │       │
       │       └── RankingUpdated
       │
       ├──► Application Service
       │       │
       │       └── AnswerKeyAccessEvaluated
       │
       └──► Communication Service
```

O Orchestrator deve avançar somente após receber a confirmação esperada da etapa anterior quando houver dependência entre elas.

Se etapas puderem ocorrer em paralelo futuramente, o workflow poderá possuir ramificações.

Exemplo:

```text
                 ScoreCalculated
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
        Performance           Ranking
             │                   │
             └─────────┬─────────┘
                       ▼
                ResultAvailable
```

A estratégia exata poderá evoluir sem alterar a responsabilidade dos serviços.

---

## 8 - Arquitetura interna dos microservices

Cada microservice seguirá Arquitetura Hexagonal.

```text
src/main/kotlin/com/diadsimulation/{service}/
│
├── core/
│   ├── common/
│   ├── domain/
│   │   ├── model/
│   │   ├── exception/
│   │   └── valueobject/
│   │
│   ├── port/
│   │   ├── input/
│   │   └── output/
│   │
│   └── usecase/
│
└── app/
    ├── adapter/
    │   ├── input/
    │   │   ├── web/
    │   │   └── messaging/
    │   │
    │   └── output/
    │       ├── persistence/
    │       ├── messaging/
    │       └── client/
    │
    └── configuration/
```

Fluxo interno:

```text
ENTRADAS
   │
   ├── REST Controller
   └── Message Consumer
           │
           ▼
       INPUT PORT
           │
           ▼
        USE CASE
           │
           ▼
         DOMAIN
           │
           ▼
       OUTPUT PORT
           │
     ┌─────┼─────────┐
     ▼     ▼         ▼
Repository Producer Client
 Adapter    Adapter   Adapter
```

O domínio não deve conhecer:

```text
Spring
HTTP
Kafka
RabbitMQ
JPA
Redis
provedores externos
```

---

## 9 - Arquitetura interna do Orchestrator

O Orchestrator também seguirá Hexagonal Architecture.

Porém, seu domínio representa processos, não entidades dos demais serviços.

Estrutura conceitual:

```text
core/
├── domain/
│   ├── model/
│   │   ├── Workflow
│   │   ├── WorkflowStep
│   │   └── ProcessInstance
│   │
│   ├── valueobject/
│   │   ├── WorkflowId
│   │   ├── CorrelationId
│   │   └── WorkflowType
│   │
│   └── exception/
│
├── port/
│   ├── input/
│   │   ├── StartWorkflowUseCase
│   │   └── HandleWorkflowEventUseCase
│   │
│   └── output/
│       ├── WorkflowRepositoryPort
│       ├── EventPublisherPort
│       └── ClockPort
│
└── usecase/
```

Fluxo interno:

```text
ApplicationFinished Consumer
            │
            ▼
       INPUT PORT
            │
            ▼
      StartWorkflow
            │
            ▼
        Workflow
            │
            ├── persistir estado
            │
            └── publicar próxima ação
                    │
                    ▼
              OUTPUT PORT
```

Exemplo de estado:

```text
workflowId
workflowType
aggregateId
correlationId
currentStep
status
startedAt
updatedAt
completedAt
retryCount
failureReason
```

Estados possíveis:

```text
PENDING
RUNNING
WAITING
COMPLETED
FAILED
TIMED_OUT
CANCELLED
```

---

## 10 - Comunicação síncrona

REST será utilizado quando o cliente precisa de resposta imediata.

Exemplo:

```text
Client
  │
  ▼
Gateway
  │
  ▼
Service
  │
  ▼
Response
```

Casos típicos:

```text
login
consulta de perfil
consulta de provas
inscrição
consulta da sessão
carregamento de questões
envio/alteração de resposta
consulta de resultado
consulta de relatório
consulta de ranking
```

Chamadas síncronas entre microservices devem ser utilizadas somente quando a operação realmente exigir resposta imediata.

---

## 11 - Comunicação assíncrona

Kafka será utilizado principalmente para eventos de domínio e integração assíncrona.

```text
Service
   │
   ▼
Outbox
   │
   ▼
Kafka
   │
   ├── Consumer A
   ├── Consumer B
   └── Orchestrator
```

RabbitMQ será utilizado principalmente para filas de trabalho.

```text
Communication Service
        │
        ▼
     RabbitMQ
        │
        ├── Email Worker
        ├── WhatsApp Worker
        └── Retry / DLQ
```

Separação conceitual:

```text
Kafka
→ algo aconteceu

RabbitMQ
→ algo precisa ser executado
```

---

## 12 - Eventos, comandos e workflows

Eventos representam fatos ocorridos.

Exemplos:

```text
ApplicationFinished
ScoreCalculated
PerformanceReportGenerated
RankingUpdated
AnswerKeyAccessGranted
ResultAvailable
```

Comandos representam intenção de execução.

Exemplos:

```text
CalculateScoreRequested
GeneratePerformanceReportRequested
UpdateRankingRequested
EvaluateAnswerKeyAccessRequested
SendResultNotificationRequested
```

Fluxo:

```text
EVENTO
ApplicationFinished
        │
        ▼
Orchestrator
        │
        ▼
COMANDO
CalculateScoreRequested
        │
        ▼
Scoring Service
        │
        ▼
EVENTO
ScoreCalculated
```

Essa distinção deve permanecer explícita nos contratos.

---

## 13 - Estado e consistência distribuída

Não haverá transação ACID única atravessando vários microservices.

Cada serviço confirma sua própria transação local.

```text
Service A DB
   ✓ commit

Service B DB
   ✓ commit

Service C DB
   ✓ commit
```

A consistência entre serviços será eventual.

O Orchestrator mantém o estado do processo, mas não transforma o sistema em uma transação distribuída tradicional.

Exemplo:

```text
Workflow = WAITING_SCORE

ScoreCalculated recebido

Workflow = WAITING_PERFORMANCE
```

Isso permite recuperar o fluxo caso um serviço ou broker fique temporariamente indisponível.

---

## 14 - Outbox e Inbox

### Outbox

Mudança de estado e evento devem ser persistidos na mesma transação local.

```text
BEGIN

UPDATE domain_state

INSERT outbox_event

COMMIT
```

Posteriormente:

```text
Outbox Publisher
      │
      ▼
    Kafka
```

Evita:

```text
banco atualizado
+
evento perdido
```

### Inbox

Consumers deverão registrar eventos processados.

```text
eventId
consumer
processedAt
```

Antes do processamento:

```text
eventId já existe?
    │
    ├── SIM → ignorar
    └── NÃO → processar
```

O Orchestrator também utilizará Inbox para impedir que o mesmo evento avance um workflow duas vezes.

---

## 15 - Idempotência

Idempotência é obrigatória em operações críticas.

Exemplos:

```text
finalização da aplicação
envio de respostas
cálculo de nota
avanço de workflow
geração de relatório
atualização de ranking
comunicações
```

Um mesmo evento poderá ser entregue mais de uma vez.

O resultado final deverá permanecer consistente.

Exemplo:

```text
ScoreCalculated(eventId=123)
ScoreCalculated(eventId=123)

→ apenas um avanço do workflow
```

---

## 16 - Autoridade de domínio

Cada serviço é autoridade apenas sobre seu próprio contexto.

```text
Auth
→ credenciais e autorização

Candidate
→ perfil do candidato

Exam
→ definição da prova

Question
→ conteúdo das questões

Application
→ aplicação e sessão

Answer
→ respostas

Security
→ eventos de integridade

Orchestrator
→ estado dos workflows

Scoring
→ pontuação

Performance
→ análise de desempenho

Ranking
→ classificação

Communication
→ entrega de comunicação
```

Nenhum serviço acessa diretamente o banco de outro serviço.

O Orchestrator não é exceção.

```text
Orchestrator ─X─> scoring_db
Orchestrator ─X─> ranking_db
Orchestrator ─X─> application_db
```

Ele conhece contratos, eventos, comandos e identificadores.

Não conhece a persistência interna dos outros contextos.

---

## 17 - Autoridade de tempo

O tempo oficial da aplicação pertence ao backend.

Nunca ao dispositivo do candidato.

```text
serverTime
applicationStartsAt
applicationEndsAt
sessionExpiresAt
```

O frontend recebe referências temporais e apenas representa o cronômetro.

```text
Backend
   │
   │ serverTime + applicationEndsAt
   ▼
Frontend
   │
   ▼
Cronômetro visual
```

Ao executar operações críticas, o backend valida novamente o horário.

Isso evita que a alteração do relógio local modifique as regras da prova.

---

## 18 - Ciclo de vida da aplicação

Estados previstos:

```text
REGISTERED
    │
    ▼
WAITING
    │
    ▼
AVAILABLE
    │
    ▼
IN_PROGRESS
    │
    ▼
FINISHED
```

Estados alternativos:

```text
EXPIRED
CANCELLED
DISQUALIFIED
```

Durante `IN_PROGRESS`:

```text
Application
     │
     ├── Answer Service
     ├── Security Service
     └── Controle de tempo
```

Ao chegar em `FINISHED`:

```text
ApplicationFinished
        │
        ▼
Kafka
        │
        ▼
Orchestrator
        │
        ▼
Workflow pós-prova
```

---

## 19 - Políticas e estratégias

As regras de prova devem ser configuráveis e extensíveis.

```text
ApplicationPolicy
├── StartPolicy
├── EndPolicy
├── MinimumStayPolicy
├── ExitPolicy
├── SecurityPolicy
├── SessionPolicy
├── SubmissionPolicy
└── AnswerKeyAccessPolicy
```

Pontuação:

```text
ScoringStrategy
├── SimpleScore
├── WeightedScore
├── NegativeScore
├── CebraspeScore
├── TriScore
├── PercentageScore
└── CustomScore
```

Evitar:

```kotlin
if (examType == "ENEM") { ... }
else if (examType == "OAB") { ... }
else if (examType == "CEBRASPE") { ... }
```

O Orchestrator não seleciona nem executa essas políticas.

Ele reage ao resultado produzido pelo serviço responsável.

---

## 20 - Persistência e isolamento

Será adotado:

```text
Database per Service
```

Exemplo:

```text
auth_db
candidate_db
exam_db
question_db
application_db
answer_db
orchestrator_db
scoring_db
performance_db
ranking_db
security_db
communication_db
```

Cada serviço possui:

```text
migrations próprias
schema próprio
repository próprio
ciclo de evolução independente
```

O `orchestrator_db` persiste apenas estado de coordenação.

Ele não funciona como banco central da plataforma.

---

## 21 - Redis

Redis será um componente de apoio.

Possíveis utilizações:

```text
cache
rate limiting
sessões temporárias
locks distribuídos
ranking de leitura rápida
dados efêmeros da aplicação
estado temporário quando apropriado
```

Dados críticos do workflow não deverão depender exclusivamente de Redis.

O estado necessário para recuperação do Orchestrator deverá possuir persistência durável.

---

## 22 - Segurança arquitetural

O fluxo externo será:

```text
Client
   │ JWT
   ▼
API Gateway
   │
   ├── valida token
   ├── rate limit
   ├── CORS
   ├── correlationId
   └── roteamento
```

O Gateway é uma fronteira técnica.

Não deve executar regra de negócio.

O Orchestrator é um componente interno e não deve ser utilizado pelo frontend como API de domínio para controlar workflows.

Princípios:

```text
least privilege
segredos fora do código
validação de entrada
proteção de endpoints
auditoria
propagação segura de identidade
rate limiting
mascaramento de dados sensíveis
```

---

## 23 - Resiliência e tratamento de falhas

Falhas devem ser tratadas como parte normal de um sistema distribuído.

Possíveis situações:

```text
serviço indisponível
timeout
evento duplicado
evento fora de ordem
broker temporariamente indisponível
consumer reiniciado
workflow interrompido
provedor externo indisponível
```

Estratégias:

```text
timeout
retry com backoff
DLQ
idempotência
Outbox
Inbox
health checks
circuit breaker quando aplicável
reprocessamento controlado
```

No Orchestrator:

```text
RUNNING
   │
   ▼
etapa falha
   │
   ├── retry permitido → nova tentativa
   │
   ├── timeout → TIMED_OUT
   │
   └── falha definitiva → FAILED
```

A recuperação deverá ser possível sem reiniciar todo o processo.

---

## 24 - Escalabilidade e concorrência

O sistema deverá suportar picos concentrados de acesso.

Esse cenário é especialmente relevante porque milhares de candidatos podem:

```text
entrar na plataforma
iniciar a prova
carregar questões
salvar respostas
finalizar a prova
```

em janelas de tempo próximas.

Os serviços deverão ser preparados para escalabilidade horizontal.

```text
             Load Balancer
                  │
          ┌───────┼───────┐
          ▼       ▼       ▼
       Instance Instance Instance
```

Consumers Kafka devem utilizar particionamento adequado.

Eventos de uma mesma entidade devem preservar uma chave consistente quando a ordem for necessária.

Exemplos:

```text
applicationId
candidateId
workflowId
```

O Orchestrator também deverá permitir múltiplas instâncias sem que duas instâncias avancem simultaneamente o mesmo workflow.

Isso poderá exigir:

```text
controle otimista de versão
locks apropriados
idempotência
particionamento por workflowId
```

---

## 25 - Observabilidade distribuída

A arquitetura deverá permitir acompanhar uma operação entre vários serviços.

Exemplo:

```text
Frontend
   │
Gateway
   │
Application
   │
Kafka
   │
Orchestrator
   │
Scoring
   │
Performance
```

Todos devem preservar contexto de rastreamento.

Campos:

```text
correlationId
traceId
spanId
eventId
workflowId
workflowType
workflowStep
candidateId
applicationId
examId
```

Métricas específicas do Orchestrator:

```text
orchestrator_workflows_started_total
orchestrator_workflows_completed_total
orchestrator_workflows_failed_total
orchestrator_workflows_timed_out_total
orchestrator_workflow_duration_seconds
orchestrator_workflow_step_duration_seconds
orchestrator_workflow_retries_total
```

O objetivo é permitir responder:

```text
Qual workflow falhou?
Em qual etapa?
Qual evento iniciou o processo?
Quanto tempo cada etapa levou?
Qual serviço não respondeu?
Quantas tentativas ocorreram?
O candidato recebeu o resultado?
```

---

## 26 - Contratos

Contratos devem ser explícitos e versionados.

### REST

```text
OpenAPI
```

### Mensageria

```text
AsyncAPI
Event Schemas
Command Schemas
```

O repositório:

```text
diad-shared-contracts
```

centralizará contratos compartilhados.

Não deverá conter regra de domínio.

Envelope de evento conceitual:

```json
{
  "eventId": "uuid",
  "eventType": "ScoreCalculated",
  "eventVersion": 1,
  "occurredAt": "timestamp",
  "correlationId": "uuid",
  "causationId": "uuid",
  "aggregateId": "uuid",
  "payload": {}
}
```

Para workflows, o contexto poderá carregar:

```text
workflowId
```

quando necessário.

---

## 27 - Configuração

O `diad-config-server` será responsável por configuração centralizada.

Configurações poderão incluir:

```text
URLs internas
timeouts
feature toggles
parâmetros técnicos
configurações de mensageria
limites operacionais
```

Segredos não deverão ser tratados como configuração comum versionada.

---

## 28 - Infraestrutura e implantação

O repositório:

```text
diad-platform-infra
```

centralizará infraestrutura da plataforma.

Ambiente local:

```text
Docker Compose
```

Ambientes distribuídos:

```text
Kubernetes
```

Componentes:

```text
PostgreSQL
Redis
Kafka
RabbitMQ
Prometheus
Grafana
OpenTelemetry
```

Visão:

```text
                   Kubernetes
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   Microservices     Brokers        Data
        │              │              │
        │          Kafka/Rabbit     DB/Redis
        │
        ▼
 OpenTelemetry
        │
        ├── Prometheus
        └── Grafana
```

---

## 29 - Regras arquiteturais obrigatórias

As seguintes regras devem permanecer válidas durante a evolução do projeto:

```text
1. Controller não acessa repository diretamente.

2. Consumer não contém regra de negócio.

3. DTO não entra no domínio.

4. Entidade JPA não é modelo de domínio.

5. Domínio não depende de Spring.

6. Domínio não conhece Kafka.

7. Domínio não conhece RabbitMQ.

8. Domínio não conhece HTTP.

9. Toda entrada passa por input port.

10. Toda saída passa por output port.

11. Um serviço não acessa o banco de outro serviço.

12. API Gateway não contém regra de negócio.

13. Orchestrator não contém regra pertencente aos serviços coordenados.

14. Orchestrator coordena workflows, não substitui serviços de domínio.

15. Eventos devem possuir eventId.

16. Consumers críticos devem ser idempotentes.

17. correlationId deve ser propagado.

18. Workflows devem possuir workflowId.

19. Estado crítico de workflow deve ser persistido de forma durável.

20. Eventos simples devem preferir coreografia.

21. Workflows distribuídos complexos podem utilizar orquestração.

22. Backend é autoridade sobre o tempo da prova.

23. Frontend não contém regra crítica de negócio.

24. Regras específicas de exames devem utilizar políticas/estratégias.

25. Integrações externas devem ser abstraídas por portas.
```

---

## 30 - Decisões arquiteturais

### ADR-001 — Microservices

A plataforma será dividida por responsabilidades de domínio para permitir evolução e escalabilidade independentes.

### ADR-002 — Arquitetura Hexagonal

Cada serviço terá domínio isolado de frameworks e infraestrutura.

### ADR-003 — Database per Service

Nenhum serviço acessará diretamente a persistência de outro contexto.

### ADR-004 — Event-Driven Architecture

Eventos serão utilizados para desacoplar processos assíncronos e representar fatos relevantes da plataforma.

### ADR-005 — Kafka para eventos

Kafka será utilizado para eventos de domínio e integração assíncrona.

### ADR-006 — RabbitMQ para filas de trabalho

RabbitMQ será utilizado principalmente para processamento de tarefas, retry e DLQ.

### ADR-007 — Backend como autoridade de tempo

O relógio do cliente nunca determinará o estado oficial da aplicação.

### ADR-008 — Policies e Strategies

Variações entre tipos de prova serão implementadas por abstrações configuráveis, evitando condicionais espalhadas pelo código.

### ADR-009 — Outbox / Inbox

Operações assíncronas críticas utilizarão padrões de confiabilidade e deduplicação.

### ADR-010 — Orchestrator Service

Será criado:

```text
diad-orchestrator-service
```

para coordenar workflows distribuídos que necessitem de acompanhamento explícito.

O serviço será um Process Manager / Saga Orchestrator.

### ADR-011 — Orchestrator não possui domínio alheio

O Orchestrator não calculará nota, ranking, desempenho ou regras de aplicação.

Ele conhecerá apenas o estado do processo e os contratos necessários para coordená-lo.

### ADR-012 — Coreografia e orquestração coexistem

A arquitetura não utilizará o Orchestrator para todo evento.

```text
coreografia
→ eventos simples e independentes

orquestração
→ processos distribuídos com dependências entre etapas
```

### ADR-013 — Persistência própria do Orchestrator

O serviço possuirá:

```text
orchestrator_db
```

para armazenar o estado durável dos workflows.

### ADR-014 — Workflow pós-prova

O primeiro processo distribuído explicitamente orquestrado será o fluxo pós-prova:

```text
ApplicationFinished
        ↓
Scoring
        ↓
Performance
        ↓
Ranking
        ↓
AnswerKeyAccess
        ↓
ResultAvailable
        ↓
Communication
```

### ADR-015 — Rastreabilidade de workflows

Cada instância deverá possuir identificador próprio:

```text
workflowId
```

correlacionado com:

```text
correlationId
applicationId
candidateId
eventId
```

Isso permitirá rastrear todo o processo entre serviços.

---

## Princípio final

A arquitetura da Dia D deve preservar três níveis de responsabilidade:

```text
Gateway
→ protege e roteia

Orchestrator
→ coordena processos

Microservices de domínio
→ executam e decidem regras
```

O fluxo distribuído deve permanecer observável, recuperável e idempotente.

```text
Evento
   ↓
Workflow
   ↓
Serviço de domínio
   ↓
Resultado
   ↓
Próxima etapa
```

O objetivo não é centralizar a plataforma no Orchestrator.

O objetivo é utilizá-lo apenas onde a coordenação explícita torna o processo mais previsível, rastreável e resiliente.

---

**Dia D Simulation Platform — Architecture**

> Viva o dia antes dele chegar.
