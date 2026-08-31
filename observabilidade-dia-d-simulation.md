# Observabilidade --- Dia D Simulation

Documentação específica da estratégia de **observabilidade** da
plataforma **Dia D Simulation**, cobrindo logs, métricas, traces
distribuídos, OpenTelemetry, Prometheus, Grafana, alertas, health
checks, correlação entre requisições e eventos, SLI/SLO e operação dos
serviços.

------------------------------------------------------------------------

## Sumário

1.  [Visão geral](#1-visão-geral)
2.  [Objetivos da observabilidade](#2-objetivos-da-observabilidade)
3.  [Arquitetura de observabilidade](#3-arquitetura-de-observabilidade)
4.  [OpenTelemetry](#4-opentelemetry)
5.  [Logs estruturados](#5-logs-estruturados)
6.  [Correlation ID e
    rastreabilidade](#6-correlation-id-e-rastreabilidade)
7.  [Distributed Tracing](#7-distributed-tracing)
8.  [Métricas](#8-métricas)
9.  [Métricas por aplicação](#9-métricas-por-aplicação)
10. [Métricas REST/HTTP](#10-métricas-resthttp)
11. [Métricas Kafka](#11-métricas-kafka)
12. [Métricas RabbitMQ](#12-métricas-rabbitmq)
13. [Métricas de banco de dados](#13-métricas-de-banco-de-dados)
14. [Prometheus](#14-prometheus)
15. [Grafana](#15-grafana)
16. [Dashboards](#16-dashboards)
17. [Alertas](#17-alertas)
18. [Health Checks](#18-health-checks)
19. [SLI e SLO](#19-sli-e-slo)
20. [Observabilidade dos workflows](#20-observabilidade-dos-workflows)
21. [Observabilidade do dia da
    prova](#21-observabilidade-do-dia-da-prova)
22. [Tratamento de erros](#22-tratamento-de-erros)
23. [DLQ e retries](#23-dlq-e-retries)
24. [Auditoria](#24-auditoria)
25. [Retenção e cardinalidade](#25-retenção-e-cardinalidade)
26. [Endpoints operacionais](#26-endpoints-operacionais)
27. [Resumo da arquitetura](#27-resumo-da-arquitetura)

------------------------------------------------------------------------

# 1. Visão geral

A observabilidade do **Dia D Simulation** deve permitir entender o
estado da plataforma em tempo real, especialmente durante a janela
oficial de aplicação do simulado.

Os três pilares utilizados são:

``` text
Logs
Métricas
Traces
```

A instrumentação é centralizada através do **OpenTelemetry**, enquanto
**Prometheus** e **Grafana** são utilizados para coleta, consulta,
visualização e acompanhamento operacional das métricas.

A observabilidade deve cobrir tanto chamadas síncronas REST/HTTP quanto
os fluxos assíncronos executados através de Kafka e RabbitMQ.

------------------------------------------------------------------------

# 2. Objetivos da observabilidade

A solução deve permitir responder rapidamente perguntas como:

``` text
A plataforma está disponível?

Quantos candidatos estão realizando a prova agora?

Qual serviço está apresentando lentidão?

Qual endpoint está degradado?

Existem erros aumentando?

Kafka está acumulando consumer lag?

RabbitMQ está acumulando mensagens?

Existem mensagens em DLQ?

A correção de determinada prova terminou?

Onde determinado workflow falhou?

Quanto tempo a correção está demorando?

Quantos candidatos já finalizaram a prova?

O banco de dados está saturado?
```

O objetivo não é apenas detectar falhas, mas também identificar
**onde**, **quando** e **por que** elas ocorreram.

------------------------------------------------------------------------

# 3. Arquitetura de observabilidade

Fluxo conceitual:

``` text
Microservices
     |
     +---- Logs
     |
     +---- Metrics
     |
     +---- Traces
     |
     v
OpenTelemetry
     |
     +----------------+
     |                |
     v                v
Prometheus        Trace Backend
     |
     v
Grafana
     |
     v
Dashboards / Alerts
```

Todos os serviços devem produzir telemetria seguindo o mesmo padrão.

Aplicações monitoradas:

``` text
Auth Service
Candidate Service
Exam Service
Question Service
Application Service
Answer Service
Scoring Service
Performance Service
Ranking Service
Answer Key Service
Result Service
Communication Service
Orchestrator Service
Audit Service
```

------------------------------------------------------------------------

# 4. OpenTelemetry

O **OpenTelemetry** é o padrão de instrumentação da plataforma.

Responsabilidades:

-   geração de traces;
-   criação de spans;
-   propagação de contexto;
-   coleta de métricas;
-   associação entre logs e traces;
-   exportação de telemetria.

Cada aplicação deve possuir identificação própria.

Exemplo:

``` yaml
OTEL_SERVICE_NAME: application-service
OTEL_RESOURCE_ATTRIBUTES: deployment.environment=production
```

Atributos mínimos:

``` text
service.name
service.version
deployment.environment
service.instance.id
```

Exemplo de recurso:

``` json
{
  "service.name": "application-service",
  "service.version": "1.4.0",
  "deployment.environment": "production",
  "service.instance.id": "application-service-7f8bdc9d"
}
```

------------------------------------------------------------------------

# 5. Logs estruturados

Logs devem ser produzidos preferencialmente em **JSON estruturado**.

Exemplo:

``` json
{
  "timestamp": "2026-11-08T15:34:18.542Z",
  "level": "INFO",
  "service": "application-service",
  "environment": "production",
  "message": "Application started",
  "candidateId": "candidate-uuid",
  "applicationId": "application-uuid",
  "examId": "exam-uuid",
  "correlationId": "3f2f8468-5d16-4af8-b65e-f624a17cc973",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7"
}
```

Campos mínimos recomendados:

``` text
timestamp
level
service
environment
message
correlationId
traceId
spanId
```

Campos de domínio podem ser adicionados quando necessários:

``` text
candidateId
applicationId
examId
questionId
scoreId
workflowId
eventId
commandId
```

## Níveis

``` text
TRACE -> diagnóstico extremamente detalhado
DEBUG -> diagnóstico de desenvolvimento
INFO  -> acontecimentos normais do negócio
WARN  -> situação inesperada recuperável
ERROR -> operação não concluída
```

Não devem ser registrados:

-   senhas;
-   tokens JWT completos;
-   secrets;
-   credenciais;
-   dados sensíveis desnecessários;
-   payloads integrais quando contiverem informações protegidas.

------------------------------------------------------------------------

# 6. Correlation ID e rastreabilidade

Toda operação iniciada na plataforma deve possuir:

``` text
X-Correlation-Id
```

Exemplo:

``` http
X-Correlation-Id: 3f2f8468-5d16-4af8-b65e-f624a17cc973
```

Se o cliente não enviar o identificador, a primeira aplicação deve
gerá-lo.

O mesmo valor deve ser propagado por:

``` text
REST / HTTP
Kafka
RabbitMQ
Orchestrator
Logs
```

Exemplo:

``` text
Frontend
   |
   | correlationId = ABC-123
   v
Application Service
   |
   | ABC-123
   v
Kafka
   |
   | ABC-123
   v
Orchestrator
   |
   | ABC-123
   v
RabbitMQ
   |
   | ABC-123
   v
Scoring Service
```

Assim, uma única execução pode ser localizada em toda a arquitetura.

------------------------------------------------------------------------

# 7. Distributed Tracing

Cada requisição ou processamento assíncrono deve gerar um trace.

Exemplo de chamada síncrona:

``` text
HTTP POST /applications/{id}/start
                |
                v
        Application Service
                |
                +---- database query
                |
                +---- validation
                |
                +---- response
```

Exemplo de trace:

``` text
Trace
|
+-- HTTP POST /applications/{id}/start
    |
    +-- validateApplication
    |
    +-- validateExamWindow
    |
    +-- SELECT application
    |
    +-- UPDATE application
```

No processamento assíncrono:

``` text
ApplicationFinished
        |
        v
Orchestrator
        |
        v
ProcessScoring
        |
        v
Scoring Service
```

O contexto de tracing deve ser propagado através dos headers das
mensagens.

Atributos úteis:

``` text
messaging.system
messaging.destination.name
messaging.operation
event.type
event.id
command.id
application.id
exam.id
```

------------------------------------------------------------------------

# 8. Métricas

As métricas devem ser classificadas em quatro grupos:

``` text
Infraestrutura
Aplicação
Mensageria
Negócio
```

## Infraestrutura

``` text
CPU
memória
disco
network
pods
restarts
threads
JVM
GC
```

## Aplicação

``` text
requisições
latência
erros
timeouts
conexões
pool de banco
```

## Mensageria

``` text
consumer lag
mensagens publicadas
mensagens consumidas
retries
DLQ
tempo de processamento
```

## Negócio

``` text
candidatos inscritos
candidatos online
provas iniciadas
provas finalizadas
respostas registradas
correções concluídas
resultados disponíveis
```

------------------------------------------------------------------------

# 9. Métricas por aplicação

## Auth Service

``` text
auth_login_total
auth_login_success_total
auth_login_failure_total
auth_token_refresh_total
```

## Candidate Service

``` text
candidate_created_total
candidate_active_total
candidate_update_total
```

## Exam Service

``` text
exam_active_total
exam_request_total
```

## Question Service

``` text
question_request_total
question_request_duration_seconds
```

## Application Service

``` text
application_registered_total
application_started_total
application_finished_total
application_in_progress
application_start_failure_total
application_finish_failure_total
```

## Answer Service

``` text
answer_submitted_total
answer_updated_total
answer_deleted_total
answer_submission_failure_total
```

## Scoring Service

``` text
scoring_started_total
scoring_finished_total
scoring_failed_total
scoring_duration_seconds
scoring_in_progress
```

## Performance Service

``` text
performance_calculated_total
performance_failed_total
performance_calculation_duration_seconds
```

## Ranking Service

``` text
ranking_generated_total
ranking_failed_total
ranking_generation_duration_seconds
```

## Answer Key Service

``` text
answer_key_released_total
answer_key_request_total
```

## Result Service

``` text
result_available_total
result_request_total
result_processing_total
```

## Communication Service

``` text
communication_sent_total
communication_failed_total
communication_retry_total
```

## Orchestrator Service

``` text
workflow_started_total
workflow_finished_total
workflow_failed_total
workflow_in_progress
workflow_duration_seconds
```

------------------------------------------------------------------------

# 10. Métricas REST/HTTP

Para todos os serviços REST:

``` text
http_server_requests_total
http_server_request_duration_seconds
http_server_errors_total
```

Dimensões:

``` text
method
uri
status
service
```

Exemplo:

``` text
http_server_requests_total{
  service="application-service",
  method="POST",
  uri="/api/v1/applications/{applicationId}/start",
  status="200"
}
```

Nunca utilizar IDs dinâmicos diretamente como label:

``` text
ERRADO:

uri="/applications/75f8c928"
candidateId="9af8..."
```

Isso causa alta cardinalidade.

------------------------------------------------------------------------

# 11. Métricas Kafka

Métricas essenciais:

``` text
kafka_messages_produced_total
kafka_messages_consumed_total
kafka_consumer_lag
kafka_consumer_errors_total
kafka_processing_duration_seconds
```

Consumer lag é uma das métricas mais importantes.

Exemplo:

``` text
kafka_consumer_lag{
  topic="scoring.events",
  consumer_group="performance-service"
}
```

Alerta deve ser gerado quando o lag permanecer elevado por período
significativo.

------------------------------------------------------------------------

# 12. Métricas RabbitMQ

Métricas:

``` text
rabbitmq_queue_messages
rabbitmq_queue_messages_ready
rabbitmq_queue_messages_unacked
rabbitmq_consumers
rabbitmq_messages_published_total
rabbitmq_messages_delivered_total
rabbitmq_messages_retried_total
rabbitmq_dlq_messages
```

Filas críticas:

``` text
scoring.process.queue
performance.calculate.queue
ranking.generate.queue
answer-key.release.queue
communication.send.queue
```

DLQs:

``` text
scoring.process.dlq
performance.calculate.dlq
ranking.generate.dlq
answer-key.release.dlq
communication.send.dlq
```

------------------------------------------------------------------------

# 13. Métricas de banco de dados

Devem ser acompanhadas:

``` text
conexões ativas
conexões disponíveis
tempo de aquisição
queries lentas
timeouts
erros
pool esgotado
```

Para aplicações Spring com HikariCP:

``` text
hikaricp_connections_active
hikaricp_connections_idle
hikaricp_connections_pending
hikaricp_connections_timeout_total
hikaricp_connections_max
```

Uma quantidade crescente de threads aguardando conexão pode indicar
saturação do banco ou dimensionamento inadequado do pool.

------------------------------------------------------------------------

# 14. Prometheus

O Prometheus coleta métricas expostas pelas aplicações.

Endpoint:

``` http
GET /actuator/prometheus
```

Exemplo:

``` text
http_server_requests_total 258491

application_in_progress 18432

answer_submitted_total 2384521

scoring_finished_total 17532
```

Configuração conceitual:

``` yaml
management:
  endpoints:
    web:
      exposure:
        include:
          - health
          - info
          - prometheus
```

Endpoints administrativos não devem ser expostos publicamente sem
controle de acesso.

------------------------------------------------------------------------

# 15. Grafana

O Grafana é utilizado para:

-   dashboards;
-   visualização de métricas;
-   acompanhamento operacional;
-   análise de tendências;
-   investigação de incidentes;
-   alertas.

Organização sugerida:

``` text
Dia D Simulation
|
+-- Platform Overview
+-- Exam Day
+-- REST APIs
+-- Kafka
+-- RabbitMQ
+-- Database
+-- JVM
+-- Scoring
+-- Workflows
+-- Business Metrics
```

------------------------------------------------------------------------

# 16. Dashboards

## Platform Overview

Deve mostrar:

``` text
Disponibilidade
Requests por segundo
Latência P50
Latência P95
Latência P99
Taxa de erro
CPU
Memória
Pods ativos
Pods reiniciados
```

## Exam Day

Dashboard específico para o dia da prova:

``` text
Candidatos inscritos
Candidatos autenticados
Candidatos online
Aplicações iniciadas
Aplicações em andamento
Aplicações finalizadas
Respostas por segundo
Falhas ao salvar resposta
Tempo médio para salvar resposta
```

## Kafka

``` text
Mensagens por segundo
Consumer lag
Erros de consumo
Tempo de processamento
Consumers ativos
```

## RabbitMQ

``` text
Mensagens prontas
Mensagens unacked
Consumers
Retries
DLQ
Throughput
```

## Scoring

``` text
Correções aguardando
Correções em andamento
Correções concluídas
Correções com erro
Tempo médio de correção
P95 de correção
```

## Workflows

``` text
Workflows iniciados
Workflows concluídos
Workflows em andamento
Workflows falhos
Etapa atual
Tempo médio total
```

------------------------------------------------------------------------

# 17. Alertas

Alertas devem indicar condições que exigem análise ou intervenção.

## Disponibilidade

``` text
Serviço indisponível
Health check falhando
Quantidade insuficiente de instâncias
```

## HTTP

``` text
Taxa de erro 5xx elevada
P95 acima do limite
P99 acima do limite
Timeouts elevados
```

## Kafka

``` text
Consumer lag elevado
Consumer parado
Falhas de publicação
Falhas de consumo
```

## RabbitMQ

``` text
Fila crescendo continuamente
Ausência de consumers
Mensagens em DLQ
Mensagens unacked elevadas
```

## Banco

``` text
Pool próximo do limite
Conexões pendentes
Timeout de conexão
Queries lentas
```

## Negócio

``` text
Falhas ao iniciar prova
Falhas ao registrar respostas
Queda abrupta na taxa de respostas
Correções paradas
Resultados não publicados
```

Um alerta deve possuir:

``` text
nome
severidade
serviço
condição
duração
descrição
runbook
```

------------------------------------------------------------------------

# 18. Health Checks

Aplicações Spring Boot devem utilizar Actuator.

## Liveness

``` http
GET /actuator/health/liveness
```

Indica se o processo está operacional.

Exemplo:

``` json
{
  "status": "UP"
}
```

## Readiness

``` http
GET /actuator/health/readiness
```

Indica se a aplicação está pronta para receber tráfego.

Exemplo:

``` json
{
  "status": "UP"
}
```

Readiness pode considerar dependências essenciais.

Exemplo:

``` json
{
  "status": "DOWN",
  "components": {
    "db": {
      "status": "DOWN"
    }
  }
}
```

------------------------------------------------------------------------

# 19. SLI e SLO

## SLI

Indicadores utilizados para medir o serviço.

Exemplos:

``` text
Disponibilidade
Taxa de sucesso
Latência
Tempo de processamento
```

## SLO

Objetivos definidos para os SLIs.

Exemplo inicial:

  Indicador                                             Objetivo
  ----------------------------------- --------------------------
  Disponibilidade das APIs críticas                    \>= 99,9%
  Requests HTTP sem erro 5xx                           \>= 99,9%
  Registro de resposta P95                            \<= 500 ms
  Consulta de sessão P95                              \<= 500 ms
  Início da aplicação P95                                \<= 1 s
  Processamento assíncrono              acompanhado por workflow

Os valores devem ser calibrados posteriormente através de testes de
carga e comportamento real da plataforma.

------------------------------------------------------------------------

# 20. Observabilidade dos workflows

Cada workflow do Orchestrator deve possuir:

``` text
workflowId
correlationId
workflowType
status
startedAt
finishedAt
currentStep
```

Estados:

``` text
PENDING
IN_PROGRESS
FINISHED
FAILED
```

Exemplo:

``` json
{
  "workflowId": "workflow-uuid",
  "correlationId": "ABC-123",
  "workflowType": "APPLICATION_POST_PROCESSING",
  "status": "IN_PROGRESS",
  "currentStep": "CALCULATE_PERFORMANCE",
  "startedAt": "2026-11-08T20:12:40Z"
}
```

Métricas:

``` text
workflow_started_total
workflow_finished_total
workflow_failed_total
workflow_in_progress
workflow_duration_seconds
```

------------------------------------------------------------------------

# 21. Observabilidade do dia da prova

O período de aplicação é a janela mais crítica da plataforma.

A operação deve possuir uma visão específica em tempo real.

## Indicadores principais

``` text
Candidatos autenticados
Candidatos online
Provas iniciadas
Provas em andamento
Provas finalizadas
Respostas recebidas por segundo
Falhas de resposta
Latência do Answer Service
Latência do Application Service
CPU e memória
Conexões de banco
Kafka lag
RabbitMQ queues
```

Exemplo conceitual:

``` text
DIA D — LIVE

Candidatos inscritos............. 50.000
Online........................... 42.812
Em prova......................... 41.903
Finalizaram...................... 2.145

Respostas/s...................... 8.420
Falhas de resposta............... 0,03%

Application API P95.............. 180 ms
Answer API P95................... 210 ms

Kafka Lag........................ 24
RabbitMQ DLQ..................... 0
```

Esse dashboard deve ser a principal visão da equipe durante a simulação.

------------------------------------------------------------------------

# 22. Tratamento de erros

Erros devem possuir logs estruturados.

Exemplo:

``` json
{
  "timestamp": "2026-11-08T15:40:12Z",
  "level": "ERROR",
  "service": "answer-service",
  "message": "Failed to persist answer",
  "applicationId": "application-uuid",
  "questionId": "question-uuid",
  "correlationId": "ABC-123",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "exception": "AnswerPersistenceException"
}
```

Stack traces devem ser mantidos no backend de logs, sem exposição ao
cliente HTTP.

------------------------------------------------------------------------

# 23. DLQ e retries

Toda ocorrência de DLQ deve ser observável.

Métricas:

``` text
message_retry_total
message_dlq_total
```

Logs devem registrar:

``` text
eventId
commandId
queue
retryCount
error
correlationId
```

Exemplo:

``` json
{
  "level": "ERROR",
  "service": "scoring-service",
  "message": "Message moved to DLQ",
  "queue": "scoring.process.queue",
  "dlq": "scoring.process.dlq",
  "commandId": "command-uuid",
  "retryCount": 3,
  "correlationId": "ABC-123"
}
```

A presença de mensagem em DLQ deve gerar alerta.

------------------------------------------------------------------------

# 24. Auditoria

Observabilidade e auditoria possuem objetivos diferentes.

``` text
Observabilidade
    -> saúde e comportamento técnico

Auditoria
    -> histórico de ações e eventos relevantes
```

Exemplos de auditoria:

``` text
Candidato iniciou prova
Candidato finalizou prova
Gabarito foi publicado
Resultado foi disponibilizado
Administrador alterou exame
```

Logs técnicos não devem substituir registros formais de auditoria.

------------------------------------------------------------------------

# 25. Retenção e cardinalidade

Labels de métricas devem possuir cardinalidade controlada.

Permitido:

``` text
service
method
status
uri_template
topic
queue
consumer_group
environment
```

Evitar:

``` text
candidateId
applicationId
questionId
eventId
commandId
correlationId
```

Esses identificadores devem permanecer em **logs e traces**, e não como
labels de métricas.

Isso evita crescimento explosivo das séries temporais no Prometheus.

------------------------------------------------------------------------

# 26. Endpoints operacionais

Endpoints recomendados:

``` text
GET /actuator/health
GET /actuator/health/liveness
GET /actuator/health/readiness
GET /actuator/info
GET /actuator/prometheus
```

Exemplo:

``` http
GET /actuator/health
```

Response:

``` json
{
  "status": "UP"
}
```

Esses endpoints devem possuir política de exposição apropriada ao
ambiente.

------------------------------------------------------------------------

# 27. Resumo da arquitetura

A estratégia de observabilidade do **Dia D Simulation** utiliza:

``` text
OpenTelemetry
      |
      +---- Traces
      |
      +---- Metrics
      |
      +---- Context Propagation

Prometheus
      |
      +---- Métricas

Grafana
      |
      +---- Dashboards
      |
      +---- Alertas
```

Princípios principais:

-   logs estruturados;
-   métricas técnicas e de negócio;
-   tracing distribuído;
-   `Correlation ID` ponta a ponta;
-   observabilidade de REST, Kafka e RabbitMQ;
-   monitoramento específico para o dia da prova;
-   acompanhamento de retries e DLQs;
-   health checks;
-   SLI/SLO;
-   baixa cardinalidade nas métricas;
-   dashboards operacionais;
-   alertas acionáveis.

A observabilidade deve permitir acompanhar uma operação desde a
requisição inicial até o término de seus processamentos assíncronos,
mantendo visibilidade sobre toda a arquitetura distribuída do **Dia D
Simulation**.
