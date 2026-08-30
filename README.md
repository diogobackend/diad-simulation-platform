# Dia D Simulation Platform

## Sumário

1. [Sobre](#1---sobre)
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
15. [Gabarito e acesso pós-prova](#15---gabarito-e-acesso-pós-prova)
16. [Relatórios de desempenho](#16---relatórios-de-desempenho)
17. [Padrões aplicados](#17---padrões-aplicados)
18. [Observabilidade](#18---observabilidade)
19. [Tecnologias](#19---tecnologias)
20. [Modelo de negócio](#20---modelo-de-negócio)
21. [Roadmap de desenvolvimento](#21---roadmap-de-desenvolvimento)
22. [Status](#22---status)

---

## 1 - Sobre

É uma plataforma digital de simulação realista de provas, exames, vestibulares, concursos e processos seletivos.

O objetivo não é apenas disponibilizar questões para estudo.

A proposta é reproduzir digitalmente, da forma mais fiel possível, a experiência do dia real de uma prova.

O candidato poderá:

- Realizar inscrição em uma aplicação;
- Escolher município, quando aplicável;
- Receber local e sala virtual;
- Aguardar o dia e horário definidos;
- Entrar em uma sessão controlada;
- Realizar a prova com cronômetro sincronizado pelo servidor;
- Cumprir regras de permanência, saída e finalização;
- Responder exatamente à estrutura definida para aquele exame;
- Ter eventos de segurança registrados durante a sessão;
- Receber resultado, classificação e ranking;
- Acessar o gabarito quando cumprir as regras de permanência definidas;
- Consultar relatórios individuais de desempenho;
- Identificar matérias, assuntos e tipos de questão em que mais errou;
- Receber recomendações de revisão e dicas de estudo;
- Acompanhar sua evolução entre diferentes aplicações.

A plataforma será construída para suportar diferentes tipos de provas, seletivos e concursos, adaptada a cada cenário específico. Como base inicial, iremos trabalhar com o ENEM.

Exemplos:

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

---

## 2 - Repositórios

| Repositório | Responsabilidade |
|---|---|
| `diad-auth-service` | Login, autenticação, JWT, refresh token, roles e permissões |
| `diad-api-gateway` | Entrada única, roteamento, segurança e filtros técnicos |
| `diad-candidate-service` | Cadastro, perfil e dados do candidato |
| `diad-exam-service` | Tipos de prova, edições, estruturas e políticas |
| `diad-question-service` | Banco de questões, disciplinas, assuntos e alternativas |
| `diad-application-service` | Aplicação da prova, horários, sessão, sala e regras operacionais |
| `diad-answer-service` | Recebimento e persistência das respostas |
| `diad-orchestrator-service` | Coordenação de workflows distribuídos entre microservices |
| `diad-scoring-service` | Correção e cálculo de notas |
| `diad-performance-service` | Relatórios de desempenho, evolução e recomendações |
| `diad-ranking-service` | Rankings e classificações |
| `diad-security-service` | Eventos de segurança e integridade da sessão |
| `diad-communication-service` | E-mail, WhatsApp, notificações, lembretes, retry e DLQ |
| `diad-config-server` | Configuração centralizada |
| `diad-shared-contracts` | OpenAPI, AsyncAPI e schemas de eventos |
| `diad-observability-starter` | Logs, tracing, correlationId e métricas |
| `diad-platform-infra` | Docker, Kubernetes, bancos, mensageria e observabilidade |
| `diad-web` | Frontend React |

---

## 3 - O que este produto resolve?

Plataformas tradicionais de simulado normalmente concentram a experiência em:

- banco de questões;
- resolução livre;
- cronômetro;
- correção;
- estatísticas básicas.

A **Dia D Simulation Platform** adiciona uma camada de experiência operacional.

O sistema deve simular o contexto em que uma prova realmente acontece.

Isso inclui:

- inscrição em aplicação específica;
- horário oficial de abertura;
- horário limite para início;
- duração real da prova;
- cronômetro controlado pelo backend;
- regras de permanência mínima;
- regras de saída;
- local e sala virtual;
- sessão única;
- controle de múltiplos logins;
- registro de troca de aba;
- registro de perda de foco;
- encerramento automático;
- envio definitivo da prova;
- cálculo de resultado;
- classificação;
- ranking;
- liberação de gabarito conforme regras;
- análise pós-prova;
- histórico de aplicações.

A experiência não deve terminar quando a prova acaba.

Após a aplicação, a plataforma também deve ajudar o candidato a entender onde errou e onde precisa melhorar.

Os relatórios deverão permitir identificar:

- disciplinas com maior quantidade de erros;
- assuntos com maior dificuldade;
- áreas com melhor e pior desempenho;
- questões corretas, incorretas e não respondidas;
- percentual de acertos por disciplina;
- percentual de acertos por assunto;
- evolução entre aplicações;
- comparação com médias da aplicação;
- pontos fortes;
- pontos fracos;
- prioridades de revisão;
- recomendações e dicas de estudo.

Os fluxos distribuídos que dependem da conclusão de múltiplas etapas poderão ser coordenados pelo `diad-orchestrator-service`.

Exemplo:

![Uploading image.png…]()

## 4 - Exemplo prático

Imagine o seguinte cenário.

1. Diogo cria uma conta.
2. O `diad-auth-service` autentica o usuário.
3. O `diad-candidate-service` mantém seu perfil.
4. Diogo escolhe uma aplicação disponível.
5. O `diad-exam-service` fornece a estrutura e as regras da prova.
6. Diogo realiza a inscrição.
7. O `diad-application-service` cria sua participação.
8. O sistema associa município, local virtual e sala.
9. No dia da prova, Diogo entra na sala de espera.
10. O servidor libera a prova no horário configurado.
11. O `diad-question-service` fornece as questões.
12. O `diad-answer-service` registra as respostas.
13. O `diad-security-service` registra eventos de integridade.
14. A aplicação é finalizada pelo candidato ou automaticamente pelo servidor.
15. O `diad-application-service` publica o evento de finalização.
16. O `diad-orchestrator-service` inicia o workflow pós-prova e acompanha suas etapas.
17. O `diad-scoring-service` calcula o resultado.
18. O `diad-performance-service` gera a análise de desempenho.
19. O `diad-ranking-service` atualiza as classificações.
20. O `diad-application-service` verifica se o candidato cumpriu os critérios para acessar o gabarito.
21. O `diad-orchestrator-service` consolida o status do fluxo e identifica quando o resultado está pronto para disponibilização.
22. O `diad-communication-service` notifica o candidato quando resultado e relatório estiverem disponíveis.

---

## 5 - Fluxo resumido

```text
Candidato / React
      |
      v
diad-api-gateway
      |
      +----------------------+
      |                      |
      v                      v
diad-auth-service    diad-candidate-service
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
diad-question-service  diad-security-service
      |
      v
diad-answer-service
      |
      v
ApplicationFinished
      |
      v
Kafka
      |
      v
diad-orchestrator-service
      |
      +----------------------+----------------------+
      |                      |                      |
      v                      v                      v
diad-scoring-service  diad-performance-service  diad-ranking-service
      |                      |                      |
      +----------------------+----------------------+
                             |
                             v
                 diad-communication-service
```

O `diad-orchestrator-service` não substitui Kafka nem os eventos de domínio.

Sua função é coordenar processos distribuídos que possuem dependências entre etapas, mantendo o estado do workflow e acompanhando sua conclusão.

Fluxo pós-prova:

```text
ApplicationFinished
        |
        v
diad-orchestrator-service
        |
        v
ScoringRequested / ScoreCalculated
        |
        v
PerformanceRequested / PerformanceReportGenerated
        |
        v
RankingUpdateRequested / RankingUpdated
        |
        v
AnswerKeyAccessEvaluated
        |
        v
ResultAvailable
        |
        v
CommunicationRequested
```

Eventos simples que não exigem coordenação de múltiplas etapas poderão continuar utilizando coreografia diretamente entre os serviços.

---

## 6 - Objetivo técnico

Principais objetivos:

- Kotlin;
- Java 21;
- Spring Boot;
- Arquitetura Hexagonal;
- microservices;
- event-driven architecture;
- orquestração de workflows distribuídos;
- database per service;
- API Gateway;
- autenticação centralizada;
- JWT;
- Spring Security;
- Kafka;
- RabbitMQ;
- Redis;
- Outbox Pattern;
- Inbox Pattern;
- Saga / Process Manager quando aplicável;
- idempotência;
- Feature Toggles;
- Flyway;
- OpenAPI;
- AsyncAPI;
- correlationId;
- logs estruturados;
- OpenTelemetry;
- Prometheus;
- Grafana;
- Docker;
- Kubernetes;
- escalabilidade horizontal;
- regras configuráveis por prova;
- backend como autoridade de tempo;
- desenvolvimento incremental entre backend e frontend.

---

## 7 - Microservices

| Serviço | Responsabilidade |
|---|---|
| `diad-auth-service` | Autenticação e autorização |
| `diad-api-gateway` | Entrada técnica e roteamento |
| `diad-candidate-service` | Cadastro e perfil do candidato |
| `diad-exam-service` | Estrutura e políticas das provas |
| `diad-question-service` | Banco de questões |
| `diad-application-service` | Sessão, horário, sala e regras de aplicação |
| `diad-answer-service` | Respostas |
| `diad-orchestrator-service` | Coordenação de workflows distribuídos |
| `diad-scoring-service` | Correção e nota |
| `diad-performance-service` | Desempenho, evolução e recomendações |
| `diad-ranking-service` | Ranking e classificação |
| `diad-security-service` | Integridade da sessão |
| `diad-communication-service` | Comunicações |
| `diad-config-server` | Configuração centralizada |

---

## 8 - Bibliotecas e repositórios de apoio

| Repositório | Responsabilidade |
|---|---|
| `diad-platform-infra` | Docker Compose, Kubernetes, Kafka, RabbitMQ, Redis, bancos, Prometheus e Grafana |
| `diad-shared-contracts` | OpenAPI, AsyncAPI e schemas de eventos |
| `diad-observability-starter` | Logs estruturados, correlationId, tracing, MDC e métricas |
| `diad-web` | Frontend React |

---

## 9 - Responsabilidade dos serviços

### diad-auth-service

Responsável por:

- cadastro de usuário;
- login;
- validação de senha;
- JWT;
- refresh token;
- roles;
- permissões.

Roles iniciais:

```text
CANDIDATE
SCHOOL_ADMIN
PLATFORM_ADMIN
```

---

### diad-api-gateway

Responsável por:

- receber requisições externas;
- validar JWT;
- aplicar CORS;
- aplicar rate limit;
- propagar correlationId;
- rotear chamadas.

Não deve conter regra de negócio.

---

### diad-candidate-service

Responsável por:

- criar candidato;
- consultar candidato;
- atualizar perfil;
- manter município;
- vincular candidato a escola;
- manter dados básicos do candidato.

Eventos:

```text
CandidateCreated
CandidateUpdated
CandidateSchoolLinked
```

---

### diad-exam-service

Responsável pela definição das provas.

Conceitos principais:

```text
ExamType
ExamEdition
ExamDay
ExamSection
QuestionStructure
ApplicationPolicy
ScoringPolicy
RankingPolicy
```

Responsabilidades:

- cadastrar tipo de prova;
- cadastrar edição;
- definir duração;
- definir número de dias;
- definir seções;
- definir tipos de questão;
- definir pesos;
- definir critérios de eliminação;
- definir políticas de aplicação;
- definir políticas de pontuação;
- definir políticas de ranking.

---

### diad-question-service

Responsável pelo banco de questões.

Responsabilidades:

- cadastrar questão;
- importar questão;
- definir disciplina;
- definir assunto;
- definir área;
- definir alternativas;
- definir resposta correta;
- registrar ano;
- registrar banca;
- registrar fonte;
- registrar dificuldade.

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

Responsável pela experiência real de aplicação.

Responsabilidades:

- inscrição;
- associação do candidato à aplicação;
- sala virtual;
- local virtual;
- horário de início;
- horário limite;
- duração;
- permanência mínima;
- sessão;
- status;
- encerramento automático;
- regras de saída;
- regras de submissão;
- avaliação de acesso ao gabarito.

Conceitos:

```text
Application
ApplicationSession
VirtualLocation
VirtualRoom
ApplicationStatus
ApplicationPolicy
AnswerKeyAccessPolicy
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

O `diad-application-service` mantém a autoridade sobre o estado da aplicação.

O `diad-orchestrator-service` poderá coordenar processos que se iniciam após mudanças relevantes de estado, mas não poderá alterar regras internas da aplicação por conta própria.

---

### diad-answer-service

Responsável por:

- registrar resposta;
- alterar resposta durante a sessão;
- garantir idempotência;
- registrar data/hora;
- bloquear alteração após finalização;
- consolidar respostas.

Eventos:

```text
AnswerSubmitted
AnswerChanged
ExamAnswersFinalized
```

---

### diad-orchestrator-service

Responsável por coordenar workflows distribuídos que atravessam múltiplos microservices.

O serviço atua como **Process Manager / Saga Orchestrator** quando um fluxo exige acompanhamento explícito das etapas.

Responsabilidades:

- iniciar workflows a partir de eventos relevantes;
- acompanhar o estado de processos distribuídos;
- coordenar a sequência lógica entre serviços;
- emitir comandos ou eventos de continuidade;
- aguardar eventos de conclusão;
- controlar timeout de etapas;
- registrar tentativas;
- evitar execução duplicada de um mesmo workflow;
- permitir retomada após falhas;
- marcar workflows como concluídos ou falhos;
- propagar correlationId e identificadores de processo;
- fornecer rastreabilidade do fluxo completo.

Exemplo:

```text
ApplicationFinished
        |
        v
POST_EXAM_WORKFLOW_STARTED
        |
        v
ScoreCalculated
        |
        v
PerformanceReportGenerated
        |
        v
RankingUpdated
        |
        v
AnswerKeyAccessEvaluated
        |
        v
ResultAvailable
        |
        v
CommunicationRequested
        |
        v
POST_EXAM_WORKFLOW_COMPLETED
```

O serviço não deve:

- calcular nota;
- gerar relatório de desempenho;
- calcular ranking;
- decidir permanência mínima;
- decidir regras de aplicação;
- armazenar respostas;
- executar regra de negócio pertencente a outro domínio.

Regra central:

```text
Orchestrator coordena.
Serviços de domínio decidem.
```

Fluxos simples e independentes continuarão utilizando coreografia por eventos sem necessidade do Orchestrator.

---

### diad-scoring-service

Responsável exclusivamente pela correção e cálculo de nota.

Responsabilidades:

- corrigir respostas;
- calcular nota;
- aplicar pesos;
- aplicar regras de pontuação;
- consolidar acertos;
- consolidar erros;
- consolidar não respondidas;
- publicar resultado de pontuação.

Estratégias possíveis:

```text
SimpleScore
WeightedScore
NegativeScore
CebraspeScore
TriScore
PercentageScore
CustomScore
```

---

### diad-performance-service

Responsável pela análise pós-prova.

Responsabilidades:

- gerar relatório individual;
- analisar desempenho geral;
- analisar desempenho por área;
- analisar desempenho por disciplina;
- analisar desempenho por assunto;
- identificar pontos fortes;
- identificar pontos fracos;
- identificar assuntos prioritários;
- comparar aplicações anteriores;
- calcular evolução;
- gerar recomendações;
- gerar dicas de estudo;
- disponibilizar dados para dashboards;
- gerar relatórios agregados para escolas.

---

### diad-ranking-service

Responsável por:

- ranking global;
- ranking por país;
- ranking por estado;
- ranking por município;
- ranking por escola;
- ranking por turma;
- ranking por prova;
- ranking por curso;
- ranking por categoria.

Redis poderá ser utilizado para leitura frequente de rankings.

---

### diad-security-service

Responsável por eventos de integridade.

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

---

### diad-communication-service

Responsável pelas comunicações.

Exemplos:

- confirmação de cadastro;
- confirmação de inscrição;
- cartão de confirmação;
- local e sala;
- lembrete da prova;
- aviso de início;
- aviso de resultado;
- aviso de gabarito disponível;
- envio de relatório de desempenho;
- notificações institucionais.

Canais possíveis:

```text
E-mail
WhatsApp
Push
SMS
```

RabbitMQ será utilizado para filas de comunicação, retry e DLQ.

---

## 10 - Arquitetura Hexagonal

Cada microservice deve seguir:

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

No `diad-orchestrator-service`, o mesmo padrão será utilizado.

Seu domínio será relacionado ao estado do workflow:

```text
Workflow
WorkflowStep
WorkflowStatus
WorkflowContext
ProcessInstance
```

---

## 11 - Comunicação entre serviços

### REST

Para operações síncronas.

### Kafka

Para eventos de domínio e integração entre contextos.

Exemplos:

```text
CandidateCreated
ExamPublished
ApplicationRegistered
ApplicationStarted
AnswerSubmitted
ApplicationFinished
PostExamWorkflowStarted
ScoreCalculated
PerformanceReportGenerated
RankingUpdated
AnswerKeyAccessGranted
ResultAvailable
PostExamWorkflowCompleted
```

O `diad-orchestrator-service` poderá consumir eventos de domínio e publicar comandos/eventos necessários para continuidade dos workflows.

### RabbitMQ

Para filas de trabalho.

Exemplos:

```text
SendRegistrationEmail
SendExamReminder
SendResultAvailableNotification
SendPerformanceReport
RetryNotification
ReprocessFailedJob
```

---

## 12 - Autenticação e autorização

Fluxo:

```text
1. usuário realiza login
2. Auth Service valida credenciais
3. Auth Service gera JWT
4. frontend envia JWT
5. API Gateway valida token
6. Gateway propaga contexto
7. requisição segue para o serviço interno
```

O `diad-orchestrator-service` não será exposto diretamente ao candidato para execução de workflows internos.

---

## 13 - Banco de dados

Estratégia:

```text
database per service
```

| Serviço | Banco |
|---|---|
| `diad-auth-service` | `auth_db` |
| `diad-candidate-service` | `candidate_db` |
| `diad-exam-service` | `exam_db` |
| `diad-question-service` | `question_db` |
| `diad-application-service` | `application_db` |
| `diad-answer-service` | `answer_db` |
| `diad-orchestrator-service` | `orchestrator_db` |
| `diad-scoring-service` | `scoring_db` |
| `diad-performance-service` | `performance_db` |
| `diad-ranking-service` | `ranking_db` |
| `diad-security-service` | `security_db` |
| `diad-communication-service` | `communication_db` |

O `orchestrator_db` deverá armazenar somente informações de coordenação do processo:

```text
workflow_id
workflow_type
aggregate_id
correlation_id
current_step
workflow_status
started_at
updated_at
completed_at
retry_count
failure_reason
```

Tecnologia inicial:

```text
PostgreSQL
```

---

## 14 - Regras de aplicação de provas

As regras não devem ficar fixas para um exame específico.

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

O Orchestrator não implementará essas regras.

Ele apenas reagirá ao resultado das decisões produzidas pelos serviços responsáveis.

---

## 15 - Gabarito e acesso pós-prova

O acesso ao gabarito será controlado pelo `diad-application-service`.

O `diad-orchestrator-service` poderá aguardar essa avaliação como parte do workflow pós-prova, mas não decidirá a regra.

---

## 16 - Relatórios de desempenho

O `diad-performance-service` deverá gerar relatórios pós-prova.

O `diad-orchestrator-service` poderá coordenar o momento em que a geração deve ocorrer dentro do workflow, sem absorver a regra analítica.

---

## 17 - Padrões aplicados

- Arquitetura Hexagonal;
- Ports and Adapters;
- Microservices;
- Event-driven architecture;
- Database per service;
- API Gateway;
- Orchestration;
- Saga / Process Manager;
- Use Cases;
- DTOs nas bordas;
- mappers explícitos;
- entidades JPA separadas de domínio;
- Constructor Injection;
- Strategy Pattern;
- Policy Pattern;
- Spring Security;
- JWT;
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
- Flyway;
- OpenAPI;
- AsyncAPI.

---

## 18 - Observabilidade

Campos adicionais para workflows:

```text
workflowId
workflowType
workflowStep
```

Métricas adicionais:

```text
orchestrator_workflows_started_total
orchestrator_workflows_completed_total
orchestrator_workflows_failed_total
orchestrator_workflow_duration_seconds
```

---

## 19 - Tecnologias

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

Infraestrutura:

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

---

## 20 - Modelo de negócio

A plataforma será SaaS, com modelos B2C e B2B.

---

## 21 - Roadmap de desenvolvimento

```text
1. diad-candidate-service
2. cadastro React

3. diad-auth-service
4. login React

5. diad-api-gateway
6. integração inicial frontend → gateway → serviços

7. diad-exam-service
8. listagem de provas React

9. diad-application-service
10. inscrição React

11. local e sala virtual
12. cartão de confirmação React

13. diad-question-service
14. estrutura da prova

15. diad-answer-service
16. tela de prova React

17. cronômetro sincronizado pelo backend
18. regras de início, permanência e finalização
19. política de acesso ao gabarito

20. diad-security-service
21. eventos de integridade

22. Kafka
23. Outbox
24. Inbox
25. contratos de eventos

26. diad-orchestrator-service
27. modelo de Workflow / Process Manager
28. workflow pós-prova
29. idempotência e recuperação de workflows

30. diad-scoring-service
31. resultado React

32. diad-performance-service
33. relatório de desempenho React
34. recomendações e evolução

35. diad-ranking-service
36. ranking React

37. diad-communication-service
38. RabbitMQ
39. retry e DLQ

40. diad-observability-starter
41. métricas e tracing
42. observabilidade dos workflows

43. diad-config-server

44. diad-platform-infra
45. Docker Compose
46. Kubernetes

47. testes de carga
48. testes de concorrência
49. testes de falha e retomada do Orchestrator
50. hardening de segurança
51. preparação do MVP
```

---

## 22 - Status

Projeto em desenvolvimento.

```text
diad-simulation-platform: criado
README.md: em construção
docs/: em construção
microservices: ainda não iniciados
frontend: ainda não iniciado
```

As documentações específicas deverão considerar o `diad-orchestrator-service` e os workflows distribuídos coordenados por ele.

---

## Princípios do projeto

```text
domínio independente de framework
baixo acoplamento
responsabilidade única por contexto
contratos explícitos
idempotência
observabilidade
segurança por padrão
escalabilidade horizontal
regras configuráveis
backend como autoridade da prova
frontend sem regra crítica de negócio
orquestração sem absorção de regras de domínio
coreografia para eventos simples
orquestração para workflows distribuídos complexos
evolução incremental
```

**Dia D Simulation Platform**

> Viva o dia antes dele chegar.
