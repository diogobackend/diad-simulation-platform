# Eventos Assíncronos — Dia D Simulation

Documentação da comunicação assíncrona do **Dia D Simulation**, com foco em **Kafka**, **RabbitMQ**, eventos de domínio, comandos, tópicos, filas, retries, DLQ, idempotência, versionamento e rastreabilidade.

---

## Sumário

1. [Visão geral](#1-visão-geral)  
2. [Responsabilidade do Kafka e RabbitMQ](#2-responsabilidade-do-kafka-e-rabbitmq)  
3. [Fluxo assíncrono principal](#3-fluxo-assíncrono-principal)  
4. [Padrão dos eventos](#4-padrão-dos-eventos)  
5. [Eventos Kafka](#5-eventos-kafka)  
6. [Application Events](#6-application-events)  
7. [Scoring Events](#7-scoring-events)  
8. [Performance Events](#8-performance-events)  
9. [Ranking Events](#9-ranking-events)  
10. [Answer Key Events](#10-answer-key-events)  
11. [Communication Events](#11-communication-events)  
12. [▤ RabbitMQ](#12-rabbitmq)  
13. [Filas RabbitMQ](#13-filas-rabbitmq)  
14. [🧭 Orchestrator](#14-orchestrator)  
15. [Retry e DLQ](#15-retry-e-dlq)  
16. [Idempotência](#16-idempotência)  
17. [Correlation ID](#17-correlation-id)  
18. [Versionamento dos contratos](#18-versionamento-dos-contratos)  
19. [Nomenclatura](#19-nomenclatura)  
20. [Diagrama Kafka](#20-diagrama-kafka)  
21. [Diagrama RabbitMQ](#21-diagrama-rabbitmq)  
22. [Diagrama geral](#22-diagrama-geral)  
23. [Resumo da arquitetura assíncrona](#23-resumo-da-arquitetura-assíncrona)

---

# 1. Visão geral


A comunicação assíncrona da plataforma utiliza:

- **Kafka** para eventos de domínio;
- **RabbitMQ** para comandos e processamento direcionado;
- **Orchestrator** para controlar workflows distribuídos.

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 250" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="250" rx="18" fill="#06172E"/>
<defs><marker id="arrow1" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<rect x="470.0" y="93.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="131.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="15" font-weight="600" fill="#F8FAFC">🖥️ Application] --&gt; B[📊 Scoring</text>
</svg>

---

# 2. Responsabilidade do Kafka e RabbitMQ

## Kafka

O Kafka transporta **eventos de domínio**, ou seja, fatos que já ocorreram.

Exemplos:

```text
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

O RabbitMQ recebe **comandos assíncronos**, representando ações que devem ser executadas por um serviço específico.

Exemplos:

```text
ProcessScoring
CalculatePerformance
GenerateRanking
ReleaseAnswerKey
SendCommunication
```

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 276" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="276" rx="18" fill="#06172E"/>
<defs><marker id="arrow2" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<rect x="470.0" y="57.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="95.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="15" font-weight="600" fill="#F8FAFC">◉ Kafka] --&gt; E[⚡ Evento de domínio</text>
<rect x="470.0" y="155.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="193.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="15" font-weight="600" fill="#F8FAFC">▤ RabbitMQ] --&gt; C[➤ Comando assíncrono</text>
</svg>

---

# 3. Fluxo assíncrono principal

O fluxo de pós-prova é coordenado pelo Orchestrator.

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 1452" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="1452" rx="18" fill="#06172E"/>
<defs><marker id="arrow3" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<path d="M600.0,124.0 C600.0,153.0 600.0,153.0 600.0,182.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow3)" opacity="0.95"/>
<path d="M600.0,246.0 C600.0,275.0 600.0,275.0 600.0,304.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow3)" opacity="0.95"/>
<path d="M600.0,368.0 C600.0,397.0 600.0,397.0 600.0,426.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow3)" opacity="0.95"/>
<path d="M600.0,490.0 C600.0,519.0 600.0,519.0 600.0,548.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow3)" opacity="0.95"/>
<path d="M600.0,612.0 C600.0,641.0 600.0,641.0 600.0,670.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow3)" opacity="0.95"/>
<path d="M600.0,734.0 C600.0,763.0 600.0,763.0 600.0,792.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow3)" opacity="0.95"/>
<path d="M600.0,856.0 C600.0,885.0 600.0,885.0 600.0,914.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow3)" opacity="0.95"/>
<path d="M600.0,978.0 C600.0,1007.0 600.0,1007.0 600.0,1036.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow3)" opacity="0.95"/>
<path d="M600.0,1100.0 C600.0,1129.0 600.0,1129.0 600.0,1158.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow3)" opacity="0.95"/>
<path d="M600.0,1222.0 C600.0,1251.0 600.0,1251.0 600.0,1280.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow3)" opacity="0.95"/>
<rect x="470.0" y="60.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="98.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🏁 ApplicationFinished</text>
<rect x="470.0" y="182.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="220.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">⚙️ ProcessScoring</text>
<rect x="470.0" y="304.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="342.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📊 ScoringFinished</text>
<rect x="470.0" y="426.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="464.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📈 CalculatePerformance</text>
<rect x="470.0" y="548.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="586.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📈 PerformanceCalculated</text>
<rect x="470.0" y="670.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="708.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🏆 GenerateRanking</text>
<rect x="470.0" y="792.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="830.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🏆 RankingUpdated</text>
<rect x="470.0" y="914.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="952.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🔑 ReleaseAnswerKey</text>
<rect x="470.0" y="1036.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="1074.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🔑 AnswerKeyReleased</text>
<rect x="470.0" y="1158.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="1196.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📄 ResultAvailable</text>
<rect x="470.0" y="1280.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="1318.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">💬 SendCommunication</text>
</svg>

---

# 4. Padrão dos eventos

Todos os eventos seguem um envelope comum.

```json
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

| Campo | Descrição |
|---|---|
| `eventId` | Identificador único do evento |
| `eventType` | Tipo do evento |
| `eventVersion` | Versão do contrato |
| `occurredAt` | Data e hora do evento |
| `correlationId` | Identificador de rastreabilidade |
| `aggregateId` | Entidade principal relacionada |
| `payload` | Dados específicos do evento |

---

# 5. Eventos Kafka

| Tópico | Responsabilidade |
|---|---|
| `application.events` | Eventos da execução da prova |
| `scoring.events` | Eventos de correção |
| `performance.events` | Eventos de desempenho |
| `ranking.events` | Eventos de classificação |
| `answer-key.events` | Eventos de gabarito |
| `result.events` | Eventos de resultado |
| `communication.events` | Eventos de comunicação |

---

# 6. Application Events

## `ApplicationStarted`

Publicado quando o candidato inicia a prova.

```json
{
  "applicationId": "uuid",
  "candidateId": "uuid",
  "examId": "uuid",
  "startedAt": "timestamp"
}
```

## `AnswerSubmitted`

Publicado quando uma resposta é registrada.

```json
{
  "applicationId": "uuid",
  "questionId": "uuid",
  "alternativeId": "uuid",
  "answeredAt": "timestamp"
}
```

## `ApplicationFinished`

Publicado quando a prova é finalizada.

```json
{
  "applicationId": "uuid",
  "candidateId": "uuid",
  "examId": "uuid",
  "finishedAt": "timestamp"
}
```

Consumidores principais:

- Orchestrator Service
- Audit Service

---

# 7. Scoring Events

## `ScoringStarted`

```json
{
  "applicationId": "uuid",
  "scoringId": "uuid",
  "startedAt": "timestamp"
}
```

## `ScoringFinished`

```json
{
  "applicationId": "uuid",
  "candidateId": "uuid",
  "examId": "uuid",
  "scoreId": "uuid",
  "finalScore": 782.45,
  "finishedAt": "timestamp"
}
```

Consumidores principais:

- Orchestrator Service
- Performance Service
- Audit Service

## `ScoringFailed`

```json
{
  "applicationId": "uuid",
  "scoringId": "uuid",
  "reason": "string",
  "failedAt": "timestamp"
}
```

---

# 8. Performance Events

## `PerformanceCalculated`

```json
{
  "candidateId": "uuid",
  "applicationId": "uuid",
  "scoreId": "uuid",
  "performanceId": "uuid",
  "overallPercentage": 81.50,
  "percentile": 92.30,
  "calculatedAt": "timestamp"
}
```

Consumidores:

- Orchestrator Service
- Ranking Service
- Audit Service

---

# 9. Ranking Events

## `RankingUpdated`

```json
{
  "candidateId": "uuid",
  "examEventId": "uuid",
  "rankingId": "uuid",
  "position": 152,
  "score": 782.45,
  "updatedAt": "timestamp"
}
```

Consumidores:

- Orchestrator Service
- Audit Service

---

# 10. Answer Key Events

## `AnswerKeyReleased`

```json
{
  "examId": "uuid",
  "publicationId": "uuid",
  "version": 1,
  "releasedAt": "timestamp"
}
```

Consumidores:

- Orchestrator Service
- Communication Service
- Audit Service

---

# 11. Communication Events

## `ResultAvailable`

Publicado quando o resultado final está disponível para consulta.

```json
{
  "candidateId": "uuid",
  "applicationId": "uuid",
  "scoreId": "uuid",
  "performanceId": "uuid",
  "availableAt": "timestamp"
}
```

Consumidores:

- Communication Service
- Audit Service

## `CommunicationSent`

```json
{
  "communicationId": "uuid",
  "candidateId": "uuid",
  "channel": "EMAIL",
  "sentAt": "timestamp"
}
```

## `CommunicationFailed`

```json
{
  "communicationId": "uuid",
  "candidateId": "uuid",
  "channel": "EMAIL",
  "reason": "string",
  "failedAt": "timestamp"
}
```

---

# 12. RabbitMQ

O RabbitMQ executa o processamento direcionado dos comandos.

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 250" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="250" rx="18" fill="#06172E"/>
<defs><marker id="arrow4" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<rect x="470.0" y="93.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="131.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="15" font-weight="600" fill="#F8FAFC">📤 Producer] --&gt; X[🔀 Exchange</text>
</svg>

Exchanges principais:

```text
dia-d.commands
dia-d.retry
dia-d.dlx
```

---

# 13. Filas RabbitMQ

| Fila | Consumidor |
|---|---|
| `scoring.process.queue` | Scoring Service |
| `performance.calculate.queue` | Performance Service |
| `ranking.generate.queue` | Ranking Service |
| `answer-key.release.queue` | Answer Key Service |
| `communication.send.queue` | Communication Service |

## `scoring.process.queue`

```json
{
  "commandId": "uuid",
  "applicationId": "uuid",
  "candidateId": "uuid",
  "examId": "uuid",
  "correlationId": "uuid"
}
```

## `performance.calculate.queue`

```json
{
  "commandId": "uuid",
  "candidateId": "uuid",
  "applicationId": "uuid",
  "scoreId": "uuid",
  "correlationId": "uuid"
}
```

## `ranking.generate.queue`

```json
{
  "commandId": "uuid",
  "candidateId": "uuid",
  "examEventId": "uuid",
  "performanceId": "uuid",
  "correlationId": "uuid"
}
```

## `answer-key.release.queue`

```json
{
  "commandId": "uuid",
  "examId": "uuid",
  "correlationId": "uuid"
}
```

## `communication.send.queue`

```json
{
  "commandId": "uuid",
  "candidateId": "uuid",
  "communicationType": "RESULT_AVAILABLE",
  "channel": "EMAIL",
  "correlationId": "uuid"
}
```

---

# 14. Orchestrator

O Orchestrator coordena a sequência do workflow assíncrono.

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 842" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="842" rx="18" fill="#06172E"/>
<defs><marker id="arrow5" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<path d="M600.0,124.0 C600.0,153.0 600.0,153.0 600.0,182.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow5)" opacity="0.95"/>
<path d="M600.0,246.0 C600.0,275.0 600.0,275.0 600.0,304.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow5)" opacity="0.95"/>
<path d="M600.0,368.0 C600.0,397.0 600.0,397.0 600.0,426.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow5)" opacity="0.95"/>
<path d="M600.0,490.0 C600.0,519.0 600.0,519.0 600.0,548.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow5)" opacity="0.95"/>
<path d="M600.0,612.0 C600.0,641.0 600.0,641.0 600.0,670.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow5)" opacity="0.95"/>
<rect x="450.0" y="60.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="98.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">⚡ Evento</text>
<rect x="450.0" y="182.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="220.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">⛓ Workflow</text>
<rect x="450.0" y="304.0" width="300" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="342.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">➤ Comando RabbitMQ</text>
<rect x="450.0" y="426.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="464.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">⚙️ Serviço de domínio</text>
<rect x="450.0" y="548.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="586.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ Evento Kafka</text>
<rect x="450.0" y="670.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="708.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">➡ Próxima etapa</text>
</svg>

Exemplo aplicado:

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 964" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="964" rx="18" fill="#06172E"/>
<defs><marker id="arrow6" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<path d="M600.0,124.0 C600.0,153.0 600.0,153.0 600.0,182.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow6)" opacity="0.95"/>
<path d="M600.0,246.0 C600.0,275.0 600.0,275.0 600.0,304.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow6)" opacity="0.95"/>
<path d="M600.0,368.0 C600.0,397.0 600.0,397.0 600.0,426.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow6)" opacity="0.95"/>
<path d="M600.0,490.0 C600.0,519.0 600.0,519.0 600.0,548.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow6)" opacity="0.95"/>
<path d="M600.0,612.0 C600.0,641.0 600.0,641.0 600.0,670.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow6)" opacity="0.95"/>
<path d="M600.0,734.0 C600.0,763.0 600.0,763.0 600.0,792.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow6)" opacity="0.95"/>
<rect x="450.0" y="60.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="98.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🏁 ApplicationFinished</text>
<rect x="450.0" y="182.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="220.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🧭 Orchestrator</text>
<rect x="450.0" y="304.0" width="300" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="342.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">⚙️ ProcessScoring</text>
<rect x="450.0" y="426.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="464.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📊 Scoring Service</text>
<rect x="450.0" y="548.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="586.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📊 ScoringFinished</text>
<rect x="450.0" y="670.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="708.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🧭 Orchestrator</text>
<rect x="450.0" y="792.0" width="300" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="830.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📈 CalculatePerformance</text>
</svg>

---

# 15. Retry e DLQ

Falhas temporárias passam por retry antes do envio para DLQ.

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 250" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="250" rx="18" fill="#06172E"/>
<defs><marker id="arrow7" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<rect x="470.0" y="93.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="131.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">▤ Queue] --&gt; C[📥 Consumer</text>
</svg>

DLQs previstas:

```text
scoring.process.dlq
performance.calculate.dlq
ranking.generate.dlq
answer-key.release.dlq
communication.send.dlq
```

Política inicial:

```text
Tentativa 1 -> 5 segundos
Tentativa 2 -> 30 segundos
Tentativa 3 -> 2 minutos
Tentativa 4 -> DLQ
```

---

# 16. Idempotência

Eventos e comandos podem ser entregues mais de uma vez.

O consumidor deve validar:

```text
eventId
commandId
```

antes de executar novamente.

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 598" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="598" rx="18" fill="#06172E"/>
<defs><marker id="arrow8" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<path d="M600.0,124.0 C600.0,153.0 600.0,153.0 600.0,182.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow8)" opacity="0.95"/>
<path d="M600.0,246.0 C600.0,275.0 430.0,275.0 430.0,304.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow8)" opacity="0.95"/>
<rect x="480.0" y="251.0" width="70" height="26" rx="8" fill="#06172E"/>
<text x="515.0" y="269.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">Sim</text>
<path d="M600.0,246.0 C600.0,275.0 770.0,275.0 770.0,304.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow8)" opacity="0.95"/>
<rect x="650.0" y="251.0" width="70" height="26" rx="8" fill="#06172E"/>
<text x="685.0" y="269.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">Não</text>
<path d="M770.0,368.0 C770.0,397.0 600.0,397.0 600.0,426.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow8)" opacity="0.95"/>
<rect x="450.0" y="60.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="98.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📨 Mensagem recebida</text>
<rect x="450.0" y="182.0" width="300" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="220.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">ID já processado?</text>
<rect x="280.0" y="304.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="430.0" y="342.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">✓ ACK</text>
<rect x="620.0" y="304.0" width="300" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="770.0" y="342.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">⚙️ Processa</text>
<rect x="450.0" y="426.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="464.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📝 Registra processamento</text>
</svg>

---

# 17. Correlation ID

Todas as mensagens devem transportar:

```text
correlationId
```

O mesmo identificador acompanha todo o workflow.

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 250" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="250" rx="18" fill="#06172E"/>
<defs><marker id="arrow9" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<path d="M310.0,125.0 C320.0,125.0 320.0,125.0 330.0,125.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow9)" opacity="0.95"/>
<rect x="280.0" y="100.0" width="80" height="26" rx="8" fill="#06172E"/>
<text x="320.0" y="118.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">ABC-123</text>
<path d="M590.0,125.0 C600.0,125.0 600.0,125.0 610.0,125.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow9)" opacity="0.95"/>
<rect x="560.0" y="100.0" width="80" height="26" rx="8" fill="#06172E"/>
<text x="600.0" y="118.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">ABC-123</text>
<path d="M870.0,125.0 C880.0,125.0 880.0,125.0 890.0,125.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow9)" opacity="0.95"/>
<rect x="840.0" y="100.0" width="80" height="26" rx="8" fill="#06172E"/>
<text x="880.0" y="118.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">ABC-123</text>
<rect x="50.0" y="93.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="180.0" y="131.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🏁 ApplicationFinished</text>
<rect x="330.0" y="93.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="460.0" y="131.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">⚙️ ProcessScoring</text>
<rect x="610.0" y="93.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="740.0" y="131.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📊 ScoringFinished</text>
<rect x="890.0" y="93.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="1020.0" y="131.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📈 PerformanceCalculated</text>
</svg>

---

# 18. Versionamento dos contratos

Eventos devem possuir versão explícita.

```json
{
  "eventType": "ScoringFinished",
  "eventVersion": 1
}
```

Alterações incompatíveis geram uma nova versão do contrato:

```text
ScoringFinished v1
ScoringFinished v2
```

---

# 19. Nomenclatura

## Eventos

Formato:

```text
PascalCase
```

Exemplos:

```text
ApplicationFinished
ScoringFinished
PerformanceCalculated
RankingUpdated
ResultAvailable
```

Eventos representam fatos e utilizam passado.

## Tópicos Kafka

Formato:

```text
<dominio>.events
```

Exemplos:

```text
application.events
scoring.events
performance.events
ranking.events
answer-key.events
result.events
communication.events
```

## Filas RabbitMQ

Formato:

```text
<dominio>.<acao>.queue
```

Exemplos:

```text
scoring.process.queue
performance.calculate.queue
ranking.generate.queue
communication.send.queue
```

DLQ:

```text
<dominio>.<acao>.dlq
```

---

# 20. Diagrama Kafka

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 908" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="908" rx="18" fill="#06172E"/>
<defs><marker id="arrow10" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<path d="M262.0,119.0 C262.0,144.0 262.0,144.0 262.0,169.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<rect x="174.0" y="120.0" width="176" height="26" rx="8" fill="#06172E"/>
<text x="262.0" y="138.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">ApplicationFinished</text>
<path d="M262.0,233.0 C262.0,486.0 600.0,486.0 600.0,739.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<path d="M600.0,119.0 C600.0,144.0 600.0,144.0 600.0,169.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<rect x="528.0" y="120.0" width="144" height="26" rx="8" fill="#06172E"/>
<text x="600.0" y="138.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">ScoringFinished</text>
<path d="M600.0,233.0 C600.0,486.0 600.0,486.0 600.0,739.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<path d="M600.0,233.0 C600.0,258.0 600.0,258.0 600.0,283.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<path d="M600.0,347.0 C600.0,372.0 769.0,372.0 769.0,397.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<rect x="588.5" y="348.0" width="192" height="26" rx="8" fill="#06172E"/>
<text x="684.5" y="366.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">PerformanceCalculated</text>
<path d="M769.0,461.0 C769.0,600.0 600.0,600.0 600.0,739.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<path d="M769.0,461.0 C769.0,486.0 769.0,486.0 769.0,511.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<path d="M769.0,575.0 C769.0,600.0 600.0,600.0 600.0,625.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<rect x="616.5" y="576.0" width="136" height="26" rx="8" fill="#06172E"/>
<text x="684.5" y="594.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">RankingUpdated</text>
<path d="M600.0,689.0 C600.0,714.0 600.0,714.0 600.0,739.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<path d="M938.0,119.0 C938.0,144.0 938.0,144.0 938.0,169.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<rect x="858.0" y="120.0" width="160" height="26" rx="8" fill="#06172E"/>
<text x="938.0" y="138.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">AnswerKeyReleased</text>
<path d="M938.0,233.0 C938.0,486.0 600.0,486.0 600.0,739.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<path d="M938.0,233.0 C938.0,372.0 431.0,372.0 431.0,511.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<path d="M600.0,803.0 C600.0,600.0 431.0,600.0 431.0,397.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<rect x="443.5" y="576.0" width="144" height="26" rx="8" fill="#06172E"/>
<text x="515.5" y="594.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">ResultAvailable</text>
<path d="M431.0,461.0 C431.0,486.0 431.0,486.0 431.0,511.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow10)" opacity="0.95"/>
<rect x="112.0" y="55.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="262.0" y="93.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🖥️ Application Service</text>
<rect x="450.0" y="55.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="93.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📊 Scoring Service</text>
<rect x="450.0" y="283.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="321.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📈 Performance Service</text>
<rect x="619.0" y="511.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="769.0" y="549.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🏆 Ranking Service</text>
<rect x="788.0" y="55.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="938.0" y="93.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🔑 Answer Key Service</text>
<rect x="281.0" y="511.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="431.0" y="549.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">💬 Communication Service</text>
<rect x="450.0" y="739.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="777.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🧭 Orchestrator</text>
<rect x="112.0" y="169.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="262.0" y="207.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ application.events</text>
<rect x="450.0" y="169.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="207.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ scoring.events</text>
<rect x="619.0" y="397.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="769.0" y="435.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ performance.events</text>
<rect x="450.0" y="625.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="663.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ ranking.events</text>
<rect x="788.0" y="169.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="938.0" y="207.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ answer-key.events</text>
<rect x="281.0" y="397.0" width="300" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="431.0" y="435.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ result.events</text>
</svg>

---

# 21. Diagrama RabbitMQ

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 476" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="476" rx="18" fill="#06172E"/>
<defs><marker id="arrow11" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<path d="M600.0,124.0 C600.0,153.0 0.0,153.0 0.0,182.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow11)" opacity="0.95"/>
<path d="M0.0,246.0 C0.0,275.0 0.0,275.0 0.0,304.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow11)" opacity="0.95"/>
<path d="M600.0,124.0 C600.0,153.0 300.0,153.0 300.0,182.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow11)" opacity="0.95"/>
<path d="M300.0,246.0 C300.0,275.0 300.0,275.0 300.0,304.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow11)" opacity="0.95"/>
<path d="M600.0,124.0 C600.0,153.0 600.0,153.0 600.0,182.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow11)" opacity="0.95"/>
<path d="M600.0,246.0 C600.0,275.0 600.0,275.0 600.0,304.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow11)" opacity="0.95"/>
<path d="M600.0,124.0 C600.0,153.0 900.0,153.0 900.0,182.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow11)" opacity="0.95"/>
<path d="M900.0,246.0 C900.0,275.0 900.0,275.0 900.0,304.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow11)" opacity="0.95"/>
<path d="M600.0,124.0 C600.0,153.0 1200.0,153.0 1200.0,182.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow11)" opacity="0.95"/>
<path d="M1200.0,246.0 C1200.0,275.0 1200.0,275.0 1200.0,304.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow11)" opacity="0.95"/>
<rect x="470.0" y="60.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="98.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🧭 Orchestrator</text>
<rect x="-130.0" y="182.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="0.0" y="220.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">scoring.process.queue</text>
<rect x="170.0" y="182.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="300.0" y="220.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="15" font-weight="600" fill="#F8FAFC">performance.calculate.queue</text>
<rect x="470.0" y="182.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="220.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">ranking.generate.queue</text>
<rect x="770.0" y="182.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="900.0" y="220.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">answer-key.release.queue</text>
<rect x="1070.0" y="182.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="1200.0" y="220.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">communication.send.queue</text>
<rect x="-130.0" y="304.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="0.0" y="342.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📊 Scoring Service</text>
<rect x="170.0" y="304.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="300.0" y="342.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📈 Performance Service</text>
<rect x="470.0" y="304.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="342.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🏆 Ranking Service</text>
<rect x="770.0" y="304.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="900.0" y="342.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🔑 Answer Key Service</text>
<rect x="1070.0" y="304.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="1200.0" y="342.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">💬 Communication Service</text>
</svg>

---

# 22. Diagrama geral

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 964" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="964" rx="18" fill="#06172E"/>
<defs><marker id="arrow12" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<path d="M600.0,124.0 C600.0,153.0 600.0,153.0 600.0,182.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<rect x="512.0" y="129.0" width="176" height="26" rx="8" fill="#06172E"/>
<text x="600.0" y="147.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">ApplicationFinished</text>
<path d="M600.0,246.0 C600.0,519.0 600.0,519.0 600.0,792.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<path d="M600.0,856.0 C600.0,641.0 0.0,641.0 0.0,426.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<rect x="232.0" y="617.0" width="136" height="26" rx="8" fill="#06172E"/>
<text x="300.0" y="635.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">ProcessScoring</text>
<path d="M0.0,490.0 C0.0,519.0 0.0,519.0 0.0,548.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<path d="M0.0,612.0 C0.0,641.0 150.0,641.0 150.0,670.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<rect x="3.0" y="617.0" width="144" height="26" rx="8" fill="#06172E"/>
<text x="75.0" y="635.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">ScoringFinished</text>
<path d="M150.0,734.0 C150.0,763.0 600.0,763.0 600.0,792.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<path d="M600.0,856.0 C600.0,641.0 300.0,641.0 300.0,426.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<rect x="358.0" y="617.0" width="184" height="26" rx="8" fill="#06172E"/>
<text x="450.0" y="635.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">CalculatePerformance</text>
<path d="M300.0,490.0 C300.0,519.0 300.0,519.0 300.0,548.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<path d="M300.0,612.0 C300.0,641.0 450.0,641.0 450.0,670.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<rect x="279.0" y="617.0" width="192" height="26" rx="8" fill="#06172E"/>
<text x="375.0" y="635.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">PerformanceCalculated</text>
<path d="M450.0,734.0 C450.0,763.0 600.0,763.0 600.0,792.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<path d="M600.0,856.0 C600.0,641.0 600.0,641.0 600.0,426.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<rect x="528.0" y="617.0" width="144" height="26" rx="8" fill="#06172E"/>
<text x="600.0" y="635.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">GenerateRanking</text>
<path d="M600.0,490.0 C600.0,519.0 600.0,519.0 600.0,548.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<path d="M600.0,612.0 C600.0,641.0 750.0,641.0 750.0,670.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<rect x="607.0" y="617.0" width="136" height="26" rx="8" fill="#06172E"/>
<text x="675.0" y="635.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">RankingUpdated</text>
<path d="M750.0,734.0 C750.0,763.0 600.0,763.0 600.0,792.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<path d="M600.0,856.0 C600.0,641.0 900.0,641.0 900.0,426.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<rect x="674.0" y="617.0" width="152" height="26" rx="8" fill="#06172E"/>
<text x="750.0" y="635.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">ReleaseAnswerKey</text>
<path d="M900.0,490.0 C900.0,519.0 900.0,519.0 900.0,548.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<path d="M900.0,612.0 C900.0,641.0 1050.0,641.0 1050.0,670.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<rect x="895.0" y="617.0" width="160" height="26" rx="8" fill="#06172E"/>
<text x="975.0" y="635.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">AnswerKeyReleased</text>
<path d="M1050.0,734.0 C1050.0,763.0 600.0,763.0 600.0,792.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<path d="M600.0,856.0 C600.0,641.0 1200.0,641.0 1200.0,426.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<rect x="828.0" y="617.0" width="144" height="26" rx="8" fill="#06172E"/>
<text x="900.0" y="635.0" text-anchor="middle" font-family="Inter,Segoe UI,Arial,sans-serif" font-size="14" fill="#BED7F3">ResultAvailable</text>
<path d="M1200.0,490.0 C1200.0,519.0 1200.0,519.0 1200.0,548.0" fill="none" stroke="#38BDF8" stroke-width="2.2" marker-end="url(#arrow12)" opacity="0.95"/>
<rect x="470.0" y="60.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="98.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🖥️ Application Service</text>
<rect x="470.0" y="792.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="830.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🧭 Orchestrator</text>
<rect x="-130.0" y="548.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="0.0" y="586.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📊 Scoring Service</text>
<rect x="170.0" y="548.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="300.0" y="586.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">📈 Performance Service</text>
<rect x="470.0" y="548.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="586.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🏆 Ranking Service</text>
<rect x="770.0" y="548.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="900.0" y="586.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🔑 Answer Key Service</text>
<rect x="1070.0" y="548.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="1200.0" y="586.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">💬 Communication Service</text>
<rect x="470.0" y="182.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="220.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ Kafka</text>
<rect x="-130.0" y="426.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="0.0" y="464.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">▤ RabbitMQ</text>
<rect x="20.0" y="670.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="150.0" y="708.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ Kafka</text>
<rect x="170.0" y="426.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="300.0" y="464.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">▤ RabbitMQ</text>
<rect x="320.0" y="670.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="450.0" y="708.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ Kafka</text>
<rect x="470.0" y="426.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="464.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">▤ RabbitMQ</text>
<rect x="620.0" y="670.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="750.0" y="708.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ Kafka</text>
<rect x="770.0" y="426.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="900.0" y="464.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">▤ RabbitMQ</text>
<rect x="920.0" y="670.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="1050.0" y="708.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ Kafka</text>
<rect x="1070.0" y="426.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="1200.0" y="464.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ Kafka</text>
</svg>

---

# 23. Resumo da arquitetura assíncrona

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 1200 374" role="img" aria-label="Diagrama de eventos assíncronos">
<rect x="0" y="0" width="1200" height="374" rx="18" fill="#06172E"/>
<defs><marker id="arrow13" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth"><path d="M0,0 L0,6 L9,3 z" fill="#38BDF8"/></marker></defs>
<rect x="470.0" y="57.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="95.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">◉ Kafka</text>
<rect x="470.0" y="155.0" width="260" height="64" rx="16" fill="#12375F" stroke="#38BDF8" stroke-width="2.4"/>
<text x="600.0" y="193.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">▤ RabbitMQ</text>
<rect x="470.0" y="253.0" width="260" height="64" rx="16" fill="#0D2542" stroke="#2F8CFF" stroke-width="2.4"/>
<text x="600.0" y="291.0" text-anchor="middle" font-family="Inter,Segoe UI Emoji,Segoe UI,Arial,sans-serif" font-size="17" font-weight="600" fill="#F8FAFC">🧭 Orchestrator</text>
</svg>

A separação principal é:

```text
Evento = fato ocorrido
Comando = ação a executar
```

No Dia D Simulation:

- **Kafka** propaga eventos de domínio;
- **RabbitMQ** direciona comandos para processamento;
- **Orchestrator** controla a sequência dos workflows;
- **Retry e DLQ** tratam falhas;
- **Idempotência** evita processamento duplicado;
- **Correlation ID** garante rastreabilidade entre serviços.
