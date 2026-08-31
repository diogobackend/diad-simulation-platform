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
<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 330">
```
`<rect width="1200" height="330" rx="22" fill="#061426"/>`{=html}
`<defs>`{=html}`<marker id="a1" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">`{=html}`<path d="M0,0 L0,6 L9,3z" fill="#8B5CF6"/>`{=html}`</marker>`{=html}`</defs>`{=html}
`<path d="M360 165 H470" stroke="#8B5CF6" stroke-width="3" marker-end="url(#a1)"/>`{=html}
`<path d="M730 165 H840" stroke="#8B5CF6" stroke-width="3" marker-end="url(#a1)"/>`{=html}
`<g fill="#0A2342" stroke="#168BFF" stroke-width="3">`{=html}
`<rect x="80" y="115" width="280" height="100" rx="18"/>`{=html}`<rect x="470" y="115" width="260" height="100" rx="18"/>`{=html}`<rect x="840" y="115" width="280" height="100" rx="18"/>`{=html}
`</g>`{=html}
`<g fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="24" font-weight="700" text-anchor="middle">`{=html}
`<text x="220" y="175">`{=html}Application
Service`</text>`{=html}`<text x="600" y="175">`{=html}Orchestrator`</text>`{=html}`<text x="980" y="175">`{=html}Domínio
assíncrono`</text>`{=html} `</g>`{=html}
```{=html}
</svg>
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
<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 380">
```
`<rect width="1200" height="380" rx="22" fill="#061426"/>`{=html}
`<defs>`{=html}`<marker id="a2" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">`{=html}`<path d="M0,0 L0,6 L9,3z" fill="#8B5CF6"/>`{=html}`</marker>`{=html}`</defs>`{=html}
`<g fill="#0A2342" stroke="#168BFF" stroke-width="3">`{=html}
`<rect x="90" y="70" width="300" height="100" rx="18"/>`{=html}`<rect x="810" y="70" width="300" height="100" rx="18"/>`{=html}
`<rect x="90" y="220" width="300" height="100" rx="18"/>`{=html}`<rect x="810" y="220" width="300" height="100" rx="18"/>`{=html}
`</g>`{=html}
`<path d="M390 120 H810" stroke="#8B5CF6" stroke-width="3" marker-end="url(#a2)"/>`{=html}`<path d="M390 270 H810" stroke="#8B5CF6" stroke-width="3" marker-end="url(#a2)"/>`{=html}
`<g fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="24" font-weight="700" text-anchor="middle">`{=html}
`<text x="240" y="130">`{=html}Kafka`</text>`{=html}`<text x="960" y="130">`{=html}Evento
de
domínio`</text>`{=html}`<text x="240" y="280">`{=html}RabbitMQ`</text>`{=html}`<text x="960" y="280">`{=html}Comando
assíncrono`</text>`{=html} `</g>`{=html}
```{=html}
</svg>
```

------------------------------------------------------------------------

# 3. Fluxo assíncrono principal

O fluxo de pós-prova é coordenado pelo Orchestrator.

