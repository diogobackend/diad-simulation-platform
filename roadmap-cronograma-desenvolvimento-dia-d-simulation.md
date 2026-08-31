# Roadmap / Cronograma de Desenvolvimento --- Dia D Simulation

Documentação oficial da **ordem de desenvolvimento** da plataforma **Dia
D Simulation**.

Este roadmap foi organizado para reduzir retrabalho, preservar
dependências entre módulos e evitar implementação prematura de
componentes que dependem de decisões ainda não consolidadas.

> **Regra principal:** seguir a ordem das fases. Uma fase só deve ser
> considerada concluída quando todos os critérios de aceite definidos
> nela estiverem atendidos.

------------------------------------------------------------------------

## Sumário

1.  [Objetivo do roadmap](#1-objetivo-do-roadmap)
2.  [Princípios de execução](#2-princípios-de-execução)
3.  [Visão geral da ordem de
    desenvolvimento](#3-visão-geral-da-ordem-de-desenvolvimento)
4.  [Fase 0 --- Fundação do projeto](#4-fase-0--fundação-do-projeto)
5.  [Fase 1 --- Infraestrutura local e padrões
    técnicos](#5-fase-1--infraestrutura-local-e-padrões-técnicos)
6.  [Fase 2 --- Auth Service](#6-fase-2--auth-service)
7.  [Fase 3 --- Candidate Service](#7-fase-3--candidate-service)
8.  [Fase 4 --- Exam Service](#8-fase-4--exam-service)
9.  [Fase 5 --- Question Service](#9-fase-5--question-service)
10. [Fase 6 --- Application Service](#10-fase-6--application-service)
11. [Fase 7 --- Answer Service](#11-fase-7--answer-service)
12. [Fase 8 --- Fluxo completo de
    prova](#12-fase-8--fluxo-completo-de-prova)
13. [Fase 9 --- Kafka e eventos de
    domínio](#13-fase-9--kafka-e-eventos-de-domínio)
14. [Fase 10 --- RabbitMQ e comandos
    assíncronos](#14-fase-10--rabbitmq-e-comandos-assíncronos)
15. [Fase 11 --- Orchestrator
    Service](#15-fase-11--orchestrator-service)
16. [Fase 12 --- Scoring Service](#16-fase-12--scoring-service)
17. [Fase 13 --- Performance Service](#17-fase-13--performance-service)
18. [Fase 14 --- Ranking Service](#18-fase-14--ranking-service)
19. [Fase 15 --- Answer Key Service](#19-fase-15--answer-key-service)
20. [Fase 16 --- Result Service](#20-fase-16--result-service)
21. [Fase 17 --- Communication
    Service](#21-fase-17--communication-service)
22. [Fase 18 --- Audit Service](#22-fase-18--audit-service)
23. [Fase 19 --- Observabilidade](#23-fase-19--observabilidade)
24. [Fase 20 --- Containerização](#24-fase-20--containerização)
25. [Fase 21 --- Kubernetes](#25-fase-21--kubernetes)
26. [Fase 22 --- CI/CD](#26-fase-22--cicd)
27. [Fase 23 --- Testes integrados](#27-fase-23--testes-integrados)
28. [Fase 24 --- Testes de carga](#28-fase-24--testes-de-carga)
29. [Fase 25 --- Hardening](#29-fase-25--hardening)
30. [Fase 26 --- Pré-produção e
    homologação](#30-fase-26--pré-produção-e-homologação)
31. [Fase 27 --- Preparação para o Dia
    D](#31-fase-27--preparação-para-o-dia-d)
32. [Dependências entre serviços](#32-dependências-entre-serviços)
33. [Ordem exata de implementação](#33-ordem-exata-de-implementação)
34. [Critérios para evitar
    retrabalho](#34-critérios-para-evitar-retrabalho)
35. [Checklist final do roadmap](#35-checklist-final-do-roadmap)

------------------------------------------------------------------------

# 1. Objetivo do roadmap

Este roadmap define:

-   o que deve ser criado;
-   em qual ordem;
-   quais dependências devem existir antes de avançar;
-   quais decisões precisam estar estáveis;
-   quando cada aplicação deve entrar;
-   quando infraestrutura, mensageria e observabilidade devem ser
    adicionadas;
-   quais validações devem acontecer antes da próxima fase.

A sequência foi organizada para evitar situações como:

``` text
criar scoring antes de existir aplicação finalizada
criar ranking antes de existir performance
criar orchestrator antes dos eventos e comandos estarem definidos
criar Kubernetes antes dos serviços estarem estabilizados
criar observabilidade no final sem instrumentação planejada
```

------------------------------------------------------------------------

# 2. Princípios de execução

## 2.1 Não criar dependência antes do domínio

Primeiro:

``` text
domínio
casos de uso
contratos
persistência
API
```

Depois:

``` text
mensageria
orquestração
infraestrutura
```

------------------------------------------------------------------------

## 2.2 Não começar por microserviços complexos

A primeira prioridade é o **fluxo funcional mínimo da prova**:

``` text
candidato
  ->
exame
  ->
questões
  ->
aplicação
  ->
respostas
  ->
finalização
```

Sem esse fluxo funcionando, nenhum processamento pós-prova deve ser
iniciado.

------------------------------------------------------------------------

## 2.3 Contratos antes de integração

Antes de integrar dois serviços:

``` text
request definido
response definido
event definido
command definido
error contract definido
```

Isso reduz alterações simultâneas em producer e consumer.

------------------------------------------------------------------------

## 2.4 Banco evoluído por migration

Toda mudança estrutural deve ser feita via Flyway.

Nunca:

``` text
alteração manual em banco
migration sobrescrita
ajuste direto em produção
```

------------------------------------------------------------------------

## 2.5 Cada fase deve terminar utilizável

Evitar fases "pela metade".

Uma fase só termina quando houver:

``` text
código
testes
persistência
endpoint ou consumer
tratamento de erro
logs básicos
documentação
```

------------------------------------------------------------------------

# 3. Visão geral da ordem de desenvolvimento

Ordem macro:

``` text
0. Fundação
1. Infraestrutura local
2. Auth
3. Candidate
4. Exam
5. Question
6. Application
7. Answer
8. Fluxo completo da prova
9. Kafka
10. RabbitMQ
11. Orchestrator
12. Scoring
13. Performance
14. Ranking
15. Answer Key
16. Result
17. Communication
18. Audit
19. Observabilidade
20. Docker
21. Kubernetes
22. CI/CD
23. Testes integrados
24. Testes de carga
25. Hardening
26. Homologação
27. Preparação Dia D
```

------------------------------------------------------------------------

# 4. Fase 0 --- Fundação do projeto

## Objetivo

Definir a base arquitetural antes de criar regras de negócio.

## Criar

``` text
estrutura dos repositórios
padrão de branches
convenção de commits
arquitetura hexagonal
package structure
tratamento global de erros
DTOs base
correlation ID
padrões REST
padrões de eventos
padrões de comandos
```

## Estrutura base de cada serviço

``` text
domain
application
infrastructure
adapter
```

Exemplo:

``` text
src/main/java
|
+-- domain
|   +-- model
|   +-- exception
|
+-- application
|   +-- port
|   +-- usecase
|
+-- adapter
|   +-- in
|   +-- out
|
+-- infrastructure
```

## Também definir agora

``` text
Java 21
Spring Boot
PostgreSQL
Flyway
JUnit
Mockito
Testcontainers
OpenAPI
Docker
```

## Critério de aceite

``` text
[ ] projeto sobe
[ ] padrão arquitetural validado
[ ] tratamento de erros criado
[ ] correlationId preparado
[ ] base de testes funcionando
[ ] migration inicial funcionando
```

------------------------------------------------------------------------

# 5. Fase 1 --- Infraestrutura local e padrões técnicos

## Objetivo

Criar dependências locais para desenvolvimento.

## Criar

``` text
docker-compose
PostgreSQL
Redis
Kafka
RabbitMQ
```

Ainda não integrar todos aos serviços.

O objetivo nesta fase é apenas garantir que a infraestrutura possa ser
executada localmente.

## Criar também

``` text
application-local.yml
application-test.yml
application-prod.yml
```

## Critério de aceite

``` text
[ ] PostgreSQL acessível
[ ] Redis acessível
[ ] Kafka acessível
[ ] RabbitMQ acessível
[ ] aplicações conseguem subir localmente
```

------------------------------------------------------------------------

# 6. Fase 2 --- Auth Service

## Criar primeiro

``` text
User / Credential
Role
Authentication
JWT
Refresh Token
```

## Endpoints mínimos

``` text
POST /api/v1/auth/login
POST /api/v1/auth/refresh
GET  /api/v1/auth/me
```

## Não criar ainda

``` text
OAuth externo
MFA
SSO
integrações sociais
```

## Critério de aceite

``` text
[ ] login funcionando
[ ] JWT válido
[ ] refresh funcionando
[ ] endpoints protegidos
[ ] testes unitários
[ ] testes de integração
```

------------------------------------------------------------------------

# 7. Fase 3 --- Candidate Service

## Criar

``` text
Candidate
CandidateStatus
MunicipalityCode
```

## Endpoints

``` text
POST  /api/v1/candidates
GET   /api/v1/candidates/{candidateId}
PATCH /api/v1/candidates/{candidateId}
```

## Regras importantes

``` text
documento único
email único
status ativo
município obrigatório
```

## Critério de aceite

``` text
[ ] candidato cadastrado
[ ] duplicidade bloqueada
[ ] consulta funcionando
[ ] atualização funcionando
[ ] integração com Auth definida
```

------------------------------------------------------------------------

# 8. Fase 4 --- Exam Service

## Criar

``` text
Exam
ExamStatus
ExamSchedule
```

## Estados

``` text
DRAFT
SCHEDULED
OPEN
CLOSED
CANCELLED
```

## Endpoints

``` text
POST /api/v1/exams
GET  /api/v1/exams
GET  /api/v1/exams/{examId}
```

## Definir agora

``` text
data
horário inicial
horário final
duração
quantidade de questões
```

## Critério de aceite

``` text
[ ] exame criado
[ ] agenda definida
[ ] consulta funcionando
[ ] status controlado
```

------------------------------------------------------------------------

# 9. Fase 5 --- Question Service

## Criar

``` text
Question
Alternative
KnowledgeArea
QuestionStatus
```

## Endpoints

``` text
GET /api/v1/exams/{examId}/questions
GET /api/v1/questions/{questionId}
```

Endpoints administrativos podem ser adicionados depois.

## Regra crítica

O endpoint utilizado pelo candidato nunca deve retornar:

``` text
correctAlternative
answerKey
score
```

## Critério de aceite

``` text
[ ] questões associadas ao exame
[ ] alternativas persistidas
[ ] resposta correta protegida
[ ] consulta ordenada
```

------------------------------------------------------------------------

# 10. Fase 6 --- Application Service

Esta é uma das fases mais importantes do sistema.

## Criar

``` text
Application
ApplicationStatus
Allocation
School
Room
```

## Estados

``` text
REGISTERED
READY
IN_PROGRESS
FINISHED
CANCELLED
```

## Endpoints

``` text
POST /api/v1/applications

GET /api/v1/applications/{applicationId}

GET /api/v1/candidates/{candidateId}/applications

GET /api/v1/applications/{applicationId}/allocation

POST /api/v1/applications/{applicationId}/start

GET /api/v1/applications/{applicationId}/session

POST /api/v1/applications/{applicationId}/finish
```

## Regras a estabilizar antes de avançar

``` text
horário permitido para início
tempo máximo de prova
regra de finalização
reentrada
status da aplicação
alocação virtual
```

## Critério de aceite

``` text
[ ] candidato se inscreve
[ ] recebe escola e sala
[ ] aplicação inicia
[ ] cronômetro funciona
[ ] aplicação finaliza
[ ] não finaliza duas vezes
```

------------------------------------------------------------------------

# 11. Fase 7 --- Answer Service

## Criar

``` text
Answer
AnswerStatus
```

## Endpoints

``` text
PUT    /api/v1/applications/{applicationId}/answers/{questionId}

DELETE /api/v1/applications/{applicationId}/answers/{questionId}

GET    /api/v1/applications/{applicationId}/answers
```

## Regras

``` text
só responder com prova em andamento
uma resposta por questão
permitir alteração antes do fim
não permitir alteração após finalização
```

## Critério de aceite

``` text
[ ] resposta salva
[ ] resposta atualizada
[ ] resposta removida
[ ] prova finalizada bloqueia alteração
[ ] resposta recuperável após refresh
```

------------------------------------------------------------------------

# 12. Fase 8 --- Fluxo completo de prova

Antes de criar qualquer etapa pós-prova, validar o fluxo ponta a ponta:

``` text
1. candidato cadastrado
2. autenticação
3. exame disponível
4. inscrição
5. alocação
6. início
7. carregamento de questões
8. resposta
9. alteração de resposta
10. consulta da sessão
11. finalização
```

## Testes obrigatórios

``` text
fluxo normal
tentativa antes do horário
tentativa depois do horário
duplo start
duplo finish
resposta após finish
candidato inválido
exame inválido
sessão expirada
```

## Gate obrigatório

> **Não avançar para mensageria enquanto este fluxo não estiver
> estável.**

------------------------------------------------------------------------

# 13. Fase 9 --- Kafka e eventos de domínio

Agora o domínio principal já existe e pode emitir fatos reais.

## Criar

``` text
event envelope
producer padrão
consumer padrão
eventId
eventVersion
correlationId
```

## Primeiros eventos

``` text
ApplicationStarted
AnswerSubmitted
ApplicationFinished
```

## Tópico inicial

``` text
application.events
```

## Critério de aceite

``` text
[ ] eventos publicados
[ ] contrato versionado
[ ] correlationId propagado
[ ] consumer de teste funcionando
[ ] duplicidade considerada
```

------------------------------------------------------------------------

# 14. Fase 10 --- RabbitMQ e comandos assíncronos

Só criar comandos depois de eventos existirem.

## Criar

``` text
dia-d.commands
dia-d.retry
dia-d.dlx
```

## Comando inicial

``` text
ProcessScoring
```

## Fila inicial

``` text
scoring.process.queue
```

## Criar também

``` text
retry
dead letter
commandId
idempotência
```

## Critério de aceite

``` text
[ ] comando publicado
[ ] comando consumido
[ ] retry funcionando
[ ] DLQ funcionando
[ ] commandId validado
```

------------------------------------------------------------------------

# 15. Fase 11 --- Orchestrator Service

O Orchestrator só entra agora porque eventos e comandos já estão
definidos.

## Criar workflow

``` text
APPLICATION_POST_PROCESSING
```

Fluxo inicial:

``` text
ApplicationFinished
       |
       v
Orchestrator
       |
       v
ProcessScoring
```

## Estado

``` text
workflowId
correlationId
currentStep
status
startedAt
finishedAt
```

## Critério de aceite

``` text
[ ] ApplicationFinished inicia workflow
[ ] ProcessScoring publicado
[ ] estado persistido
[ ] idempotência funcionando
[ ] workflow recuperável
```

------------------------------------------------------------------------

# 16. Fase 12 --- Scoring Service

Agora existe uma aplicação finalizada real para corrigir.

## Criar

``` text
Score
ScoringStatus
ScoringEngine
```

## Entrada

``` text
ProcessScoring
```

## Saída

``` text
ScoringStarted
ScoringFinished
ScoringFailed
```

## Endpoint de consulta

``` text
GET /api/v1/scores/{scoreId}

GET /api/v1/applications/{applicationId}/score
```

## Critério de aceite

``` text
[ ] respostas carregadas
[ ] correção executada
[ ] score persistido
[ ] ScoringFinished publicado
[ ] falha gera evento apropriado
```

------------------------------------------------------------------------

# 17. Fase 13 --- Performance Service

Dependência:

``` text
ScoringFinished
```

## Criar

``` text
Performance
PerformanceByArea
Percentile
```

## Entrada

``` text
CalculatePerformance
```

## Saída

``` text
PerformanceCalculated
```

## Endpoint

``` text
GET /api/v1/applications/{applicationId}/performance
```

## Critério de aceite

``` text
[ ] score consumido
[ ] desempenho calculado
[ ] áreas calculadas
[ ] resultado persistido
[ ] evento publicado
```

------------------------------------------------------------------------

# 18. Fase 14 --- Ranking Service

Dependência:

``` text
PerformanceCalculated
```

## Criar

``` text
Ranking
RankingPosition
```

## Entrada

``` text
GenerateRanking
```

## Saída

``` text
RankingUpdated
```

## Endpoints

``` text
GET /api/v1/exams/{examId}/ranking

GET /api/v1/exams/{examId}/ranking/candidates/{candidateId}
```

## Critério de aceite

``` text
[ ] ranking gerado
[ ] paginação funcionando
[ ] posição individual disponível
[ ] evento publicado
```

------------------------------------------------------------------------

# 19. Fase 15 --- Answer Key Service

Pode ser implementado após a estrutura principal de correção estar
estável.

## Criar

``` text
AnswerKey
AnswerKeyVersion
AnswerKeyPublication
```

## Entrada

``` text
ReleaseAnswerKey
```

## Saída

``` text
AnswerKeyReleased
```

## Endpoint

``` text
GET /api/v1/exams/{examId}/answer-key
```

## Critério de aceite

``` text
[ ] gabarito versionado
[ ] publicação controlada
[ ] consulta bloqueada antes da liberação
[ ] evento publicado
```

------------------------------------------------------------------------

# 20. Fase 16 --- Result Service

O Result Service deve ser criado somente depois que já existirem:

``` text
Score
Performance
Ranking
```

## Responsabilidade

Consolidar leitura.

## Endpoint

``` text
GET /api/v1/applications/{applicationId}/result
```

## Response composto

``` text
exam
score
performance
ranking
status
```

## Critério de aceite

``` text
[ ] resultado consolidado
[ ] PROCESSING enquanto incompleto
[ ] AVAILABLE após conclusão
[ ] nenhuma regra duplicada
```

------------------------------------------------------------------------

# 21. Fase 17 --- Communication Service

Só faz sentido após existir resultado real.

## Criar

``` text
Communication
CommunicationType
CommunicationChannel
CommunicationStatus
```

## Entrada

``` text
SendCommunication
```

## Saídas

``` text
CommunicationSent
CommunicationFailed
```

## Primeiro canal

``` text
EMAIL
```

Outros canais entram depois.

## Critério de aceite

``` text
[ ] comando consumido
[ ] envio realizado
[ ] retry funcionando
[ ] falha vai para fluxo apropriado
[ ] histórico disponível
```

------------------------------------------------------------------------

# 22. Fase 18 --- Audit Service

A auditoria deve entrar depois que os eventos principais estiverem
estáveis.

## Registrar

``` text
ApplicationStarted
ApplicationFinished
ScoringFinished
PerformanceCalculated
RankingUpdated
AnswerKeyReleased
ResultAvailable
CommunicationSent
```

## Endpoint

``` text
GET /api/v1/audit/events
```

## Critério de aceite

``` text
[ ] trilha persistida
[ ] busca por correlationId
[ ] busca por aggregateId
[ ] histórico cronológico
```

------------------------------------------------------------------------

# 23. Fase 19 --- Observabilidade

A instrumentação básica deve existir desde o início, mas a
infraestrutura completa entra aqui.

## Consolidar

``` text
OpenTelemetry
Prometheus
Grafana
logs estruturados
traces
metrics
alerts
```

## Instrumentar

``` text
HTTP
Kafka
RabbitMQ
Database
Workflows
Business Metrics
```

## Dashboards

``` text
Platform Overview
Exam Day
Kafka
RabbitMQ
Scoring
Workflows
```

## Critério de aceite

``` text
[ ] trace ponta a ponta
[ ] correlationId pesquisável
[ ] métricas por serviço
[ ] dashboards funcionando
[ ] alertas essenciais configurados
```

------------------------------------------------------------------------

# 24. Fase 20 --- Containerização

Somente agora padronizar todos os serviços em imagens.

## Para cada aplicação

``` text
Dockerfile
.dockerignore
non-root user
health endpoint
environment variables
```

## Critério de aceite

``` text
[ ] build local
[ ] container sobe
[ ] health funcionando
[ ] config externa
[ ] nenhuma secret na imagem
```

------------------------------------------------------------------------

# 25. Fase 21 --- Kubernetes

Criar manifests somente após containers estabilizados.

## Criar por serviço

``` text
Deployment
Service
ConfigMap
Secret
HPA
PodDisruptionBudget
```

## Depois

``` text
Ingress
NetworkPolicy
observability
```

## Ordem dentro do Kubernetes

``` text
1. namespace
2. config
3. secrets
4. stateful dependencies
5. backend services
6. gateway
7. ingress
8. observability
9. autoscaling
```

## Critério de aceite

``` text
[ ] pods saudáveis
[ ] readiness
[ ] liveness
[ ] service discovery
[ ] rollout
[ ] autoscaling
```

------------------------------------------------------------------------

# 26. Fase 22 --- CI/CD

Não automatizar deploy de aplicação instável.

## Pipeline

``` text
checkout
compile
unit tests
integration tests
static analysis
build
docker build
scan
push
deploy
smoke test
```

## Ambientes

``` text
DEV
QA
PROD
```

## Critério de aceite

``` text
[ ] build automático
[ ] testes bloqueiam merge
[ ] imagem versionada
[ ] deploy automatizado
[ ] rollback possível
```

------------------------------------------------------------------------

# 27. Fase 23 --- Testes integrados

Agora validar o sistema completo.

## Fluxos

``` text
cadastro -> prova -> resultado

application -> Kafka -> orchestrator -> RabbitMQ -> scoring

scoring -> performance -> ranking -> result

result -> communication
```

## Ferramentas sugeridas

``` text
Testcontainers
WireMock
Awaitility
```

## Critério de aceite

``` text
[ ] fluxo completo verde
[ ] retries testados
[ ] DLQ testada
[ ] idempotência testada
[ ] falhas parciais testadas
```

------------------------------------------------------------------------

# 28. Fase 24 --- Testes de carga

Esta fase define o dimensionamento real.

## Cenários

### Pico de autenticação

``` text
milhares de logins simultâneos
```

### Pico de início

``` text
milhares de POST /start
```

### Durante a prova

``` text
grande volume de PUT /answers
```

### Finalização

``` text
milhares de POST /finish
```

### Pós-prova

``` text
grande volume de scoring
performance
ranking
```

## Medir

``` text
RPS
P50
P95
P99
CPU
memória
database connections
Kafka lag
RabbitMQ backlog
```

## Resultado esperado

Esta fase define:

``` text
replicas mínimas
replicas máximas
HPA
pool de conexão
partições Kafka
consumers
recursos Kubernetes
```

------------------------------------------------------------------------

# 29. Fase 25 --- Hardening

Somente depois da arquitetura funcional e dimensionada.

## Revisar

``` text
security
timeouts
retries
circuit breakers
rate limiting
RBAC
network policies
secrets
TLS
dependency vulnerabilities
```

## Também validar

``` text
OWASP
headers
JWT expiration
log sanitization
```

------------------------------------------------------------------------

# 30. Fase 26 --- Pré-produção e homologação

Criar ambiente o mais próximo possível da produção.

## Validar

``` text
deploy
migrations
rollback
backup
restore
observability
alerts
load
failover
```

## Simulações

``` text
queda de pod
queda de consumer
mensagem inválida
DB indisponível
Kafka indisponível
RabbitMQ indisponível
```

------------------------------------------------------------------------

# 31. Fase 27 --- Preparação para o Dia D

Última fase.

## Antes do evento

``` text
congelamento de versão
smoke test
backup
verificação de DLQ
verificação de dashboards
pre-scaling
validação de certificados
validação de consumers
validação do banco
```

## Durante o evento

Monitorar:

``` text
candidatos online
provas iniciadas
respostas/s
falhas
latência
CPU
memória
Kafka lag
RabbitMQ
database
```

## Após a prova

Monitorar:

``` text
finalizações
scoring
performance
ranking
resultados
comunicações
DLQ
```

------------------------------------------------------------------------

# 32. Dependências entre serviços

Dependência funcional principal:

``` text
Auth
 |
 v
Candidate
 |
 v
Exam
 |
 v
Question
 |
 v
Application
 |
 v
Answer
 |
 v
ApplicationFinished
 |
 v
Orchestrator
 |
 v
Scoring
 |
 v
Performance
 |
 v
Ranking
 |
 v
Answer Key
 |
 v
Result
 |
 v
Communication
```

Auditoria e observabilidade acompanham toda a cadeia.

------------------------------------------------------------------------

# 33. Ordem exata de implementação

A ordem abaixo deve ser seguida como checklist principal.

``` text
01. Criar estrutura base e arquitetura hexagonal

02. Configurar PostgreSQL + Flyway

03. Criar tratamento global de erros

04. Criar correlationId

05. Configurar testes base

06. Criar Auth Service

07. Criar Candidate Service

08. Criar Exam Service

09. Criar Question Service

10. Criar Application Service

11. Criar Answer Service

12. Validar fluxo completo da prova

13. Criar contratos de eventos

14. Integrar Kafka

15. Publicar ApplicationStarted

16. Publicar AnswerSubmitted

17. Publicar ApplicationFinished

18. Criar contratos de comandos

19. Integrar RabbitMQ

20. Criar retry e DLQ

21. Criar Orchestrator Service

22. Criar workflow APPLICATION_POST_PROCESSING

23. Criar Scoring Service

24. Publicar ScoringFinished

25. Criar Performance Service

26. Publicar PerformanceCalculated

27. Criar Ranking Service

28. Publicar RankingUpdated

29. Criar Answer Key Service

30. Publicar AnswerKeyReleased

31. Criar Result Service

32. Criar ResultAvailable

33. Criar Communication Service

34. Criar Audit Service

35. Consolidar OpenTelemetry

36. Configurar Prometheus

37. Configurar Grafana

38. Criar dashboards

39. Criar alertas

40. Containerizar todos os serviços

41. Criar manifests Kubernetes

42. Configurar HPA

43. Configurar CI/CD

44. Criar testes integrados completos

45. Executar testes de falha

46. Executar testes de carga

47. Ajustar recursos e escalabilidade

48. Hardening

49. Homologação

50. Simulação completa

51. Congelamento de versão

52. Preparação para o Dia D
```

------------------------------------------------------------------------

# 34. Critérios para evitar retrabalho

## Não criar antes da hora

Não implementar:

``` text
Ranking
```

antes de:

``` text
Performance
```

Não implementar:

``` text
Performance
```

antes de:

``` text
Scoring
```

Não implementar:

``` text
Scoring
```

antes de:

``` text
ApplicationFinished
```

Não implementar:

``` text
Orchestrator
```

antes de:

``` text
eventos
comandos
Kafka
RabbitMQ
```

Não implementar:

``` text
Kubernetes completo
```

antes de:

``` text
Docker
serviços estáveis
health checks
configuração externa
```

------------------------------------------------------------------------

## Congelar contratos por fase

Antes de criar o consumidor de um evento:

``` text
eventType
eventVersion
payload
correlationId
aggregateId
```

devem estar definidos.

Antes de criar frontend ou consumer REST:

``` text
endpoint
request
response
status
error contract
```

devem estar definidos.

------------------------------------------------------------------------

## Não duplicar regra de negócio

Exemplo:

``` text
Application Service
```

decide se prova pode iniciar.

Não repetir a mesma decisão em:

``` text
Frontend
Auth Service
Answer Service
```

Outros serviços podem validar estado, mas a regra principal deve possuir
um dono claro.

------------------------------------------------------------------------

## Não acoplar Result Service à lógica

Result Service:

``` text
consulta
agrega
retorna
```

Não deve recalcular:

``` text
score
performance
ranking
```

------------------------------------------------------------------------

## Não usar infraestrutura como regra de negócio

Kafka e RabbitMQ transportam informações.

Eles não devem determinar:

``` text
quem pode iniciar prova
qual é a nota
qual é o ranking
```

Essas decisões pertencem aos serviços de domínio.

------------------------------------------------------------------------

# 35. Checklist final do roadmap

## Fundação

``` text
[ ] arquitetura definida
[ ] padrões definidos
[ ] banco e Flyway
[ ] testes base
```

## Core da prova

``` text
[ ] Auth
[ ] Candidate
[ ] Exam
[ ] Question
[ ] Application
[ ] Answer
[ ] fluxo ponta a ponta
```

## Assíncrono

``` text
[ ] Kafka
[ ] RabbitMQ
[ ] Retry
[ ] DLQ
[ ] Orchestrator
```

## Pós-prova

``` text
[ ] Scoring
[ ] Performance
[ ] Ranking
[ ] Answer Key
[ ] Result
[ ] Communication
```

## Operação

``` text
[ ] Audit
[ ] OpenTelemetry
[ ] Prometheus
[ ] Grafana
[ ] Alertas
```

## Infraestrutura

``` text
[ ] Docker
[ ] Kubernetes
[ ] HPA
[ ] CI/CD
```

## Qualidade

``` text
[ ] testes unitários
[ ] testes de integração
[ ] testes ponta a ponta
[ ] testes de falha
[ ] testes de carga
```

## Produção

``` text
[ ] hardening
[ ] homologação
[ ] rollback
[ ] backup
[ ] restore
[ ] pre-scaling
[ ] simulação final
```

------------------------------------------------------------------------

# Conclusão

A ordem de desenvolvimento do **Dia D Simulation** deve priorizar
primeiro o **fluxo real de execução da prova**, depois os
**processamentos assíncronos e pós-prova**, e somente depois consolidar
**infraestrutura, escala e operação**.

A sequência central é:

``` text
CORE DA PROVA
    |
    v
MENSAGERIA
    |
    v
ORQUESTRAÇÃO
    |
    v
PÓS-PROVA
    |
    v
OBSERVABILIDADE
    |
    v
INFRAESTRUTURA
    |
    v
ESCALA
    |
    v
PRODUÇÃO
```

Seguindo essa ordem, cada camada nasce sobre contratos e comportamentos
já estabilizados, reduzindo significativamente o risco de mudanças
estruturais em cascata.
