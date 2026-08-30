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

A plataforma será construída para suportar diferentes tipos de provas, seletivos e concursos, adptada a cada cenário específico.. como base primeiro, iremos trabalhar com o ENEM.

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
| `diad-scoring-service` | Correção e cálculo de notas |
| `diad-performance-service` | Relatórios de desempenho, evolução e recomendações |
| `diad-ranking-service` | Rankings e classificações |
| `diad-security-service` | Eventos de segurança e integridade da sessão |
| `diad-communication-service` | E-mail, Whatsapp, notificações, lembretes, retry e DLQ |
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

O objetivo é transformar cada aplicação em uma experiência completa de:

```text
simulação
+
resultado
+
diagnóstico
+
orientação de estudo
```

---

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
15. O `diad-scoring-service` calcula o resultado.
16. O `diad-performance-service` gera a análise de desempenho.
17. O `diad-ranking-service` atualiza as classificações.
18. O `diad-application-service` verifica se o candidato cumpriu os critérios para acessar o gabarito.
19. O `diad-communication-service` notifica o candidato quando resultado e relatório estiverem disponíveis.

Exemplo:

```text
Candidato: Diogo Ferreira
Prova: ENEM - Simulação Nacional #01
Município: São Luís - MA
Local virtual: Escola X
Sala virtual: 07

Status: FINISHED
Nota: 712.40

Ranking nacional: 1.842
Ranking estadual: 73
Ranking municipal: 21

Tempo de permanência: 04h17
Gabarito: LIBERADO

Melhor área: Linguagens
Pior área: Ciências da Natureza

Maior dificuldade:
- Física
- Eletrodinâmica
- Cinemática

Recomendação:
Revisar os assuntos com taxa de acerto abaixo de 50% antes da próxima aplicação.
```

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
Kafka
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

Fluxo pós-prova:

```text
ApplicationFinished
        |
        v
ScoreCalculated
        |
        +----------------------+
        |                      |
        v                      v
PerformanceReportGenerated  RankingUpdated
        |
        v
AnswerKeyAccessEvaluated
        |
        v
ResultAvailable
```

---

## 6 - Objetivo técnico

Principais objetivos:

- Kotlin;
- Java 21;
- Spring Boot;
- Arquitetura Hexagonal;
- microservices;
- event-driven architecture;
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

O serviço não deve possuir lógica semelhante a:

```kotlin
if (examType == "ENEM") {
    ...
} else if (examType == "CEBRASPE") {
    ...
}
```

As regras devem ser implementadas através de políticas e estratégias.

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

Exemplo de relatório:

```text
PerformanceReport

candidateId
examId
overallScore

subjects[]
topics[]
strongPoints[]
weakPoints[]
recommendations[]
```

Exemplo de regra inicial:

```text
Abaixo de 40%
→ prioridade alta de revisão

40% até 70%
→ precisa reforçar

Acima de 70%
→ bom desempenho
```

Essas regras poderão evoluir futuramente para modelos mais sofisticados.

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

Responsabilidades:

- registrar eventos;
- calcular nível de risco;
- aplicar políticas configuráveis;
- sinalizar sessão suspeita;
- recomendar bloqueio ou desqualificação;
- manter histórico.

O serviço não deve assumir que qualquer evento representa automaticamente fraude.

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

Integrações externas devem ser abstraídas por portas.

Exemplo:

```text
NotificationProviderPort
    |
    +--> EmailProviderAdapter
    +--> WhatsAppProviderAdapter
    +--> PushProviderAdapter
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

Regras:

- controller não acessa repository diretamente;
- consumer não executa regra de negócio diretamente;
- repository não conhece use case;
- DTO não entra no domínio;
- entidade JPA não é modelo de domínio;
- domínio não depende de Spring;
- domínio não conhece Kafka;
- domínio não conhece RabbitMQ;
- domínio não conhece HTTP;
- toda entrada passa por porta de entrada;
- toda saída passa por porta de saída;
- integrações externas devem ser abstraídas.

---

## 11 - Comunicação entre serviços

### REST

Para operações síncronas.

### Kafka

Para eventos de domínio.

Exemplos:

```text
CandidateCreated
ExamPublished
ApplicationRegistered
ApplicationStarted
AnswerSubmitted
ApplicationFinished
ScoreCalculated
PerformanceReportGenerated
RankingUpdated
AnswerKeyAccessGranted
```

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

Exemplo:

```json
{
  "sub": "user-id",
  "candidateId": "candidate-id",
  "role": "CANDIDATE"
}
```

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
| `diad-scoring-service` | `scoring_db` |
| `diad-performance-service` | `performance_db` |
| `diad-ranking-service` | `ranking_db` |
| `diad-security-service` | `security_db` |
| `diad-communication-service` | `communication_db` |

Tecnologia inicial:

```text
PostgreSQL
```

Redis poderá ser utilizado para:

- cache;
- sessões temporárias;
- rate limiting;
- locks distribuídos;
- ranking;
- informações temporárias de aplicação.

---

## 14 - Regras de aplicação de provas

As regras não devem ficar fixas para um exame específico.

Exemplo:

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

Exemplo:

```text
ENEM