```{=html}
<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 1060">
```
`<rect width="1200" height="1060" rx="22" fill="#061426"/>`{=html}
`<defs>`{=html}`<marker id="a3" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">`{=html}`<path d="M0,0 L0,6 L9,3z" fill="#8B5CF6"/>`{=html}`</marker>`{=html}`</defs>`{=html}
`<g fill="#0A2342" stroke="#168BFF" stroke-width="3">`{=html}
`<rect x="390" y="45" width="420" height="70" rx="17"/>`{=html}`<rect x="390" y="140" width="420" height="70" rx="17"/>`{=html}`<rect x="390" y="235" width="420" height="70" rx="17"/>`{=html}
`<rect x="390" y="330" width="420" height="70" rx="17"/>`{=html}`<rect x="390" y="425" width="420" height="70" rx="17"/>`{=html}`<rect x="390" y="520" width="420" height="70" rx="17"/>`{=html}
`<rect x="390" y="615" width="420" height="70" rx="17"/>`{=html}`<rect x="390" y="710" width="420" height="70" rx="17"/>`{=html}`<rect x="390" y="805" width="420" height="70" rx="17"/>`{=html}
`<rect x="390" y="900" width="420" height="70" rx="17"/>`{=html}
`</g>`{=html}
`<g stroke="#8B5CF6" stroke-width="3" marker-end="url(#a3)">`{=html}
`<path d="M600 115 V140"/>`{=html}`<path d="M600 210 V235"/>`{=html}`<path d="M600 305 V330"/>`{=html}`<path d="M600 400 V425"/>`{=html}`<path d="M600 495 V520"/>`{=html}
`<path d="M600 590 V615"/>`{=html}`<path d="M600 685 V710"/>`{=html}`<path d="M600 780 V805"/>`{=html}`<path d="M600 875 V900"/>`{=html}
`</g>`{=html}
`<g fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="22" font-weight="700" text-anchor="middle">`{=html}
`<text x="600" y="89">`{=html}ApplicationFinished`</text>`{=html}`<text x="600" y="184">`{=html}ProcessScoring`</text>`{=html}`<text x="600" y="279">`{=html}ScoringFinished`</text>`{=html}
`<text x="600" y="374">`{=html}CalculatePerformance`</text>`{=html}`<text x="600" y="469">`{=html}PerformanceCalculated`</text>`{=html}`<text x="600" y="564">`{=html}GenerateRanking`</text>`{=html}
`<text x="600" y="659">`{=html}RankingUpdated`</text>`{=html}`<text x="600" y="754">`{=html}ReleaseAnswerKey`</text>`{=html}`<text x="600" y="849">`{=html}AnswerKeyReleased
/ ResultAvailable`</text>`{=html}
`<text x="600" y="944">`{=html}SendCommunication`</text>`{=html}
`</g>`{=html}
```{=html}
</svg>
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
<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 330">
```
`<rect width="1200" height="330" rx="22" fill="#061426"/>`{=html}
`<defs>`{=html}`<marker id="a4" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">`{=html}`<path d="M0,0 L0,6 L9,3z" fill="#8B5CF6"/>`{=html}`</marker>`{=html}`</defs>`{=html}
`<g fill="#0A2342" stroke="#168BFF" stroke-width="3">`{=html}`<rect x="65" y="115" width="260" height="100" rx="18"/>`{=html}`<rect x="470" y="115" width="260" height="100" rx="18"/>`{=html}`<rect x="875" y="115" width="260" height="100" rx="18"/>`{=html}`</g>`{=html}
`<path d="M325 165 H470" stroke="#8B5CF6" stroke-width="3" marker-end="url(#a4)"/>`{=html}`<path d="M730 165 H875" stroke="#8B5CF6" stroke-width="3" marker-end="url(#a4)"/>`{=html}
`<g fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="24" font-weight="700" text-anchor="middle">`{=html}`<text x="195" y="175">`{=html}Producer`</text>`{=html}`<text x="600" y="175">`{=html}Exchange`</text>`{=html}`<text x="1005" y="175">`{=html}Queue
/ Consumer`</text>`{=html}`</g>`{=html}
```{=html}
</svg>
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
<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 850">
```
`<rect width="1200" height="850" rx="22" fill="#061426"/>`{=html}
`<defs>`{=html}`<marker id="a5" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">`{=html}`<path d="M0,0 L0,6 L9,3z" fill="#8B5CF6"/>`{=html}`</marker>`{=html}`</defs>`{=html}
`<g fill="#0A2342" stroke="#168BFF" stroke-width="3">`{=html}
`<rect x="410" y="45" width="380" height="78" rx="18"/>`{=html}`<rect x="410" y="175" width="380" height="78" rx="18"/>`{=html}`<rect x="410" y="305" width="380" height="78" rx="18"/>`{=html}
`<rect x="410" y="435" width="380" height="78" rx="18"/>`{=html}`<rect x="410" y="565" width="380" height="78" rx="18"/>`{=html}`<rect x="410" y="695" width="380" height="78" rx="18"/>`{=html}
`</g>`{=html}
`<g stroke="#8B5CF6" stroke-width="3" marker-end="url(#a5)">`{=html}`<path d="M600 123 V175"/>`{=html}`<path d="M600 253 V305"/>`{=html}`<path d="M600 383 V435"/>`{=html}`<path d="M600 513 V565"/>`{=html}`<path d="M600 643 V695"/>`{=html}`</g>`{=html}
`<g fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="23" font-weight="700" text-anchor="middle">`{=html}
`<text x="600" y="94">`{=html}Evento
Kafka`</text>`{=html}`<text x="600" y="224">`{=html}Orchestrator /
Workflow`</text>`{=html}`<text x="600" y="354">`{=html}Comando
RabbitMQ`</text>`{=html} `<text x="600" y="484">`{=html}Serviço de
domínio`</text>`{=html}`<text x="600" y="614">`{=html}Novo evento
Kafka`</text>`{=html}`<text x="600" y="744">`{=html}Próxima
etapa`</text>`{=html} `</g>`{=html}
```{=html}
</svg>
```
Exemplo aplicado:

