# Eventos Assíncronos --- Dia D Simulation

Documentação da comunicação assíncrona do **Dia D Simulation**, com foco
em **Kafka**, **RabbitMQ**, eventos de domínio, comandos, tópicos,
filas, retries, DLQ, idempotência, versionamento e rastreabilidade.

------------------------------------------------------------------------

## Sumário

1.  [Visão geral](#1-visão-geral)
2.  [Responsabilidade do Kafka e
    RabbitMQ](#2-responsabilidade-do-kafka-e-rabbitmq)
3.  [Fluxo assíncrono principal](#3-fluxo-assíncrono-principal)
4.  [Padrão dos eventos](#4-padrão-dos-eventos)
5.  [Eventos Kafka](#5-eventos-kafka)
6.  [Application Events](#6-application-events)
7.  [Scoring Events](#7-scoring-events)
8.  [Performance Events](#8-performance-events)
9.  [Ranking Events](#9-ranking-events)
10. [Answer Key Events](#10-answer-key-events)
11. [Communication Events](#11-communication-events)
12. [RabbitMQ](#12-rabbitmq)
13. [Filas RabbitMQ](#13-filas-rabbitmq)
14. [Orchestrator](#14-orchestrator)
15. [Retry e DLQ](#15-retry-e-dlq)
16. [Idempotência](#16-idempotência)
17. [Correlation ID](#17-correlation-id)
18. [Versionamento dos contratos](#18-versionamento-dos-contratos)
19. [Nomenclatura](#19-nomenclatura)
20. [Diagrama Kafka](#20-diagrama-kafka)
21. [Diagrama RabbitMQ](#21-diagrama-rabbitmq)
22. [Diagrama geral](#22-diagrama-geral)
23. [Resumo da arquitetura
    assíncrona](#23-resumo-da-arquitetura-assíncrona)

------------------------------------------------------------------------

# 1. Visão geral

A comunicação assíncrona da plataforma utiliza:

-   **Kafka** para eventos de domínio;
-   **RabbitMQ** para comandos e processamento direcionado;
-   **Orchestrator** para controlar workflows distribuídos.

```{=html}
![Diagrama](eventos_async_corrigido/01_visao.png)
```

------------------------------------------------------------------------

# 2. Responsabilidade do Kafka e RabbitMQ

## Kafka

O Kafka transporta **eventos de domínio**, ou seja, fatos que já
ocorreram.

Exemplos:

``` text
ApplicationStarted
ApplicationFinished
ScoringFinished
PerformanceCalculated
RankingUpdated
AnswerKeyReleased
ResultAvailable
```

Um mesmo evento pode possuir múltiplos consumidores.

## RabbitMQ

O RabbitMQ recebe **comandos assíncronos**, representando ações que
devem ser executadas por um serviço específico.

Exemplos:

``` text
ProcessScoring
CalculatePerformance
GenerateRanking
ReleaseAnswerKey
SendCommunication
```

```{=html}
![Diagrama](eventos_async_corrigido/02_kafka_rabbit.png)
```

------------------------------------------------------------------------

# 3. Fluxo assíncrono principal

O fluxo de pós-prova é coordenado pelo Orchestrator.

```{=html}
![Diagrama](eventos_async_corrigido/03_fluxo_principal.png)
```

------------------------------------------------------------------------

# 4. Padrão dos eventos

Todos os eventos seguem um envelope comum.

``` json
{
  "eventId": "uuid",
  "eventType": "ApplicationFinished",
  "eventVersion": 1,
  "occurredAt": "2026-08-30T17:00:00Z",
  "correlationId": "uuid",
  "aggregateId": "uuid",
  "payload": {}
}
```

  Campo             Descrição
  ----------------- ----------------------------------
  `eventId`         Identificador único do evento
  `eventType`       Tipo do evento
  `eventVersion`    Versão do contrato
  `occurredAt`      Data e hora do evento
  `correlationId`   Identificador de rastreabilidade
  `aggregateId`     Entidade principal relacionada
  `payload`         Dados específicos do evento

------------------------------------------------------------------------

# 5. Eventos Kafka

  Tópico                   Responsabilidade
  ------------------------ ------------------------------
  `application.events`     Eventos da execução da prova
  `scoring.events`         Eventos de correção
  `performance.events`     Eventos de desempenho
  `ranking.events`         Eventos de classificação
  `answer-key.events`      Eventos de gabarito
  `result.events`          Eventos de resultado
  `communication.events`   Eventos de comunicação

------------------------------------------------------------------------

# 6. Application Events

## `ApplicationStarted`

Publicado quando o candidato inicia a prova.

``` json
{"applicationId":"uuid","candidateId":"uuid","examId":"uuid","startedAt":"timestamp"}
```

## `AnswerSubmitted`

Publicado quando uma resposta é registrada.

``` json
{"applicationId":"uuid","questionId":"uuid","alternativeId":"uuid","answeredAt":"timestamp"}
```

## `ApplicationFinished`

Publicado quando a prova é finalizada.

``` json
{"applicationId":"uuid","candidateId":"uuid","examId":"uuid","finishedAt":"timestamp"}
```

Consumidores principais:

-   Orchestrator Service
-   Audit Service

------------------------------------------------------------------------

# 7. Scoring Events

## `ScoringStarted`

``` json
{"applicationId":"uuid","scoringId":"uuid","startedAt":"timestamp"}
```

## `ScoringFinished`

``` json
{"applicationId":"uuid","candidateId":"uuid","examId":"uuid","scoreId":"uuid","finalScore":782.45,"finishedAt":"timestamp"}
```

Consumidores principais:

-   Orchestrator Service
-   Performance Service
-   Audit Service

## `ScoringFailed`

``` json
{"applicationId":"uuid","scoringId":"uuid","reason":"string","failedAt":"timestamp"}
```

------------------------------------------------------------------------

# 8. Performance Events

## `PerformanceCalculated`

``` json
{"candidateId":"uuid","applicationId":"uuid","scoreId":"uuid","performanceId":"uuid","overallPercentage":81.50,"percentile":92.30,"calculatedAt":"timestamp"}
```

Consumidores:

-   Orchestrator Service
-   Ranking Service
-   Audit Service

------------------------------------------------------------------------

# 9. Ranking Events

## `RankingUpdated`

``` json
{"candidateId":"uuid","examEventId":"uuid","rankingId":"uuid","position":152,"score":782.45,"updatedAt":"timestamp"}
```

Consumidores:

-   Orchestrator Service
-   Audit Service

------------------------------------------------------------------------

# 10. Answer Key Events

## `AnswerKeyReleased`

``` json
{"examId":"uuid","publicationId":"uuid","version":1,"releasedAt":"timestamp"}
```

Consumidores:

-   Orchestrator Service
-   Communication Service
-   Audit Service

------------------------------------------------------------------------

# 11. Communication Events

## `ResultAvailable`

Publicado quando o resultado final está disponível para consulta.

``` json
{"candidateId":"uuid","applicationId":"uuid","scoreId":"uuid","performanceId":"uuid","availableAt":"timestamp"}
```

Consumidores:

-   Communication Service
-   Audit Service

## `CommunicationSent`

``` json
{"communicationId":"uuid","candidateId":"uuid","channel":"EMAIL","sentAt":"timestamp"}
```

## `CommunicationFailed`

``` json
{"communicationId":"uuid","candidateId":"uuid","channel":"EMAIL","reason":"string","failedAt":"timestamp"}
```

------------------------------------------------------------------------

# 12. RabbitMQ

O RabbitMQ executa o processamento direcionado dos comandos.

```{=html}
![Diagrama](eventos_async_corrigido/04_rabbit.png)
```
Exchanges principais:

``` text
dia-d.commands
dia-d.retry
dia-d.dlx
```

------------------------------------------------------------------------

# 13. Filas RabbitMQ

  Fila                            Consumidor
  ------------------------------- -----------------------
  `scoring.process.queue`         Scoring Service
  `performance.calculate.queue`   Performance Service
  `ranking.generate.queue`        Ranking Service
  `answer-key.release.queue`      Answer Key Service
  `communication.send.queue`      Communication Service

## `scoring.process.queue`

``` json
{"commandId":"uuid","applicationId":"uuid","candidateId":"uuid","examId":"uuid","correlationId":"uuid"}
```

## `performance.calculate.queue`

``` json
{"commandId":"uuid","candidateId":"uuid","applicationId":"uuid","scoreId":"uuid","correlationId":"uuid"}
```

## `ranking.generate.queue`

``` json
{"commandId":"uuid","candidateId":"uuid","examEventId":"uuid","performanceId":"uuid","correlationId":"uuid"}
```

## `answer-key.release.queue`

``` json
{"commandId":"uuid","examId":"uuid","correlationId":"uuid"}
```

## `communication.send.queue`

``` json
{"commandId":"uuid","candidateId":"uuid","communicationType":"RESULT_AVAILABLE","channel":"EMAIL","correlationId":"uuid"}
```

------------------------------------------------------------------------

# 14. Orchestrator

O Orchestrator coordena a sequência do workflow assíncrono.

```{=html}
![Diagrama](eventos_async_corrigido/05_orchestrator.png)
```
Exemplo aplicado:

```{=html}
![Diagrama](eventos_async_corrigido/06_retry_dlq.png)
```

------------------------------------------------------------------------

# 15. Retry e DLQ

Falhas temporárias passam por retry antes do envio para DLQ.

```{=html}
![Diagrama](eventos_async_corrigido/07_idempotencia.png)
```
DLQs previstas:

``` text
scoring.process.dlq
performance.calculate.dlq
ranking.generate.dlq
answer-key.release.dlq
communication.send.dlq
```

Política inicial:

``` text
Tentativa 1 -> 5 segundos
Tentativa 2 -> 30 segundos
Tentativa 3 -> 2 minutos
Tentativa 4 -> DLQ
```

------------------------------------------------------------------------

# 16. Idempotência

Eventos e comandos podem ser entregues mais de uma vez.

O consumidor deve validar:

``` text
eventId
commandId
```

antes de executar novamente.

```{=html}
![Diagrama](eventos_async_corrigido/08_correlation.png)
```

------------------------------------------------------------------------

# 17. Correlation ID

Todas as mensagens devem transportar:

``` text
correlationId
```

O mesmo identificador acompanha todo o workflow.

```{=html}
![Diagrama](eventos_async_corrigido/09_geral.png)
```

------------------------------------------------------------------------

# 18. Versionamento dos contratos

Eventos devem possuir versão explícita:

``` json
{"eventType":"ScoringFinished","eventVersion":1}
```

Mudanças incompatíveis devem gerar uma nova versão do contrato, evitando
quebra de consumidores existentes.

------------------------------------------------------------------------

# 19. Nomenclatura

Eventos representam fatos e devem usar passado:

``` text
ApplicationFinished
ScoringFinished
PerformanceCalculated
RankingUpdated
AnswerKeyReleased
CommunicationSent
```

Comandos representam intenção e devem usar verbo de ação:

``` text
ProcessScoring
CalculatePerformance
GenerateRanking
ReleaseAnswerKey
SendCommunication
```

------------------------------------------------------------------------

# 20. Diagrama Kafka

```{=html}
![Diagrama](eventos_async_corrigido/09_geral.png)
```

------------------------------------------------------------------------

# 21. Diagrama RabbitMQ

```{=html}
![Diagrama](eventos_async_corrigido/09_geral.png)
```

------------------------------------------------------------------------

# 22. Diagrama geral

```{=html}
![Diagrama](eventos_async_corrigido/09_geral.png)
```

------------------------------------------------------------------------

# 23. Resumo da arquitetura assíncrona

A separação principal é:

``` text
Evento = fato ocorrido
Comando = ação a executar
```

No Dia D Simulation:

-   **Kafka** propaga eventos de domínio;
-   **RabbitMQ** direciona comandos para processamento;
-   **Orchestrator** controla a sequência dos workflows;
-   **Retry e DLQ** tratam falhas;
-   **Idempotência** evita processamento duplicado;
-   **Correlation ID** garante rastreabilidade entre serviços.