StartTime: configurado
EndTime: configurado
MinimumStay: configurado
Days: 2
Essay: enabled
Scoring: TRI
AnswerKeyAccess: condicionado à permanência mínima
```

Outro exemplo:

```text
Cebraspe

QuestionFormat: TRUE_FALSE
NegativeScoring: enabled
MinimumStay: configurado
AnswerKeyAccess: conforme política da aplicação
```

---

## 15 - Gabarito e acesso pós-prova

O acesso ao gabarito será controlado pelo `diad-application-service`.

O `diad-scoring-service` não será responsável por decidir quem pode acessar o gabarito.

A regra deverá ser abstraída.

Exemplo conceitual:

```kotlin
interface AnswerKeyAccessPolicy {
    fun canAccess(context: AnswerKeyAccessContext): Boolean
}
```

Uma política poderá considerar:

```text
tempo de permanência
status da aplicação
horário de liberação
tipo da prova
edição
possível desqualificação
```

Exemplo:

```text
Candidato permaneceu o tempo mínimo exigido
+
prova finalizada corretamente
+
gabarito já está dentro da janela de liberação

→ GABARITO LIBERADO
```

Caso contrário:

```text
→ GABARITO BLOQUEADO
```

Isso permite que diferentes provas tenham regras diferentes sem acoplamento.

---

## 16 - Relatórios de desempenho

O `diad-performance-service` deverá gerar relatórios pós-prova.

O relatório poderá exibir:

```text
Nota geral
Acertos
Erros
Não respondidas

Desempenho por:
- área
- disciplina
- assunto
- dificuldade
- tipo de questão

Pontos fortes
Pontos fracos
Assuntos prioritários
Evolução
Comparação com aplicação anterior
Comparação com média geral
Recomendações
Dicas de estudo
```

Exemplo:

```text
Física: 42% de acerto
Matemática: 78% de acerto
Português: 81% de acerto

Prioridade de revisão:
1. Eletrodinâmica
2. Cinemática
3. Óptica

Recomendação:
Revisar os conteúdos classificados como prioridade alta antes da próxima aplicação.
```

Para instituições, poderão existir relatórios agregados:

```text
aluno
turma
série
escola
município
```

---

## 17 - Padrões aplicados

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

A plataforma deverá possuir:

- logs estruturados;
- métricas;
- tracing;
- dashboards;
- health checks;
- correlationId.

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

Métricas possíveis:

```text
active_exam_sessions
exam_sessions_started_total
exam_sessions_finished_total
answers_submitted_total
security_events_total
score_calculations_total
performance_reports_generated_total
ranking_updates_total
answer_key_access_granted_total
messages_dlq_total
```

Grafana poderá exibir:

- sessões ativas;
- candidatos conectados;
- latência;
- taxa de erro;
- volume de respostas;
- eventos de segurança;
- backlog;
- DLQ;
- geração de relatórios;
- atualização de ranking.

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

A plataforma será SaaS.

### B2C

Assinatura individual para candidatos.

Possíveis recursos:

- aplicações nacionais;
- histórico;
- ranking;
- gabarito;
- relatórios;
- recomendações;
- evolução.

### B2B

Escolas, cursinhos e instituições poderão contratar por quantidade de alunos.

Possíveis recursos:

- gestão de alunos;
- turmas;
- aplicações privadas;
- ranking interno;
- relatórios;
- desempenho por turma;
- análise por disciplina;
- acompanhamento de evolução;
- API;
- white-label futuramente.

---

## 21 - Roadmap de desenvolvimento

```text
1. diad-candidate-service
2. cadastro React

3. diad-auth-service
4. login React

5. diad-exam-service
6. listagem de provas React

7. diad-application-service
8. inscrição React

9. local e sala virtual
10. cartão de confirmação React

11. diad-question-service
12. estrutura da prova

13. diad-answer-service
14. tela de prova React

15. cronômetro sincronizado pelo backend
16. regras de início, permanência e finalização
17. política de acesso ao gabarito

18. diad-security-service
19. eventos de integridade

20. diad-scoring-service
21. resultado React

22. diad-performance-service
23. relatório de desempenho React
24. recomendações e evolução

25. diad-ranking-service
26. ranking React

27. diad-communication-service

28. Kafka
29. RabbitMQ
30. Outbox
31. Inbox
32. retry e DLQ

33. diad-observability-starter
34. métricas e tracing

35. diad-api-gateway
36. diad-config-server

37. diad-platform-infra
38. Docker Compose
39. Kubernetes

40. testes de carga
41. testes de concorrência
42. hardening de segurança
43. preparação do MVP
```

---

## 22 - Status

Projeto em desenvolvimento.

```text
diad-simulation-platform: criado
README.md: em construção
docs/: a ser criado
microservices: ainda não iniciados
frontend: ainda não iniciado
```

Próximos documentos:

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
evolução incremental
```

O relógio do dispositivo do candidato não deve ser autoridade para início ou fim de uma aplicação.

O backend será responsável pelo tempo oficial.

```text
serverTime
applicationStartsAt
applicationEndsAt
sessionExpiresAt
```

O frontend apenas representa o tempo calculado pelo servidor.

---

**Dia D Simulation Platform**

> Viva o dia antes dele chegar.