```{=html}
<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 980">
```
`<rect width="1200" height="980" rx="22" fill="#061426"/>`{=html}
`<defs>`{=html}`<marker id="a6" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">`{=html}`<path d="M0,0 L0,6 L9,3z" fill="#8B5CF6"/>`{=html}`</marker>`{=html}`</defs>`{=html}
`<g fill="#0A2342" stroke="#168BFF" stroke-width="3">`{=html}
`<rect x="390" y="40" width="420" height="75" rx="18"/>`{=html}`<rect x="390" y="165" width="420" height="75" rx="18"/>`{=html}`<rect x="390" y="290" width="420" height="75" rx="18"/>`{=html}
`<rect x="390" y="415" width="420" height="75" rx="18"/>`{=html}`<rect x="390" y="540" width="420" height="75" rx="18"/>`{=html}`<rect x="390" y="665" width="420" height="75" rx="18"/>`{=html}`<rect x="390" y="790" width="420" height="75" rx="18"/>`{=html}
`</g>`{=html}
`<g stroke="#8B5CF6" stroke-width="3" marker-end="url(#a6)">`{=html}`<path d="M600 115 V165"/>`{=html}`<path d="M600 240 V290"/>`{=html}`<path d="M600 365 V415"/>`{=html}`<path d="M600 490 V540"/>`{=html}`<path d="M600 615 V665"/>`{=html}`<path d="M600 740 V790"/>`{=html}`</g>`{=html}
`<g fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="22" font-weight="700" text-anchor="middle">`{=html}
`<text x="600" y="87">`{=html}ApplicationFinished`</text>`{=html}`<text x="600" y="212">`{=html}Orchestrator`</text>`{=html}`<text x="600" y="337">`{=html}ProcessScoring`</text>`{=html}`<text x="600" y="462">`{=html}Scoring
Service`</text>`{=html}
`<text x="600" y="587">`{=html}ScoringFinished`</text>`{=html}`<text x="600" y="712">`{=html}Orchestrator`</text>`{=html}`<text x="600" y="837">`{=html}CalculatePerformance`</text>`{=html}
`</g>`{=html}
```{=html}
</svg>
```

------------------------------------------------------------------------

# 15. Retry e DLQ

Falhas temporárias passam por retry antes do envio para DLQ.

