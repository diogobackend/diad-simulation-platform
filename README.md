# Dia D Simulation Platform

## Sumário

1. [Sobre](#1---sobre-o-projeto-dia-d-simulation-platform)
2. [Repositórios](#2---repositórios)
3. [O que este produto resolve?](#3---o-que-este-produto-resolve)
4. [Exemplo prático](#4---exemplo-prático)
5. [Fluxo resumido](#5---fluxo-resumido)
6. [Objetivo técnico](#6---objetivo-técnico)
7. [Microservices](#7---microservices)
8. [Bibliotecas e repositórios de apoio](#8---bibliotecas-e-repositórios-de-apoio)
9. [Responsabilidade dos serviços](#9---responsabilidade-dos-serviços)
10. [Arquitetura Hexagonal](#10---arquitetura-hexagonal)
11. [Comunicação entre serviços](#11---comunicação-entre-serviços)
12. [Autenticação e autorização](#12---autenticação-e-autorização)
13. [Banco de dados](#13---banco-de-dados)
14. [Regras de aplicação de provas](#14---regras-de-aplicação-de-provas)
15. [Padrões aplicados](#15---padrões-aplicados)
16. [Observabilidade](#16---observabilidade)
17. [Tecnologias](#17---tecnologias)
18. [Modelo de negócio](#18---modelo-de-negócio)
19. [Roadmap de desenvolvimento](#19---roadmap-de-desenvolvimento)
20. [Status](#20---status)

---

## 1 - Sobre

É uma plataforma digital de simulação realista de provas, exames, vestibulares, concursos e processos seletivos.

O objetivo do produto não é apenas disponibilizar questões para estudo.

A proposta é reproduzir digitalmente, da forma mais fiel possível, a experiência do dia real de uma prova.

O candidato poderá:

- realizar inscrição em uma aplicação;
- escolher município, quando a prova exigir localização;
- receber um local e uma sala virtual;
- aguardar o dia e horário oficial da aplicação;
- entrar em uma sessão de prova controlada;
- realizar a prova com cronômetro sincronizado pelo servidor;
- responder exatamente à quantidade e ao formato de questões definidos para aquele exame;
- cumprir regras de permanência, saída e finalização;
- ter ações de segurança registradas durante a sessão;
- receber resultado, classificação e ranking;
- acompanhar seu histórico de desempenho.

A plataforma será construída para suportar diferentes tipos de prova sem acoplar o sistema exclusivamente ao ENEM.

Exemplos de provas que poderão ser configuradas:

```text
ENEM
Vestibulares
Concursos públicos
OAB
Residência médica
Certificações profissionais
Processos seletivos
Provas institucionais
```

O ENEM poderá ser utilizado como um dos primeiros casos de uso por possuir regras de aplicação bem definidas e uma experiência de prova amplamente conhecida.

---

## 2 - Repositórios

| Repositório | Responsabilidade |
|---|---|
| `diad-simulation-platform` | Repositório central de documentação, arquitetura, roadmap e visão geral |
| `diad-auth-service` | Login, autenticação, JWT, refresh token, roles e permissões |
| `diad-api-gateway` | Entrada única, roteamento, segurança e filtros técnicos |
| `diad-candidate-service` | Cadastro, perfil, dados pessoais e histórico básico do candidato |
| `diad-exam-service` | Cadastro e configuração dos tipos, edições, estruturas e regras das provas |
| `diad-question-service` | Banco de questões, disciplinas, áreas, alternativas e metadados |
| `diad-application-service` | Coordena a aplicação real da prova, horários, sessão, sala e regras de permanência |
| `diad-answer-service` | Recebimento, persistência e controle das respostas |
| `diad-scoring-service` | Correção e cálculo de notas conforme a política de cada exame |
| `diad-ranking-service` | Rankings, classificações e posições por recorte |
| `diad-security-service` | Eventos de segurança, troca de aba, múltiplas sessões e comportamento suspeito |
| `diad-communication-service` | E-mail, notificações, lembretes, retry e DLQ |
| `diad-config-server` | Configuração centralizada dos serviços |
| `diad-shared-contracts` | Contratos OpenAPI, AsyncAPI e schemas de eventos |
| `diad-observability-starter` | Biblioteca compartilhada para logs, tracing, correlationId e métricas |
| `diad-platform-infra` | Docker Compose, Kubernetes, bancos, mensageria e observabilidade |
| `diad-web` | Frontend React da plataforma |

---

## 3 - O que este produto resolve?

Plataformas tradicionais de simulados normalmente concentram a experiência em:

- banco de questões;
- resolução livre;
- cronômetro local;
- correção;
- estatísticas de desempenho.

A **Dia D Simulation Platform** adiciona uma camada de experiência operacional.

O sistema deve simular o contexto em que uma prova realmente acontece.

Isso inclui:

- inscrição em uma aplicação específica;
- horário oficial de abertura;
- horário limite para início;
- duração real da prova;
- cronômetro controlado pelo backend;
- regras de permanência mínima;
- regras de saída;
- local e sala virtual;
- sessão única;
- registro de troca de aba;
- registro de perda de foco;
- controle de múltiplos logins;
- encerramento automático;
- envio definitivo da prova;
- cálculo de resultado;
- classificação;
- ranking;
- histórico de aplicações.

O sistema também busca resolver um problema arquitetural:

```text
Como criar um motor de aplicação de provas suficientemente genérico para suportar exames com regras muito diferentes sem espalhar if/else por toda a plataforma?
```

A resposta será baseada em:

- domínio genérico de provas;
- políticas configuráveis;
- Strategy Pattern;
- microservices por contexto;
- arquitetura hexagonal;
- eventos de domínio;
- separação entre regras de aplicação e regras de pontuação.

---

## 4 - Exemplo prático

Imagine o seguinte cenário.

Diogo deseja participar de uma simulação do ENEM.

1. Diogo cria uma conta.
2. O `diad-auth-service` autentica o usuário e gera um token JWT.
3. O `diad-candidate-service` mantém seu perfil de candidato.
4. Diogo acessa uma edição disponível do ENEM.
5. O `diad-exam-service` fornece a estrutura e as regras daquela edição.
6. Diogo realiza a inscrição.
7. O `diad-application-service` cria sua participação.
8. O sistema associa município, local virtual e sala.
9. No dia da prova, Diogo entra na sala de espera.
10. O servidor libera a prova no horário configurado.
11. O `diad-question-service` fornece as questões previamente associadas à prova.
12. O `diad-answer-service` persiste as respostas durante a aplicação.
13. O `diad-security-service` registra eventos de segurança.
14. Quando o tempo termina ou o candidato envia a prova, a aplicação é encerrada.
15. O `diad-scoring-service` calcula o resultado.
16. O `diad-ranking-service` atualiza as classificações.
17. O `diad-communication-service` pode notificar o candidato sobre a liberação do resultado.

Exemplo:

```text
Candidato: Diogo Ferreira
Prova: ENEM - Simulação Nacional #01
Município: São Luís - MA
Local virtual: Escola X
Sala virtual: 07

Status: FINISHED
Nota calculada: 712.40
Ranking nacional: 1.842
Ranking estadual: 73
Ranking municipal: 21
Ranking do curso escolhido: 193
```

Outro exemplo:

```text
Candidato: Maria Silva
Prova: Concurso Público - Banca Cebraspe
Cargo: Analista de Tecnologia

Questões corretas: 92
Questões incorretas: 21
Questões em branco: 7
Pontuação final: calculada conforme política configurada
Status: FINISHED
```

A plataforma não deve assumir que todas as provas utilizam a mesma forma de pontuação.

---

## 5 - Fluxo resumido

```text
Candidato / Frontend React
          |
          v
   diad-api-gateway
          |
          +----------------------+
          |                      |
          v                      v
 diad-auth-service      diad-candidate-service
          |
          v
   diad-exam-service
          |
          v
diad-application-service
          |
          +----------------------+
          |                      |
          v                      v
diad-question-service   diad-security-service
          |
          v
 diad-answer-service
          |
          v
       Kafka
          |
          +----------------------+
          |                      |
          v                      v
diad-scoring-service    diad-ranking-service
          |
          v
diad-communication-service
```

Fluxo simplificado da aplicação:

```text
Inscrição
   |
   v
Sala virtual
   |
   v
Aguardando horário
   |
   v
Prova liberada
   |
   v
Sessão ativa
   |
   +--> respostas
   |
   +--> eventos de segurança
   |
   v
Prova finalizada
   |
   v
Correção
   |
   v
Ranking
   |
   v
Resultado
```

---

## 6 - Objetivo técnico

O projeto será desenvolvido como uma plataforma SaaS com foco em escalabilidade, extensibilidade e proximidade com ambientes reais de produção.

Principais objetivos técnicos:

- utilizar Kotlin com Java 21;
- aplicar Spring Boot;
- utilizar Arquitetura Hexagonal / Ports and Adapters;
- manter domínio independente de frameworks;
- construir microservices independentes;
- aplicar database per service;
- utilizar API Gateway;
- centralizar autenticação;
- utilizar JWT;
- aplicar Spring Security;
- utilizar comunicação síncrona e assíncrona;
- aplicar Kafka para eventos de domínio;
- utilizar RabbitMQ para filas de trabalho, retry e DLQ;
- aplicar Outbox Pattern;
- aplicar Inbox Pattern;
- garantir idempotência;
- aplicar Feature Toggles;
- utilizar Flyway;
- expor OpenAPI;
- documentar eventos com AsyncAPI;
- utilizar correlationId;
- gerar logs estruturados;
- utilizar OpenTelemetry;
- expor métricas para Prometheus;
- criar dashboards no Grafana;
- preparar os serviços para Docker;
- preparar os serviços para Kubernetes;
- permitir escalabilidade horizontal;
- manter regras específicas de cada prova desacopladas do núcleo;
- desenvolver backend e frontend de forma incremental por funcionalidade.

---

## 7 - Microservices

| Serviço | Responsabilidade |
|---|---|
| `diad-auth-service` | Autenticação, usuários, JWT, refresh token, roles e permissões |
| `diad-api-gateway` | Entrada única, roteamento, token, CORS, rate limit e correlationId |
| `diad-candidate-service` | Cadastro e perfil do candidato |
| `diad-exam-service` | Estrutura, edição, configuração e políticas das provas |
| `diad-question-service` | Questões, disciplinas, alternativas, origem e metadados |
| `diad-application-service` | Aplicação em tempo real, sessão, cronômetro, sala e regras operacionais |
| `diad-answer-service` | Registro e persistência de respostas |
| `diad-scoring-service` | Cálculo de nota e políticas de correção |
| `diad-ranking-service` | Ranking e classificação |
| `diad-security-service` | Eventos antifraude e integridade da sessão |
| `diad-communication-service` | Comunicações assíncronas |
| `diad-config-server` | Configuração centralizada |

---

## 8 - Bibliotecas e repositórios de apoio

Além dos microservices, a plataforma possui repositórios de apoio.

| Repositório | Responsabilidade |
|---|---|
| `diad-platform-infra` | Docker Compose, Kubernetes, Kafka, RabbitMQ, Redis, bancos, Prometheus e Grafana |
| `diad-shared-contracts` | OpenAPI, AsyncAPI, schemas de eventos e exemplos de payload |
| `diad-observability-starter` | Logs estruturados, correlationId, tracing, MDC e métricas |
| `diad-web` | Aplicação React utilizada pelos candidatos, escolas e administradores |

### diad-observability-starter

Não é um microservice.

É uma biblioteca compartilhada para evitar duplicação de código técnico.

Responsabilidades planejadas:

- annotation `@LogInfo`;
- annotation `@LogParameter`;
- logs automáticos com AOP;
- propagação de correlationId;
- configuração de MDC;
- mascaramento de informações sensíveis;
- padronização dos campos de log;
- integração com OpenTelemetry;
- integração com métricas.

Exemplo:

```kotlin
@LogInfo(logParameters = true, logReturn = true)
fun startApplication(input: StartApplicationInput): ApplicationSession
```

---

## 9 - Responsabilidade dos serviços

### diad-auth-service

Responsável pela autenticação e autorização.

Responsabilidades:

- cadastrar usuário;
- autenticar usuário;
- validar senha;
- gerar JWT;
- gerar refresh token;
- controlar roles;
- controlar permissões;
- invalidar sessão quando necessário.

Exemplo:

```text
POST /api/v1/auth/login
POST /api/v1/auth/refresh
```

Roles iniciais possíveis:

```text
CANDIDATE
SCHOOL_ADMIN
PLATFORM_ADMIN
```

---

### diad-api-gateway

Responsável pela entrada técnica da plataforma.

Responsabilidades:

- receber requisições externas;
- validar JWT;
- aplicar filtros técnicos;
- controlar CORS;
- aplicar rate limit;
- propagar correlationId;
- rotear chamadas.

O gateway não deve conter regra de negócio.

---

### diad-candidate-service

Responsável pelo perfil do candidato.

Responsabilidades:

- criar candidato;
- consultar candidato;
- atualizar perfil;
- armazenar município;
- armazenar preferências;
- manter dados básicos de participação;
- vincular candidato a uma escola quando aplicável.

Eventos possíveis:

```text
CandidateCreated
CandidateUpdated
CandidateSchoolLinked
```

---

### diad-exam-service

Responsável pela definição estrutural das provas.

Conceitos principais:

```text
ExamType
ExamEdition
ExamSection
ExamDay
QuestionStructure
ApplicationPolicy
ScoringPolicy
RankingPolicy
```

Responsabilidades:

- cadastrar tipo de prova;
- cadastrar edição;
- definir quantidade de dias;
- definir seções;
- definir duração;
- definir regras de aplicação;
- definir regras de pontuação;
- definir quantidade de questões;
- definir tipo de questão;
- definir pesos;
- definir critérios de eliminação;
- ativar/desativar edição.

Exemplo:

```text
ExamType: ENEM
Edition: Simulation 2026 #01
Days: 2
ScoringPolicy: TRI
```

---

### diad-question-service

Responsável pelo banco de questões.

Responsabilidades:

- cadastrar questão;
- consultar questão;
- importar questões;
- definir área;
- definir disciplina;
- definir assunto;
- definir alternativas;
- definir resposta correta;
- registrar ano;
- registrar banca;
- registrar fonte;
- registrar dificuldade;
- selecionar questões conforme critérios.

Tipos possíveis:

```text
MULTIPLE_CHOICE
TRUE_FALSE
DISCURSIVE
ESSAY
PRACTICAL
```

---

### diad-application-service

É o serviço central da experiência de aplicação.

Responsabilidades:

- realizar inscrição em aplicação;
- associar candidato à aplicação;
- atribuir sala virtual;
- atribuir local virtual;
- manter status da participação;
- controlar horário de início;
- controlar horário limite;
- controlar duração;
- controlar permanência mínima;
- abrir sessão;
- manter horário oficial do servidor;
- impedir início fora da janela;
- finalizar automaticamente;
- aplicar regras específicas da prova;
- receber estado de segurança quando necessário.

Conceitos importantes:

```text
Application
ApplicationSession
VirtualLocation
VirtualRoom
ApplicationStatus
ApplicationRule
```

Status possíveis:

```text
REGISTERED
WAITING
AVAILABLE
IN_PROGRESS
FINISHED
EXPIRED
CANCELLED
DISQUALIFIED
```

---

### diad-answer-service

Responsável pelas respostas do candidato.

Responsabilidades:

- registrar resposta;
- alterar resposta durante a sessão;
- garantir idempotência;
- registrar data/hora;
- associar resposta à sessão;
- bloquear alteração após finalização;
- consolidar respostas da prova.

Eventos:

```text
AnswerSubmitted
AnswerChanged
ExamAnswersFinalized
```

---

### diad-security-service

Responsável pela integridade da sessão.

Eventos possíveis:

```text
TabChanged
WindowBlurred
FullscreenExited
CopyAttempted
PasteAttempted
MultipleLoginDetected
ConnectionLost
IpChanged
SessionDuplicated
```

Responsabilidades:

- registrar eventos de segurança;
- calcular nível de risco;
- aplicar regras configuráveis;
- sinalizar sessão suspeita;
- recomendar bloqueio;
- armazenar histórico.

O serviço não deve assumir que qualquer evento representa automaticamente fraude.

As regras devem ser configuráveis por tipo de prova.

---

### diad-scoring-service

Responsável por calcular resultados.

O serviço deve suportar estratégias diferentes.

Exemplos:

```text
SimpleScore
WeightedScore
NegativeScore
CebraspeScore
TriScore
PercentageScore
CustomScore
```

O código não deve possuir estruturas semelhantes a:

```kotlin
if (examType == "ENEM") {
    ...
} else if (examType == "CEBRASPE") {
    ...
}
```

A implementação deve utilizar políticas ou estratégias.

Exemplo conceitual:

```text
ScoringPolicy
    |
    +--> SimpleScoringStrategy
    +--> WeightedScoringStrategy
    +--> NegativeScoringStrategy
    +--> TriScoringStrategy
```

Eventos:

```text
ScoreCalculationRequested
ScoreCalculated
ScoreCalculationFailed
```

---

### diad-ranking-service

Responsável pelas classificações.

Possíveis rankings:

```text
GLOBAL
COUNTRY
STATE
CITY
SCHOOL
CLASS
EXAM
COURSE
POSITION
CATEGORY
```

Responsabilidades:

- atualizar ranking;
- calcular posição;
- consultar posição;
- gerar ranking por escola;
- gerar ranking por município;
- gerar ranking por estado;
- gerar ranking por categoria;
- manter snapshots quando necessário.

Redis poderá ser utilizado para rankings de leitura frequente.

---

### diad-communication-service

Responsável pelas comunicações.

Exemplos:

- inscrição confirmada;
- lembrete de prova;
- cartão de confirmação;
- aviso de início;
- resultado disponível;
- comunicação com escola.

Pode utilizar:

```text
E-mail
WhatsApp
Push
SMS
```

Integrações externas devem ser abstraídas por portas.

Exemplo:

```text
NotificationProviderPort
    |
    +--> EmailProviderAdapter
    +--> WhatsAppProviderAdapter
```

Esse serviço é candidato ao uso de RabbitMQ.

---

## 10 - Arquitetura Hexagonal

Cada microservice deve seguir Arquitetura Hexagonal.

Estrutura padrão:

```text
src/main/kotlin/com/diadsimulation/{service}/
├── core/
│   ├── common/
│   ├── domain/
│   │   ├── model/
│   │   ├── exception/
│   │   └── valueobject/
│   ├── port/
│   │   ├── input/
│   │   └── output/
│   └── usecase/
└── app/
    ├── adapter/
    │   ├── input/
    │   │   ├── web/
    │   │   └── messaging/
    │   └── output/
    │       ├── persistence/
    │       ├── messaging/
    │       └── client/
    └── configuration/
```

Regras arquiteturais:

- controller não acessa repository diretamente;
- consumer não executa regra de negócio diretamente;
- repository não conhece use case;
- DTO não entra no domínio;
- entidade JPA não é modelo de domínio;
- domínio não depende de Spring;
- domínio não conhece Kafka;
- domínio não conhece RabbitMQ;
- domínio não conhece HTTP;
- toda entrada passa por uma porta de entrada;
- toda saída passa por uma porta de saída;
- integrações externas devem ser abstraídas;
- use cases dependem de portas, não de adapters.

---

## 11 - Comunicação entre serviços

A plataforma utilizará três formas principais de comunicação.

### REST

Usado para operações síncronas.

Exemplos:

```text
GET /api/v1/candidates/{candidateId}
GET /api/v1/exams/{examId}
GET /api/v1/applications/{applicationId}
GET /api/v1/rankings/{rankingId}
POST /api/v1/auth/login
```

### Kafka

Usado para eventos de domínio.

Exemplos:

```text
CandidateCreated
ExamPublished
ApplicationRegistered
ApplicationStarted
AnswerSubmitted
SecurityViolationDetected
ApplicationFinished
ScoreCalculated
RankingUpdated
```

### RabbitMQ

Usado para filas de trabalho.

Exemplos:

```text
SendRegistrationEmail
SendExamReminder
ProcessNotification
RetryNotification
ReprocessFailedJob
```

RabbitMQ deve ser utilizado quando for necessário:

- retry;
- DLQ;
- controle de tentativa;
- fila de trabalho;
- reprocessamento.

---

## 12 - Autenticação e autorização

A autenticação será centralizada no `diad-auth-service`.

Fluxo:

```text
1. Usuário realiza login
2. Auth Service valida as credenciais
3. Auth Service gera JWT
4. Frontend envia JWT nas chamadas seguintes
5. API Gateway valida o token
6. Gateway propaga contexto do usuário
7. Requisição segue para o microservice
```

Exemplo:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

Payload possível:

```json
{
  "sub": "user-id",
  "candidateId": "candidate-id",
  "role": "CANDIDATE"
}
```

Para usuário institucional:

```json
{
  "sub": "user-id",
  "schoolId": "school-id",
  "role": "SCHOOL_ADMIN"
}
```

---

## 13 - Banco de dados

A plataforma utilizará `database per service`.

Possível distribuição:

| Serviço | Banco |
|---|---|
| `diad-auth-service` | `auth_db` |
| `diad-candidate-service` | `candidate_db` |
| `diad-exam-service` | `exam_db` |
| `diad-question-service` | `question_db` |
| `diad-application-service` | `application_db` |
| `diad-answer-service` | `answer_db` |
| `diad-scoring-service` | `scoring_db` |
| `diad-ranking-service` | `ranking_db` |
| `diad-security-service` | `security_db` |
| `diad-communication-service` | `communication_db` |

A tecnologia inicial prevista é PostgreSQL.

Redis poderá ser utilizado para:

- cache;
- sessões temporárias;
- rate limiting;
- locks distribuídos;
- dados de leitura rápida;
- ranking em tempo real;
- informações temporárias de aplicação.

Regras:

- serviço não acessa diretamente banco de outro serviço;
- alterações devem usar Flyway;
- schema deve ser versionado;
- integrações entre serviços devem utilizar API ou mensageria.

---

## 14 - Regras de aplicação de provas

As regras não devem ser fixadas diretamente no código para um exame específico.

A plataforma deve possuir conceitos genéricos.

Exemplo:

```text
ApplicationPolicy
├── StartPolicy
├── EndPolicy
├── MinimumStayPolicy
├── ExitPolicy
├── SecurityPolicy
├── SessionPolicy
└── SubmissionPolicy
```

Exemplo ENEM:

```text
StartTime: configurado
EndTime: configurado
MinimumStay: configurado
QuestionFormat: MULTIPLE_CHOICE
Days: 2
Essay: enabled
Scoring: TRI
```

Exemplo Cebraspe:

```text
QuestionFormat: TRUE_FALSE
NegativeScoring: enabled
Duration: configurada
MinimumStay: configurada
```

Exemplo OAB:

```text
Phase 1:
  QuestionFormat: MULTIPLE_CHOICE

Phase 2:
  QuestionFormat: DISCURSIVE / PRACTICAL
```

A regra deve ser associada ao exame ou à edição.

O objetivo é permitir adicionar novos exames sem alterar o núcleo da plataforma.

---

## 15 - Padrões aplicados

- Arquitetura Hexagonal;
- Ports and Adapters;
- Microservices;
- Event-driven architecture;
- Database per service;
- API Gateway;
- Use Cases;
- DTOs nas bordas;
- mappers explícitos;
- entidades JPA separadas de domínio;
- Constructor Injection;
- Strategy Pattern;
- Policy Pattern;
- Factory quando necessário;
- Exceptions específicas;
- Spring Security;
- JWT;
- OpenAPI;
- AsyncAPI;
- Kafka;
- RabbitMQ;
- Redis;
- idempotência;
- Outbox Pattern;
- Inbox Pattern;
- Retry;
- DLQ;
- Feature Toggles;
- Correlation ID;
- logs estruturados;
- tracing distribuído;
- migrations com Flyway;
- cache;
- locks distribuídos quando necessário.

---

## 16 - Observabilidade

A plataforma deve permitir rastrear uma aplicação completa entre múltiplos serviços.

Exemplo de correlationId:

```text
correlationId=8b70f8d1-14e2-45cf-a183-f2196ef4b821
```

O correlationId deve ser propagado entre:

- React;
- API Gateway;
- microservices;
- Kafka;
- RabbitMQ;
- logs;
- tracing;
- auditoria técnica.

### Logs

Campos esperados:

```text
serviceName
operation
correlationId
traceId
spanId
userId
candidateId
examId
applicationId
sessionId
eventId
status
duration
error
```

Dados sensíveis devem ser mascarados.

### Métricas

Métricas planejadas:

```text
requests_total
request_duration_seconds
errors_total

active_exam_sessions
exam_sessions_started_total
exam_sessions_finished_total
exam_sessions_expired_total

answers_submitted_total
answers_failed_total

security_events_total
tab_change_events_total
multiple_login_events_total

score_calculations_total
score_calculation_failed_total

ranking_updates_total

messages_consumed_total
messages_failed_total
messages_dlq_total

auth_login_success_total
auth_login_failed_total
```

### Dashboards

Grafana poderá exibir:

- saúde dos serviços;
- candidatos conectados;
- sessões ativas;
- aplicações em andamento;
- quantidade de respostas por segundo;
- latência;
- taxa de erro;
- backlog Kafka;
- backlog RabbitMQ;
- DLQ;
- eventos de segurança;
- tempo de cálculo de notas;
- volume de rankings processados;
- falhas de autenticação.

---

## 17 - Tecnologias

Backend:

```text
Kotlin
Java 21
Spring Boot
Spring Web
Spring Security
Spring Data JPA
Spring Cloud Gateway
Spring Cloud Config
Spring Cloud Stream
```

Frontend:

```text
React
TypeScript
Vite
React Router
TanStack Query
Zustand
```

Dados e infraestrutura:

```text
PostgreSQL
Redis
Flyway
Kafka
RabbitMQ
Docker
Kubernetes
```

Observabilidade:

```text
OpenTelemetry
Prometheus
Grafana
Logs estruturados
MDC
Correlation ID
```

Contratos:

```text
OpenAPI
AsyncAPI
```

---

## 18 - Modelo de negócio

A plataforma será estruturada como SaaS.

### B2C

Usuários individuais poderão assinar diretamente a plataforma.

Possíveis recursos:

- acesso a aplicações nacionais;
- histórico;
- ranking;
- desempenho;
- estatísticas;
- comparações;
- evolução.

### B2B

Escolas, cursinhos e instituições poderão contratar pacotes por quantidade de alunos.

Possíveis recursos institucionais:

- gestão de alunos;
- turmas;
- aplicações privadas;
- ranking interno;
- ranking nacional;
- relatórios;
- desempenho por turma;
- desempenho por disciplina;
- acompanhamento de evolução;
- integração via API;
- white-label futuramente.

Possíveis modelos:

```text
assinatura individual
assinatura institucional
plano por quantidade de alunos
plano por quantidade de aplicações
contrato anual
white-label
API B2B
```

O aluno institucional deve preferencialmente consumir a licença fornecida pela própria escola, evitando cobrança duplicada.

---

## 19 - Roadmap de desenvolvimento

O desenvolvimento será incremental.

Backend e frontend deverão evoluir por funcionalidades verticais.

Ordem inicial sugerida:

```text
1. diad-candidate-service
2. cadastro básico no diad-web

3. diad-auth-service
4. login no diad-web

5. diad-exam-service
6. listagem e detalhes de provas no diad-web

7. diad-application-service
8. inscrição em prova no diad-web

9. local virtual e sala virtual
10. cartão de confirmação no diad-web

11. diad-question-service
12. estrutura de questões

13. diad-answer-service
14. tela de prova no diad-web

15. cronômetro sincronizado pelo backend
16. regras de início, permanência e finalização

17. diad-security-service
18. eventos de integridade da sessão

19. diad-scoring-service
20. resultado no diad-web

21. diad-ranking-service
22. ranking no diad-web

23. diad-communication-service

24. Kafka
25. RabbitMQ
26. Outbox
27. Inbox
28. retry e DLQ

29. diad-observability-starter
30. métricas e tracing

31. diad-api-gateway
32. diad-config-server

33. diad-platform-infra
34. Docker Compose
35. Kubernetes

36. testes de carga
37. testes de concorrência
38. hardening de segurança
39. preparação para MVP público
```

O primeiro caso de uso completo poderá ser uma simulação de ENEM.

Depois, deverá ser implementado um segundo tipo de prova com regras significativamente diferentes.

Isso servirá para validar se o motor de provas realmente está desacoplado do ENEM.

---

## 20 - Status

Projeto em desenvolvimento.

Status inicial:

```text
diad-simulation-platform: criado
README.md: em construção
docs/: a ser criado
microservices: ainda não iniciados
frontend: ainda não iniciado
```

Próximos passos:

```text
1. finalizar documentação central
2. criar docs/architecture.md
3. criar docs/services.md
4. criar docs/database.md
5. criar docs/messaging.md
6. criar docs/observability.md
7. criar docs/deployment.md
8. criar docs/security.md
9. criar docs/exam-rules.md
10. criar docs/business-model.md
11. criar docs/roadmap_de_desenvolvimento.md
12. iniciar diad-candidate-service
```

---

## Documentação complementar planejada

```text
docs/
├── architecture.md
├── services.md
├── database.md
├── messaging.md
├── observability.md
├── deployment.md
├── security.md
├── exam-rules.md
├── business-model.md
└── roadmap_de_desenvolvimento.md
```

Cada documento deverá aprofundar um contexto específico sem transformar o README em documentação operacional excessivamente detalhada.

---

## Princípios do projeto

A plataforma deve manter os seguintes princípios ao longo da evolução:

```text
domínio independente de framework
baixo acoplamento
responsabilidade única por contexto
contratos explícitos
idempotência
observabilidade desde o início
segurança por padrão
escalabilidade horizontal
regras configuráveis
backend como autoridade da prova
frontend sem regra crítica de negócio
evolução incremental
```

O relógio do dispositivo do candidato nunca deve ser considerado autoridade para início ou fim de uma aplicação.

O backend será responsável pelo tempo oficial da sessão.

```text
serverTime
applicationStartsAt
applicationEndsAt
sessionExpiresAt
```

A interface React apenas representa o tempo calculado pelo servidor.

---

**Dia D Simulation Platform**

> Viva o dia antes dele chegar.
