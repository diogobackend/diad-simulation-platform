# Chamadas Síncronas --- REST / HTTP

Documentação dos contratos síncronos da plataforma **Dia D Simulation**,
detalhando as APIs REST/HTTP, responsabilidades, endpoints, requests,
responses, códigos HTTP e comunicação entre aplicações.

> **Escopo:** esta documentação cobre as chamadas síncronas.
> Processamentos demorados ou desacoplados, como correção completa,
> cálculo de desempenho, geração de ranking, publicação de resultado e
> comunicações, permanecem no fluxo assíncrono.

------------------------------------------------------------------------

## Sumário

1.  [Visão geral](#1-visão-geral)
2.  [Padrões REST e HTTP](#2-padrões-rest-e-http)
3.  [Headers padrão](#3-headers-padrão)
4.  [Contrato padrão de erro](#4-contrato-padrão-de-erro)
5.  [Auth Service](#5-auth-service)
6.  [Candidate Service](#6-candidate-service)
7.  [Exam Service](#7-exam-service)
8.  [Question Service](#8-question-service)
9.  [Application Service](#9-application-service)
10. [Answer Service](#10-answer-service)
11. [Scoring Service](#11-scoring-service)
12. [Performance Service](#12-performance-service)
13. [Ranking Service](#13-ranking-service)
14. [Answer Key Service](#14-answer-key-service)
15. [Result Service](#15-result-service)
16. [Communication Service](#16-communication-service)
17. [Orchestrator Service](#17-orchestrator-service)
18. [Audit Service](#18-audit-service)
19. [Fluxos síncronos entre
    aplicações](#19-fluxos-síncronos-entre-aplicações)
20. [Códigos HTTP](#20-códigos-http)
21. [Resumo dos endpoints](#21-resumo-dos-endpoints)

------------------------------------------------------------------------

# 1. Visão geral

As chamadas síncronas são utilizadas quando o cliente ou outro serviço
precisa de uma resposta imediata.

O padrão adotado é:

``` text
Cliente
   |
   v
API Gateway
   |
   v
REST / HTTP
   |
   v
Application Service
```

Responsabilidades principais:

-   **REST/HTTP:** consultas e comandos que precisam de resposta
    imediata;
-   **Kafka:** propagação de fatos de domínio;
-   **RabbitMQ:** execução assíncrona de comandos;
-   **Orchestrator:** coordenação de workflows distribuídos.

------------------------------------------------------------------------

# 2. Padrões REST e HTTP

Base sugerida:

``` text
/api/v1
```

Convenções:

``` text
GET     -> consulta
POST    -> criação ou execução de ação
PUT     -> substituição completa
PATCH   -> alteração parcial
DELETE  -> remoção
```

Recursos são representados por substantivos:

``` text
/candidates
/exams
/questions
/applications
/answers
/scores
/performances
/rankings
/answer-keys
/results
```

------------------------------------------------------------------------

# 3. Headers padrão

## Request

``` http
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json
X-Correlation-Id: 3f2f8468-5d16-4af8-b65e-f624a17cc973
```

O `X-Correlation-Id` deve acompanhar chamadas entre serviços para
permitir rastreabilidade ponta a ponta.

## Response

``` http
Content-Type: application/json
X-Correlation-Id: 3f2f8468-5d16-4af8-b65e-f624a17cc973
```

------------------------------------------------------------------------

# 4. Contrato padrão de erro

``` json
{
  "timestamp": "2026-08-30T17:00:00Z",
  "status": 404,
  "error": "NOT_FOUND",
  "message": "Application not found",
  "path": "/api/v1/applications/0f53241e",
  "correlationId": "3f2f8468-5d16-4af8-b65e-f624a17cc973"
}
```

Para erros de validação:

``` json
{
  "timestamp": "2026-08-30T17:00:00Z",
  "status": 422,
  "error": "VALIDATION_ERROR",
  "message": "Invalid request",
  "fields": [
    {
      "field": "municipalityCode",
      "message": "must not be blank"
    }
  ],
  "correlationId": "3f2f8468-5d16-4af8-b65e-f624a17cc973"
}
```

------------------------------------------------------------------------

# 5. Auth Service

Responsável por autenticação, renovação de sessão e consulta do usuário
autenticado.

## POST `/api/v1/auth/login`

Autentica o candidato.

### Request

``` json
{
  "email": "candidato@email.com",
  "password": "senha"
}
```

### Response --- `200 OK`

``` json
{
  "accessToken": "jwt",
  "refreshToken": "jwt",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "candidate": {
    "id": "candidate-uuid",
    "name": "João da Silva",
    "email": "candidato@email.com"
  }
}
```

### Erros

``` text
400 Bad Request
401 Unauthorized
422 Unprocessable Entity
```

------------------------------------------------------------------------

## POST `/api/v1/auth/refresh`

### Request

``` json
{
  "refreshToken": "jwt"
}
```

### Response --- `200 OK`

``` json
{
  "accessToken": "novo-jwt",
  "refreshToken": "novo-refresh-token",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```

------------------------------------------------------------------------

## GET `/api/v1/auth/me`

### Response --- `200 OK`

``` json
{
  "id": "candidate-uuid",
  "name": "João da Silva",
  "email": "candidato@email.com",
  "roles": [
    "CANDIDATE"
  ]
}
```

------------------------------------------------------------------------

# 6. Candidate Service

Responsável pelo cadastro e perfil do candidato.

## POST `/api/v1/candidates`

### Request

``` json
{
  "name": "João da Silva",
  "email": "candidato@email.com",
  "password": "senha",
  "document": "00000000000",
  "birthDate": "2004-05-18",
  "municipalityCode": "2111300"
}
```

### Response --- `201 Created`

``` json
{
  "id": "candidate-uuid",
  "name": "João da Silva",
  "email": "candidato@email.com",
  "document": "00000000000",
  "birthDate": "2004-05-18",
  "municipalityCode": "2111300",
  "status": "ACTIVE",
  "createdAt": "2026-08-30T15:20:00Z"
}
```

### Erros

``` text
409 Conflict -> e-mail ou documento já cadastrado
422 Unprocessable Entity -> dados inválidos
```

------------------------------------------------------------------------

## GET `/api/v1/candidates/{candidateId}`

### Response --- `200 OK`

``` json
{
  "id": "candidate-uuid",
  "name": "João da Silva",
  "email": "candidato@email.com",
  "document": "00000000000",
  "birthDate": "2004-05-18",
  "municipalityCode": "2111300",
  "status": "ACTIVE"
}
```

------------------------------------------------------------------------

## PATCH `/api/v1/candidates/{candidateId}`

### Request

``` json
{
  "name": "João Silva",
  "municipalityCode": "2111300"
}
```

### Response --- `200 OK`

``` json
{
  "id": "candidate-uuid",
  "name": "João Silva",
  "email": "candidato@email.com",
  "municipalityCode": "2111300",
  "status": "ACTIVE",
  "updatedAt": "2026-08-30T16:00:00Z"
}
```

------------------------------------------------------------------------

# 7. Exam Service

Responsável pela definição da prova, datas, duração e configuração do
simulado.

## GET `/api/v1/exams`

### Response --- `200 OK`

``` json
{
  "content": [
    {
      "id": "exam-uuid",
      "name": "Dia D ENEM 2026",
      "applicationDate": "2026-11-08",
      "startTime": "12:00:00",
      "endTime": "17:30:00",
      "status": "SCHEDULED"
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 1
}
```

------------------------------------------------------------------------

## GET `/api/v1/exams/{examId}`

### Response --- `200 OK`

``` json
{
  "id": "exam-uuid",
  "name": "Dia D ENEM 2026",
  "applicationDate": "2026-11-08",
  "startTime": "12:00:00",
  "endTime": "17:30:00",
  "durationMinutes": 330,
  "questionCount": 90,
  "status": "SCHEDULED"
}
```

------------------------------------------------------------------------

## POST `/api/v1/exams`

Endpoint administrativo.

### Request

``` json
{
  "name": "Dia D ENEM 2026",
  "applicationDate": "2026-11-08",
  "startTime": "12:00:00",
  "endTime": "17:30:00",
  "questionCount": 90
}
```

### Response --- `201 Created`

``` json
{
  "id": "exam-uuid",
  "name": "Dia D ENEM 2026",
  "status": "DRAFT"
}
```

------------------------------------------------------------------------

# 8. Question Service

Responsável pelo banco de questões e alternativas.

## GET `/api/v1/exams/{examId}/questions`

Retorna as questões disponibilizadas para a aplicação.

### Response --- `200 OK`

``` json
{
  "examId": "exam-uuid",
  "questions": [
    {
      "id": "question-uuid",
      "number": 1,
      "area": "MATHEMATICS",
      "statement": "Enunciado da questão...",
      "alternatives": [
        {
          "id": "alternative-a",
          "label": "A",
          "text": "Alternativa A"
        },
        {
          "id": "alternative-b",
          "label": "B",
          "text": "Alternativa B"
        }
      ]
    }
  ]
}
```

> A resposta correta não deve ser exposta durante a aplicação.

------------------------------------------------------------------------

## GET `/api/v1/questions/{questionId}`

### Response --- `200 OK`

``` json
{
  "id": "question-uuid",
  "number": 1,
  "area": "MATHEMATICS",
  "statement": "Enunciado da questão...",
  "alternatives": [
    {
      "id": "alternative-a",
      "label": "A",
      "text": "Alternativa A"
    }
  ]
}
```

------------------------------------------------------------------------

# 9. Application Service

Responsável pela inscrição na aplicação, alocação virtual, início,
controle da sessão e finalização da prova.

## POST `/api/v1/applications`

Cria a aplicação do candidato para determinado exame.

### Request

``` json
{
  "candidateId": "candidate-uuid",
  "examId": "exam-uuid",
  "municipalityCode": "2111300"
}
```

### Response --- `201 Created`

``` json
{
  "id": "application-uuid",
  "candidateId": "candidate-uuid",
  "examId": "exam-uuid",
  "status": "REGISTERED",
  "municipalityCode": "2111300",
  "createdAt": "2026-08-30T17:00:00Z"
}
```

------------------------------------------------------------------------

## GET `/api/v1/applications/{applicationId}`

### Response --- `200 OK`

``` json
{
  "id": "application-uuid",
  "candidateId": "candidate-uuid",
  "examId": "exam-uuid",
  "status": "REGISTERED",
  "startedAt": null,
  "finishedAt": null
}
```

------------------------------------------------------------------------

## GET `/api/v1/candidates/{candidateId}/applications`

### Response --- `200 OK`

``` json
{
  "applications": [
    {
      "id": "application-uuid",
      "examId": "exam-uuid",
      "examName": "Dia D ENEM 2026",
      "status": "REGISTERED"
    }
  ]
}
```

------------------------------------------------------------------------

## GET `/api/v1/applications/{applicationId}/allocation`

Retorna a escola e sala virtuais definidas para o candidato.

### Response --- `200 OK`

``` json
{
  "applicationId": "application-uuid",
  "municipality": {
    "code": "2111300",
    "name": "São Luís",
    "state": "MA"
  },
  "school": {
    "id": "school-uuid",
    "name": "Unidade Escolar Virtual 12"
  },
  "room": {
    "id": "room-uuid",
    "number": "204"
  }
}
```

------------------------------------------------------------------------

## POST `/api/v1/applications/{applicationId}/start`

Inicia a aplicação.

### Request

``` json
{
  "startedAt": "2026-11-08T12:00:00-03:00"
}
```

### Response --- `200 OK`

``` json
{
  "applicationId": "application-uuid",
  "status": "IN_PROGRESS",
  "startedAt": "2026-11-08T12:00:00-03:00",
  "scheduledFinishAt": "2026-11-08T17:30:00-03:00"
}
```

------------------------------------------------------------------------

## GET `/api/v1/applications/{applicationId}/session`

### Response --- `200 OK`

``` json
{
  "applicationId": "application-uuid",
  "status": "IN_PROGRESS",
  "startedAt": "2026-11-08T12:00:00-03:00",
  "scheduledFinishAt": "2026-11-08T17:30:00-03:00",
  "remainingSeconds": 14231,
  "answeredQuestions": 42,
  "totalQuestions": 90
}
```

------------------------------------------------------------------------

## POST `/api/v1/applications/{applicationId}/finish`

Finaliza a prova.

### Request

``` json
{
  "finishedAt": "2026-11-08T17:12:38-03:00"
}
```

### Response --- `200 OK`

``` json
{
  "applicationId": "application-uuid",
  "status": "FINISHED",
  "finishedAt": "2026-11-08T17:12:38-03:00",
  "answeredQuestions": 88,
  "unansweredQuestions": 2
}
```

Após a confirmação, o fluxo de correção pode continuar de forma
assíncrona.

------------------------------------------------------------------------

# 10. Answer Service

Responsável por registrar e consultar as respostas do candidato.

## PUT `/api/v1/applications/{applicationId}/answers/{questionId}`

Cria ou substitui a resposta de uma questão.

### Request

``` json
{
  "alternativeId": "alternative-c",
  "answeredAt": "2026-11-08T13:04:12-03:00"
}
```

### Response --- `200 OK`

``` json
{
  "applicationId": "application-uuid",
  "questionId": "question-uuid",
  "alternativeId": "alternative-c",
  "answeredAt": "2026-11-08T13:04:12-03:00"
}
```

------------------------------------------------------------------------

## DELETE `/api/v1/applications/{applicationId}/answers/{questionId}`

Remove uma resposta enquanto a aplicação estiver em andamento.

### Response --- `204 No Content`

------------------------------------------------------------------------

## GET `/api/v1/applications/{applicationId}/answers`

### Response --- `200 OK`

``` json
{
  "applicationId": "application-uuid",
  "answers": [
    {
      "questionId": "question-uuid",
      "alternativeId": "alternative-c",
      "answeredAt": "2026-11-08T13:04:12-03:00"
    }
  ]
}
```

------------------------------------------------------------------------

# 11. Scoring Service

O processamento da correção é assíncrono, mas o serviço disponibiliza
endpoints síncronos de consulta.

## GET `/api/v1/scores/{scoreId}`

### Response --- `200 OK`

``` json
{
  "id": "score-uuid",
  "applicationId": "application-uuid",
  "candidateId": "candidate-uuid",
  "examId": "exam-uuid",
  "status": "FINISHED",
  "finalScore": 782.45,
  "calculatedAt": "2026-11-08T18:02:11Z"
}
```

------------------------------------------------------------------------

## GET `/api/v1/applications/{applicationId}/score`

### Response --- `200 OK`

``` json
{
  "scoreId": "score-uuid",
  "applicationId": "application-uuid",
  "finalScore": 782.45,
  "status": "FINISHED"
}
```

Quando a correção ainda não terminou:

### Response --- `202 Accepted`

``` json
{
  "applicationId": "application-uuid",
  "status": "PROCESSING"
}
```

------------------------------------------------------------------------

# 12. Performance Service

Responsável pela consulta das métricas de desempenho calculadas.

## GET `/api/v1/applications/{applicationId}/performance`

### Response --- `200 OK`

``` json
{
  "id": "performance-uuid",
  "applicationId": "application-uuid",
  "candidateId": "candidate-uuid",
  "overallPercentage": 81.50,
  "percentile": 92.30,
  "areas": [
    {
      "area": "MATHEMATICS",
      "percentage": 86.70
    },
    {
      "area": "LANGUAGES",
      "percentage": 78.40
    }
  ]
}
```

------------------------------------------------------------------------

## GET `/api/v1/candidates/{candidateId}/performances`

### Response --- `200 OK`

``` json
{
  "candidateId": "candidate-uuid",
  "performances": [
    {
      "applicationId": "application-uuid",
      "examName": "Dia D ENEM 2026",
      "score": 782.45,
      "percentile": 92.30
    }
  ]
}
```

------------------------------------------------------------------------

# 13. Ranking Service

Responsável pela consulta da classificação.

## GET `/api/v1/exams/{examId}/ranking`

### Query Parameters

``` text
page=0
size=20
municipalityCode=2111300
```

### Response --- `200 OK`

``` json
{
  "examId": "exam-uuid",
  "content": [
    {
      "position": 1,
      "candidateId": "candidate-1",
      "score": 901.22
    },
    {
      "position": 2,
      "candidateId": "candidate-2",
      "score": 897.14
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 12540
}
```

------------------------------------------------------------------------

## GET `/api/v1/exams/{examId}/ranking/candidates/{candidateId}`

### Response --- `200 OK`

``` json
{
  "examId": "exam-uuid",
  "candidateId": "candidate-uuid",
  "position": 152,
  "score": 782.45,
  "percentile": 92.30
}
```

------------------------------------------------------------------------

# 14. Answer Key Service

Responsável pela consulta do gabarito após sua liberação.

## GET `/api/v1/exams/{examId}/answer-key`

### Response --- `200 OK`

``` json
{
  "examId": "exam-uuid",
  "version": 1,
  "releasedAt": "2026-11-08T18:00:00Z",
  "answers": [
    {
      "questionId": "question-1",
      "correctAlternative": "C"
    },
    {
      "questionId": "question-2",
      "correctAlternative": "A"
    }
  ]
}
```

Antes da publicação:

### Response --- `404 Not Found`

``` json
{
  "status": 404,
  "error": "ANSWER_KEY_NOT_RELEASED",
  "message": "Answer key is not available yet"
}
```

------------------------------------------------------------------------

# 15. Result Service

Responsável por consolidar a visualização síncrona do resultado final.

## GET `/api/v1/applications/{applicationId}/result`

### Response --- `200 OK`

``` json
{
  "applicationId": "application-uuid",
  "candidateId": "candidate-uuid",
  "exam": {
    "id": "exam-uuid",
    "name": "Dia D ENEM 2026"
  },
  "score": {
    "finalScore": 782.45
  },
  "performance": {
    "overallPercentage": 81.50,
    "percentile": 92.30
  },
  "ranking": {
    "position": 152
  },
  "status": "AVAILABLE",
  "availableAt": "2026-11-08T18:15:00Z"
}
```

Se o resultado estiver sendo processado:

### Response --- `202 Accepted`

``` json
{
  "applicationId": "application-uuid",
  "status": "PROCESSING"
}
```

------------------------------------------------------------------------

# 16. Communication Service

O envio de comunicação ocorre de forma assíncrona. A API síncrona é
utilizada para consulta de histórico e preferências.

## GET `/api/v1/candidates/{candidateId}/communications`

### Response --- `200 OK`

``` json
{
  "candidateId": "candidate-uuid",
  "communications": [
    {
      "id": "communication-uuid",
      "type": "RESULT_AVAILABLE",
      "channel": "EMAIL",
      "status": "SENT",
      "sentAt": "2026-11-08T18:16:00Z"
    }
  ]
}
```

------------------------------------------------------------------------

## GET `/api/v1/communications/{communicationId}`

### Response --- `200 OK`

``` json
{
  "id": "communication-uuid",
  "candidateId": "candidate-uuid",
  "type": "RESULT_AVAILABLE",
  "channel": "EMAIL",
  "status": "SENT",
  "sentAt": "2026-11-08T18:16:00Z"
}
```

------------------------------------------------------------------------

# 17. Orchestrator Service

O Orchestrator não deve expor detalhes internos do workflow ao cliente.
Sua API síncrona é voltada principalmente para observabilidade
operacional.

## GET `/api/v1/workflows/{correlationId}`

Endpoint interno/administrativo.

### Response --- `200 OK`

``` json
{
  "correlationId": "3f2f8468-5d16-4af8-b65e-f624a17cc973",
  "workflow": "APPLICATION_POST_PROCESSING",
  "status": "IN_PROGRESS",
  "currentStep": "CALCULATE_PERFORMANCE",
  "steps": [
    {
      "name": "PROCESS_SCORING",
      "status": "FINISHED"
    },
    {
      "name": "CALCULATE_PERFORMANCE",
      "status": "IN_PROGRESS"
    },
    {
      "name": "GENERATE_RANKING",
      "status": "PENDING"
    }
  ]
}
```

------------------------------------------------------------------------

# 18. Audit Service

Responsável pela consulta administrativa da trilha de auditoria.

## GET `/api/v1/audit/events`

### Query Parameters

``` text
correlationId=3f2f8468-5d16-4af8-b65e-f624a17cc973
aggregateId=application-uuid
eventType=ApplicationFinished
```

### Response --- `200 OK`

``` json
{
  "content": [
    {
      "eventId": "event-uuid",
      "eventType": "ApplicationFinished",
      "aggregateId": "application-uuid",
      "correlationId": "3f2f8468-5d16-4af8-b65e-f624a17cc973",
      "occurredAt": "2026-11-08T20:12:38Z"
    }
  ]
}
```

------------------------------------------------------------------------

# 19. Fluxos síncronos entre aplicações

## Cadastro

``` text
Frontend
  -> Auth/Candidate API
  -> Candidate Service
  -> resposta HTTP
```

## Consulta da prova

``` text
Frontend
  -> Exam Service
  -> Question Service
  -> resposta HTTP
```

## Início da aplicação

``` text
Frontend
  -> Application Service
  -> valida candidato
  -> valida exame
  -> recupera alocação
  -> inicia sessão
  -> resposta HTTP
```

## Resposta de questão

``` text
Frontend
  -> Answer Service
  -> valida aplicação
  -> registra resposta
  -> resposta HTTP
```

## Finalização

``` text
Frontend
  -> Application Service
  -> finaliza aplicação
  -> resposta HTTP

Após a confirmação HTTP:
ApplicationFinished
  -> fluxo assíncrono
```

## Consulta do resultado

``` text
Frontend
  -> Result Service
     -> Scoring Service
     -> Performance Service
     -> Ranking Service
  -> resultado consolidado
```

------------------------------------------------------------------------

# 20. Códigos HTTP

  Código                        Uso
  ----------------------------- ---------------------------------------------
  `200 OK`                      Consulta ou operação concluída
  `201 Created`                 Recurso criado
  `202 Accepted`                Processamento assíncrono ainda em andamento
  `204 No Content`              Operação concluída sem body
  `400 Bad Request`             Request malformado
  `401 Unauthorized`            Usuário não autenticado
  `403 Forbidden`               Usuário sem permissão
  `404 Not Found`               Recurso inexistente
  `409 Conflict`                Conflito de estado ou duplicidade
  `422 Unprocessable Entity`    Violação de regra ou validação
  `500 Internal Server Error`   Falha interna
  `503 Service Unavailable`     Dependência indisponível

------------------------------------------------------------------------

# 21. Resumo dos endpoints

## Auth Service

``` text
POST /api/v1/auth/login
POST /api/v1/auth/refresh
GET  /api/v1/auth/me
```

## Candidate Service

``` text
POST  /api/v1/candidates
GET   /api/v1/candidates/{candidateId}
PATCH /api/v1/candidates/{candidateId}
```

## Exam Service

``` text
GET  /api/v1/exams
GET  /api/v1/exams/{examId}
POST /api/v1/exams
```

## Question Service

``` text
GET /api/v1/exams/{examId}/questions
GET /api/v1/questions/{questionId}
```

## Application Service

``` text
POST /api/v1/applications
GET  /api/v1/applications/{applicationId}
GET  /api/v1/candidates/{candidateId}/applications
GET  /api/v1/applications/{applicationId}/allocation
POST /api/v1/applications/{applicationId}/start
GET  /api/v1/applications/{applicationId}/session
POST /api/v1/applications/{applicationId}/finish
```

## Answer Service

``` text
PUT    /api/v1/applications/{applicationId}/answers/{questionId}
DELETE /api/v1/applications/{applicationId}/answers/{questionId}
GET    /api/v1/applications/{applicationId}/answers
```

## Scoring Service

``` text
GET /api/v1/scores/{scoreId}
GET /api/v1/applications/{applicationId}/score
```

## Performance Service

``` text
GET /api/v1/applications/{applicationId}/performance
GET /api/v1/candidates/{candidateId}/performances
```

## Ranking Service

``` text
GET /api/v1/exams/{examId}/ranking
GET /api/v1/exams/{examId}/ranking/candidates/{candidateId}
```

## Answer Key Service

``` text
GET /api/v1/exams/{examId}/answer-key
```

## Result Service

``` text
GET /api/v1/applications/{applicationId}/result
```

## Communication Service

``` text
GET /api/v1/candidates/{candidateId}/communications
GET /api/v1/communications/{communicationId}
```

## Orchestrator Service

``` text
GET /api/v1/workflows/{correlationId}
```

## Audit Service

``` text
GET /api/v1/audit/events
```

------------------------------------------------------------------------

# Conclusão

As APIs REST/HTTP representam a camada de interação síncrona do **Dia D
Simulation**.

A regra arquitetural é manter no HTTP apenas operações que necessitam de
resposta imediata. Etapas pesadas ou encadeadas do pós-prova são
iniciadas após a operação síncrona e continuam através da arquitetura
assíncrona, evitando bloquear requisições e mantendo baixo acoplamento
entre os serviços.