```{=html}
<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 390">
```
`<rect width="1200" height="390" rx="22" fill="#061426"/>`{=html}
`<defs>`{=html}`<marker id="a7" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">`{=html}`<path d="M0,0 L0,6 L9,3z" fill="#8B5CF6"/>`{=html}`</marker>`{=html}`</defs>`{=html}
`<g fill="#0A2342" stroke="#168BFF" stroke-width="3">`{=html}`<rect x="60" y="145" width="240" height="95" rx="18"/>`{=html}`<rect x="365" y="145" width="240" height="95" rx="18"/>`{=html}`<rect x="670" y="145" width="220" height="95" rx="18"/>`{=html}`<rect x="955" y="145" width="185" height="95" rx="18"/>`{=html}`</g>`{=html}
`<g stroke="#8B5CF6" stroke-width="3" marker-end="url(#a7)">`{=html}`<path d="M300 192 H365"/>`{=html}`<path d="M605 192 H670"/>`{=html}`<path d="M890 192 H955"/>`{=html}`</g>`{=html}
`<g fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="22" font-weight="700" text-anchor="middle">`{=html}`<text x="180" y="202">`{=html}Queue`</text>`{=html}`<text x="485" y="202">`{=html}Consumer`</text>`{=html}`<text x="780" y="202">`{=html}Retry`</text>`{=html}`<text x="1047" y="202">`{=html}DLQ`</text>`{=html}`</g>`{=html}
`<text x="780" y="285" fill="#BFD7F5" font-family="Inter,Segoe UI,Arial" font-size="18" text-anchor="middle">`{=html}falha
temporária → novas tentativas`</text>`{=html}
```{=html}
</svg>
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
<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 540">
```
`<rect width="1200" height="540" rx="22" fill="#061426"/>`{=html}
`<defs>`{=html}`<marker id="a8" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">`{=html}`<path d="M0,0 L0,6 L9,3z" fill="#8B5CF6"/>`{=html}`</marker>`{=html}`</defs>`{=html}
`<g fill="#0A2342" stroke="#168BFF" stroke-width="3">`{=html}`<rect x="430" y="45" width="340" height="80" rx="18"/>`{=html}`<rect x="430" y="180" width="340" height="80" rx="18"/>`{=html}`<rect x="140" y="350" width="340" height="80" rx="18"/>`{=html}`<rect x="720" y="350" width="340" height="80" rx="18"/>`{=html}`</g>`{=html}
`<path d="M600 125 V180" stroke="#8B5CF6" stroke-width="3" marker-end="url(#a8)"/>`{=html}`<path d="M520 260 C520 305 310 305 310 350" stroke="#8B5CF6" stroke-width="3" fill="none" marker-end="url(#a8)"/>`{=html}`<path d="M680 260 C680 305 890 305 890 350" stroke="#8B5CF6" stroke-width="3" fill="none" marker-end="url(#a8)"/>`{=html}
`<g fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="22" font-weight="700" text-anchor="middle">`{=html}`<text x="600" y="94">`{=html}Mensagem
recebida`</text>`{=html}`<text x="600" y="229">`{=html}eventId /
commandId já
processado?`</text>`{=html}`<text x="310" y="399">`{=html}Sim →
descartar`</text>`{=html}`<text x="890" y="399">`{=html}Não →
processar`</text>`{=html}`</g>`{=html}
```{=html}
</svg>
```

------------------------------------------------------------------------

# 17. Correlation ID

Todas as mensagens devem transportar:

``` text
correlationId
```

O mesmo identificador acompanha todo o workflow.

```{=html}
<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 330">
```
`<rect width="1200" height="330" rx="22" fill="#061426"/>`{=html}
`<defs>`{=html}`<marker id="a9" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">`{=html}`<path d="M0,0 L0,6 L9,3z" fill="#8B5CF6"/>`{=html}`</marker>`{=html}`</defs>`{=html}
`<g fill="#0A2342" stroke="#168BFF" stroke-width="3">`{=html}`<rect x="40" y="115" width="245" height="100" rx="18"/>`{=html}`<rect x="335" y="115" width="245" height="100" rx="18"/>`{=html}`<rect x="630" y="115" width="245" height="100" rx="18"/>`{=html}`<rect x="925" y="115" width="235" height="100" rx="18"/>`{=html}`</g>`{=html}
`<g stroke="#8B5CF6" stroke-width="3" marker-end="url(#a9)">`{=html}`<path d="M285 165 H335"/>`{=html}`<path d="M580 165 H630"/>`{=html}`<path d="M875 165 H925"/>`{=html}`</g>`{=html}
`<g fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="19" font-weight="700" text-anchor="middle">`{=html}`<text x="162" y="158">`{=html}ApplicationFinished`</text>`{=html}`<text x="457" y="158">`{=html}Orchestrator`</text>`{=html}`<text x="752" y="158">`{=html}Scoring
Service`</text>`{=html}`<text x="1042" y="158">`{=html}ScoringFinished`</text>`{=html}`</g>`{=html}
`<g fill="#BFD7F5" font-family="Inter,Segoe UI,Arial" font-size="16" text-anchor="middle">`{=html}`<text x="162" y="190">`{=html}ABC-123`</text>`{=html}`<text x="457" y="190">`{=html}ABC-123`</text>`{=html}`<text x="752" y="190">`{=html}ABC-123`</text>`{=html}`<text x="1042" y="190">`{=html}ABC-123`</text>`{=html}`</g>`{=html}
```{=html}
</svg>
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
<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 650">
```
`<rect width="1200" height="650" rx="22" fill="#061426"/>`{=html}
`<defs>`{=html}`<marker id="a10" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">`{=html}`<path d="M0,0 L0,6 L9,3z" fill="#8B5CF6"/>`{=html}`</marker>`{=html}`</defs>`{=html}
`<rect x="440" y="50" width="320" height="90" rx="18" fill="#0A2342" stroke="#168BFF" stroke-width="3"/>`{=html}`<text x="600" y="105" fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="26" font-weight="700" text-anchor="middle">`{=html}Kafka`</text>`{=html}
`<g fill="#0A2342" stroke="#168BFF" stroke-width="3">`{=html}`<rect x="80" y="240" width="280" height="85" rx="18"/>`{=html}`<rect x="460" y="240" width="280" height="85" rx="18"/>`{=html}`<rect x="840" y="240" width="280" height="85" rx="18"/>`{=html}`<rect x="270" y="455" width="280" height="85" rx="18"/>`{=html}`<rect x="650" y="455" width="280" height="85" rx="18"/>`{=html}`</g>`{=html}
`<g stroke="#8B5CF6" stroke-width="3" marker-end="url(#a10)">`{=html}`<path d="M520 140 L250 240"/>`{=html}`<path d="M600 140 V240"/>`{=html}`<path d="M680 140 L980 240"/>`{=html}`<path d="M540 325 L410 455"/>`{=html}`<path d="M660 325 L790 455"/>`{=html}`</g>`{=html}
`<g fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="20" font-weight="700" text-anchor="middle">`{=html}`<text x="220" y="292">`{=html}application.events`</text>`{=html}`<text x="600" y="292">`{=html}scoring.events`</text>`{=html}`<text x="980" y="292">`{=html}performance.events`</text>`{=html}`<text x="410" y="507">`{=html}ranking.events`</text>`{=html}`<text x="790" y="507">`{=html}answer-key
/ result events`</text>`{=html}`</g>`{=html}
```{=html}
</svg>
```

------------------------------------------------------------------------

# 21. Diagrama RabbitMQ

```{=html}
<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 650">
```
`<rect width="1200" height="650" rx="22" fill="#061426"/>`{=html}
`<defs>`{=html}`<marker id="a11" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">`{=html}`<path d="M0,0 L0,6 L9,3z" fill="#8B5CF6"/>`{=html}`</marker>`{=html}`</defs>`{=html}
`<g fill="#0A2342" stroke="#168BFF" stroke-width="3">`{=html}`<rect x="80" y="80" width="280" height="90" rx="18"/>`{=html}`<rect x="460" y="80" width="280" height="90" rx="18"/>`{=html}`<rect x="840" y="80" width="280" height="90" rx="18"/>`{=html}`<rect x="270" y="330" width="280" height="90" rx="18"/>`{=html}`<rect x="650" y="330" width="280" height="90" rx="18"/>`{=html}`</g>`{=html}
`<g stroke="#8B5CF6" stroke-width="3" marker-end="url(#a11)">`{=html}`<path d="M360 125 H460"/>`{=html}`<path d="M740 125 H840"/>`{=html}`<path d="M980 170 C980 265 790 265 790 330"/>`{=html}`<path d="M980 170 C980 265 410 265 410 330"/>`{=html}`</g>`{=html}
`<g fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="21" font-weight="700" text-anchor="middle">`{=html}`<text x="220" y="135">`{=html}Orchestrator`</text>`{=html}`<text x="600" y="135">`{=html}dia-d.commands`</text>`{=html}`<text x="980" y="135">`{=html}Work
Queue`</text>`{=html}`<text x="410" y="385">`{=html}Retry
Exchange`</text>`{=html}`<text x="790" y="385">`{=html}DLQ`</text>`{=html}`</g>`{=html}
```{=html}
</svg>
```

------------------------------------------------------------------------

# 22. Diagrama geral

```{=html}
<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 980">
```
`<rect width="1200" height="980" rx="22" fill="#061426"/>`{=html}
`<defs>`{=html}`<marker id="a12" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">`{=html}`<path d="M0,0 L0,6 L9,3z" fill="#8B5CF6"/>`{=html}`</marker>`{=html}`</defs>`{=html}
`<g fill="#0A2342" stroke="#168BFF" stroke-width="3">`{=html}
`<rect x="410" y="35" width="380" height="75" rx="18"/>`{=html}`<rect x="410" y="155" width="380" height="75" rx="18"/>`{=html}`<rect x="410" y="275" width="380" height="75" rx="18"/>`{=html}
`<rect x="100" y="430" width="300" height="75" rx="18"/>`{=html}`<rect x="450" y="430" width="300" height="75" rx="18"/>`{=html}`<rect x="800" y="430" width="300" height="75" rx="18"/>`{=html}
`<rect x="100" y="620" width="300" height="75" rx="18"/>`{=html}`<rect x="450" y="620" width="300" height="75" rx="18"/>`{=html}`<rect x="800" y="620" width="300" height="75" rx="18"/>`{=html}
`<rect x="450" y="810" width="300" height="75" rx="18"/>`{=html}
`</g>`{=html}
`<g stroke="#8B5CF6" stroke-width="3" fill="none" marker-end="url(#a12)">`{=html}`<path d="M600 110 V155"/>`{=html}`<path d="M600 230 V275"/>`{=html}`<path d="M500 350 L250 430"/>`{=html}`<path d="M600 350 V430"/>`{=html}`<path d="M700 350 L950 430"/>`{=html}`<path d="M250 505 V620"/>`{=html}`<path d="M600 505 V620"/>`{=html}`<path d="M950 505 V620"/>`{=html}`<path d="M600 695 V810"/>`{=html}`</g>`{=html}
`<g fill="#F8FAFC" font-family="Inter,Segoe UI,Arial" font-size="20" font-weight="700" text-anchor="middle">`{=html}`<text x="600" y="82">`{=html}Application
Service`</text>`{=html}`<text x="600" y="202">`{=html}Kafka ---
ApplicationFinished`</text>`{=html}`<text x="600" y="322">`{=html}Orchestrator`</text>`{=html}`<text x="250" y="477">`{=html}Scoring`</text>`{=html}`<text x="600" y="477">`{=html}Performance`</text>`{=html}`<text x="950" y="477">`{=html}Ranking`</text>`{=html}`<text x="250" y="667">`{=html}Answer
Key`</text>`{=html}`<text x="600" y="667">`{=html}Result`</text>`{=html}`<text x="950" y="667">`{=html}Communication`</text>`{=html}`<text x="600" y="857">`{=html}Audit
/ Observability`</text>`{=html}`</g>`{=html}
```{=html}
</svg>
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
