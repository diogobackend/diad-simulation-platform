# Infraestrutura e Deploy --- Dia D Simulation

Documentação específica da estratégia de **infraestrutura,
containerização, orquestração, ambientes, CI/CD, deploy, escalabilidade,
disponibilidade e operação** da plataforma **Dia D Simulation**.

------------------------------------------------------------------------

## Sumário

1.  [Visão geral](#1-visão-geral)
2.  [Objetivos de infraestrutura](#2-objetivos-de-infraestrutura)
3.  [Arquitetura de infraestrutura](#3-arquitetura-de-infraestrutura)
4.  [Componentes da plataforma](#4-componentes-da-plataforma)
5.  [Containerização com Docker](#5-containerização-com-docker)
6.  [Kubernetes](#6-kubernetes)
7.  [Namespaces e ambientes](#7-namespaces-e-ambientes)
8.  [Deployments](#8-deployments)
9.  [Services](#9-services)
10. [Ingress e entrada de tráfego](#10-ingress-e-entrada-de-tráfego)
11. [ConfigMaps e Secrets](#11-configmaps-e-secrets)
12. [Recursos e limites](#12-recursos-e-limites)
13. [Health Checks no Kubernetes](#13-health-checks-no-kubernetes)
14. [Escalabilidade horizontal](#14-escalabilidade-horizontal)
15. [Alta disponibilidade](#15-alta-disponibilidade)
16. [Banco de dados](#16-banco-de-dados)
17. [Redis](#17-redis)
18. [Kafka](#18-kafka)
19. [RabbitMQ](#19-rabbitmq)
20. [Persistência e volumes](#20-persistência-e-volumes)
21. [Rede e comunicação entre
    serviços](#21-rede-e-comunicação-entre-serviços)
22. [Segurança de infraestrutura](#22-segurança-de-infraestrutura)
23. [CI/CD](#23-cicd)
24. [Pipeline de build](#24-pipeline-de-build)
25. [Pipeline de deploy](#25-pipeline-de-deploy)
26. [Estratégia de versionamento](#26-estratégia-de-versionamento)
27. [Estratégia de deployment](#27-estratégia-de-deployment)
28. [Rollback](#28-rollback)
29. [Migrações de banco com Flyway](#29-migrações-de-banco-com-flyway)
30. [Observabilidade da
    infraestrutura](#30-observabilidade-da-infraestrutura)
31. [Escalabilidade para o Dia D](#31-escalabilidade-para-o-dia-d)
32. [Resiliência e recuperação](#32-resiliência-e-recuperação)
33. [Backup e recuperação](#33-backup-e-recuperação)
34. [Ambiente local](#34-ambiente-local)
35. [Estrutura de repositório e
    manifests](#35-estrutura-de-repositório-e-manifests)
36. [Fluxo completo de deploy](#36-fluxo-completo-de-deploy)
37. [Checklist de produção](#37-checklist-de-produção)
38. [Resumo da infraestrutura](#38-resumo-da-infraestrutura)

------------------------------------------------------------------------

# 1. Visão geral

A infraestrutura do **Dia D Simulation** deve suportar uma plataforma
distribuída composta por múltiplos serviços, comunicação REST/HTTP,
processamento assíncrono, bancos de dados e componentes de mensageria.

A estratégia base utiliza:

``` text
Docker
Kubernetes
PostgreSQL
Redis
Kafka
RabbitMQ
OpenTelemetry
Prometheus
Grafana
CI/CD
```

Os serviços da aplicação são executados como containers e orquestrados
pelo Kubernetes.

------------------------------------------------------------------------

# 2. Objetivos de infraestrutura

A infraestrutura deve priorizar:

-   alta disponibilidade;
-   escalabilidade horizontal;
-   isolamento entre serviços;
-   deploy automatizado;
-   rollback seguro;
-   observabilidade;
-   resiliência;
-   configuração externa;
-   gerenciamento seguro de secrets;
-   capacidade de suportar picos concentrados no horário da prova;
-   recuperação rápida em caso de falha.

O principal desafio operacional é o comportamento de carga do projeto.

Diferentemente de sistemas com tráfego distribuído ao longo do dia, o
**Dia D Simulation** possui eventos concentrados:

``` text
Antes da prova
    -> autenticação e consulta de alocação

Início da prova
    -> pico de login e início de aplicações

Durante a prova
    -> grande volume de salvamento de respostas

Final da prova
    -> pico de finalizações

Pós-prova
    -> correção, performance, ranking e resultados
```

------------------------------------------------------------------------

# 3. Arquitetura de infraestrutura

Visão lógica:

``` text
Internet
   |
   v
Load Balancer / Ingress
   |
   v
API Gateway
   |
   +-------------------------------+
   |                               |
   v                               v
REST Services                Internal Services
   |                               |
   +---------------+---------------+
                   |
        +----------+----------+
        |          |          |
        v          v          v
   PostgreSQL    Redis      Messaging
                            /       \
                         Kafka    RabbitMQ
```

Camada operacional:

``` text
Kubernetes Cluster
|
+-- Application Workloads
+-- Messaging
+-- Cache
+-- Observability
+-- Infrastructure Services
```

------------------------------------------------------------------------

# 4. Componentes da plataforma

Aplicações previstas:

``` text
auth-service
candidate-service
exam-service
question-service
application-service
answer-service
scoring-service
performance-service
ranking-service
answer-key-service
result-service
communication-service
orchestrator-service
audit-service
```

Componentes de infraestrutura:

``` text
PostgreSQL
Redis
Kafka
RabbitMQ
OpenTelemetry Collector
Prometheus
Grafana
```

------------------------------------------------------------------------

# 5. Containerização com Docker

Cada aplicação deve possuir sua própria imagem Docker.

Exemplo para aplicações Java/Spring Boot:

``` dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app

COPY target/application.jar application.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "application.jar"]
```

Preferencialmente, o build deve utilizar múltiplos estágios.

``` dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /build

COPY pom.xml .
COPY src ./src

RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=build /build/target/*.jar application.jar

USER 10001

ENTRYPOINT ["java", "-jar", "application.jar"]
```

Princípios:

-   imagens pequenas;
-   usuário não-root;
-   versão explícita;
-   nenhuma credencial dentro da imagem;
-   configuração externa;
-   builds reproduzíveis.

------------------------------------------------------------------------

# 6. Kubernetes

O Kubernetes é responsável por:

``` text
execução dos containers
service discovery
balanceamento interno
health checks
restart automático
escalabilidade
rolling updates
configuração
gerenciamento de secrets
```

Cada aplicação normalmente possui:

``` text
Deployment
Service
ConfigMap
Secret
HorizontalPodAutoscaler
PodDisruptionBudget
```

------------------------------------------------------------------------

# 7. Namespaces e ambientes

Ambientes devem ser isolados.

Exemplo:

``` text
dia-d-dev
dia-d-qa
dia-d-prod
dia-d-observability
```

Alternativamente, observabilidade pode possuir namespace próprio em
todos os ambientes.

Separação recomendada:

``` text
development
quality-assurance
production
```

Configurações, credenciais, quantidade de réplicas e recursos variam
entre os ambientes.

------------------------------------------------------------------------

# 8. Deployments

Exemplo conceitual:

``` yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: application-service

spec:
  replicas: 3

  selector:
    matchLabels:
      app: application-service

  template:
    metadata:
      labels:
        app: application-service

    spec:
      containers:
        - name: application-service
          image: registry/application-service:1.4.0

          ports:
            - containerPort: 8080
```

Em produção, serviços críticos devem possuir múltiplas réplicas.

Exemplo:

``` text
application-service -> 3+
answer-service      -> 5+
auth-service        -> 3+
```

Os valores definitivos devem ser definidos através de testes de carga.

------------------------------------------------------------------------

# 9. Services

Cada aplicação acessada internamente possui um Kubernetes Service.

``` yaml
apiVersion: v1
kind: Service

metadata:
  name: application-service

spec:
  selector:
    app: application-service

  ports:
    - port: 80
      targetPort: 8080
```

Comunicação interna:

``` text
http://application-service
http://candidate-service
http://exam-service
```

Os serviços não devem depender de IPs fixos de pods.

------------------------------------------------------------------------

# 10. Ingress e entrada de tráfego

O tráfego externo entra pelo Ingress ou API Gateway.

``` text
Internet
   |
   v
Load Balancer
   |
   v
Ingress Controller
   |
   v
API Gateway
   |
   v
Microservices
```

Somente aplicações que precisam receber tráfego externo devem ser
publicadas.

Serviços internos como:

``` text
scoring-service
performance-service
orchestrator-service
```

podem permanecer acessíveis apenas dentro do cluster.

TLS deve ser obrigatório para tráfego externo.

------------------------------------------------------------------------

# 11. ConfigMaps e Secrets

Configurações não sensíveis:

``` yaml
SPRING_PROFILES_ACTIVE: production
KAFKA_BOOTSTRAP_SERVERS: kafka:9092
RABBITMQ_HOST: rabbitmq
REDIS_HOST: redis
```

Devem ser armazenadas em ConfigMaps.

Credenciais:

``` text
DATABASE_PASSWORD
JWT_SECRET
RABBITMQ_PASSWORD
KAFKA_CREDENTIALS
SMTP_PASSWORD
```

devem utilizar Secrets ou solução externa de gerenciamento de segredos.

Nunca:

``` text
hardcode no código
commit no Git
Dockerfile
application.yml versionado com senha real
```

------------------------------------------------------------------------

# 12. Recursos e limites

Cada container deve possuir requests e limits.

Exemplo:

``` yaml
resources:

  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "2"
    memory: "2Gi"
```

`requests` são utilizados pelo scheduler para posicionamento dos pods.

`limits` impedem consumo descontrolado.

Os valores devem ser calibrados através de métricas e testes de carga.

------------------------------------------------------------------------

# 13. Health Checks no Kubernetes

## Liveness

``` yaml
livenessProbe:

  httpGet:
    path: /actuator/health/liveness
    port: 8080

  initialDelaySeconds: 30
  periodSeconds: 10
```

Determina se o container precisa ser reiniciado.

## Readiness

``` yaml
readinessProbe:

  httpGet:
    path: /actuator/health/readiness
    port: 8080

  initialDelaySeconds: 15
  periodSeconds: 5
```

Determina se o pod pode receber tráfego.

## Startup Probe

Pode ser utilizada em aplicações que possuem inicialização mais lenta.

``` yaml
startupProbe:

  httpGet:
    path: /actuator/health/liveness
    port: 8080

  failureThreshold: 30
  periodSeconds: 5
```

------------------------------------------------------------------------

# 14. Escalabilidade horizontal

Serviços stateless devem utilizar escalabilidade horizontal.

``` text
1 pod
  |
  v
carga aumenta
  |
  v
3 pods
  |
  v
carga aumenta
  |
  v
10 pods
```

Exemplo de HPA:

``` yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: answer-service

spec:

  minReplicas: 3
  maxReplicas: 30

  metrics:

    - type: Resource

      resource:
        name: cpu

        target:
          type: Utilization
          averageUtilization: 65
```

Para serviços assíncronos, métricas de fila também podem orientar
escalabilidade.

Exemplo:

``` text
RabbitMQ queue depth
Kafka consumer lag
```

------------------------------------------------------------------------

# 15. Alta disponibilidade

Serviços críticos não devem depender de uma única instância.

Em produção:

``` text
API Gateway.............. múltiplas réplicas
Application Service...... múltiplas réplicas
Answer Service........... múltiplas réplicas
PostgreSQL............... HA
Kafka.................... cluster
RabbitMQ................. cluster
Redis.................... replicado
```

Também devem ser considerados:

``` text
anti-affinity
multi-zone
PodDisruptionBudget
```

Exemplo:

``` yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:
  name: answer-service

spec:
  minAvailable: 2
```

------------------------------------------------------------------------

# 16. Banco de dados

O banco relacional principal é PostgreSQL.

Responsabilidades:

``` text
dados dos candidatos
exames
questões
aplicações
respostas
scores
performance
ranking
auditoria
```

A infraestrutura de produção deve considerar:

``` text
replicação
backup
monitoramento
alta disponibilidade
pool de conexões
storage persistente
```

As aplicações utilizam pool de conexão.

Exemplo:

``` text
HikariCP
```

O dimensionamento deve evitar que o aumento de pods gere quantidade
excessiva de conexões no banco.

------------------------------------------------------------------------

# 17. Redis

Redis pode ser utilizado para dados temporários e de acesso frequente.

Casos possíveis:

``` text
sessões
cache
rate limiting
dados temporários da aplicação
controle de expiração
```

Não deve ser tratado como fonte definitiva para informações que precisam
de persistência durável.

------------------------------------------------------------------------

# 18. Kafka

Kafka é utilizado para eventos de domínio.

Exemplos:

``` text
application.events
scoring.events
performance.events
ranking.events
answer-key.events
result.events
communication.events
```

Infraestrutura deve considerar:

``` text
cluster com múltiplos brokers
replication factor
partições
retenção
monitoramento
consumer groups
```

Eventos críticos devem possuir replicação adequada.

Partições permitem paralelismo.

``` text
Topic
|
+-- Partition 0
+-- Partition 1
+-- Partition 2
+-- Partition 3
```

O número de consumers efetivamente paralelos é limitado pelo número de
partições disponíveis no consumer group.

------------------------------------------------------------------------

# 19. RabbitMQ

RabbitMQ executa comandos direcionados.

Filas principais:

``` text
scoring.process.queue
performance.calculate.queue
ranking.generate.queue
answer-key.release.queue
communication.send.queue
```

Infraestrutura deve possuir:

``` text
durable queues
persistent messages
cluster
monitoramento
retry
DLQ
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

# 20. Persistência e volumes

Componentes stateful precisam de armazenamento persistente.

Exemplos:

``` text
PostgreSQL
Kafka
RabbitMQ
Prometheus
```

No Kubernetes:

``` text
PersistentVolume
PersistentVolumeClaim
StorageClass
```

Aplicações stateless não devem depender do filesystem local do pod.

Um pod pode ser destruído e recriado a qualquer momento.

------------------------------------------------------------------------

# 21. Rede e comunicação entre serviços

Fluxos externos:

``` text
Cliente
  -> HTTPS
  -> Ingress
  -> API Gateway
```

Fluxos internos:

``` text
Service A
  -> Kubernetes Service
  -> Service B
```

Mensageria:

``` text
Producer
  -> Kafka / RabbitMQ
  -> Consumer
```

A infraestrutura deve restringir comunicação desnecessária entre
workloads.

Network Policies podem ser utilizadas.

Exemplo:

``` text
answer-service
    |
    +--> PostgreSQL
    |
    +--> Kafka

Não precisa:
    |
    X--> Grafana
```

------------------------------------------------------------------------

# 22. Segurança de infraestrutura

Princípios:

-   containers executados sem root;
-   TLS;
-   secrets fora do código;
-   RBAC;
-   menor privilégio;
-   imagens verificadas;
-   dependências atualizadas;
-   portas internas não expostas externamente;
-   Network Policies;
-   logs de auditoria;
-   isolamento por namespace.

Acesso administrativo ao cluster deve ser restrito.

------------------------------------------------------------------------

# 23. CI/CD

O pipeline automatiza:

``` text
commit
  |
  v
build
  |
  v
testes
  |
  v
análise
  |
  v
imagem Docker
  |
  v
registry
  |
  v
deploy
  |
  v
health check
```

O deploy não deve depender de execução manual de comandos diretamente
nos servidores.

------------------------------------------------------------------------

# 24. Pipeline de build

Etapas sugeridas:

``` text
1. Checkout
2. Compilação
3. Testes unitários
4. Testes de integração
5. Análise estática
6. Build do artefato
7. Build da imagem Docker
8. Scan da imagem
9. Push para registry
```

Exemplo:

``` text
application-service:1.4.0
application-service:1.4.1
application-service:1.5.0
```

Evitar deploy de produção utilizando apenas:

``` text
latest
```

A imagem deve possuir versão imutável.

------------------------------------------------------------------------

# 25. Pipeline de deploy

Fluxo:

``` text
Imagem publicada
      |
      v
Atualização do manifest
      |
      v
Deploy Kubernetes
      |
      v
Rolling Update
      |
      v
Readiness
      |
      v
Smoke Test
      |
      v
Deploy concluído
```

Ambientes:

``` text
DEV
 |
 v
QA
 |
 v
PROD
```

Produção deve receber somente artefatos já validados.

------------------------------------------------------------------------

# 26. Estratégia de versionamento

Versões das aplicações devem seguir padrão consistente.

Exemplo:

``` text
1.0.0
1.1.0
1.1.1
2.0.0
```

Imagem:

``` text
registry/dia-d/application-service:1.4.0
```

Também pode ser associado o commit:

``` text
application-service:1.4.0-a83f2c1
```

Isso permite identificar exatamente qual código está executando.

------------------------------------------------------------------------

# 27. Estratégia de deployment

Estratégia inicial recomendada:

``` text
Rolling Update
```

Exemplo:

``` text
Versão 1
Pod A
Pod B
Pod C

        |
        v

Versão 2
Pod D

        |
        v

Pod A removido

        |
        v

Versão 2
Pod D
Pod E
Pod F
```

Configuração:

``` yaml
strategy:

  type: RollingUpdate

  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Para mudanças de maior risco podem ser avaliados:

``` text
Blue/Green
Canary
```

------------------------------------------------------------------------

# 28. Rollback

Caso a nova versão apresente falha:

``` text
Deploy v1.5.0
    |
    v
Erro detectado
    |
    v
Rollback
    |
    v
v1.4.0
```

Com Kubernetes:

``` bash
kubectl rollout undo deployment/application-service
```

O rollback da aplicação não significa automaticamente rollback de banco.

Por isso migrations devem ser planejadas para compatibilidade entre
versões.

------------------------------------------------------------------------

# 29. Migrações de banco com Flyway

Mudanças de schema devem ser versionadas.

Estrutura:

``` text
db/migration/

V1__create_candidate.sql
V2__create_exam.sql
V3__create_application.sql
V4__create_answer.sql
```

Regras:

-   nunca alterar migration já aplicada em produção;
-   novas mudanças geram novos arquivos;
-   migrations devem ser testadas antes do deploy;
-   mudanças destrutivas devem ser evitadas no mesmo deploy da
    aplicação.

Estratégia segura:

``` text
Deploy 1
    -> adiciona nova coluna

Deploy 2
    -> aplicação passa a utilizar nova coluna

Deploy 3
    -> remove estrutura antiga, se necessário
```

------------------------------------------------------------------------

# 30. Observabilidade da infraestrutura

O cluster deve fornecer métricas de:

``` text
CPU
memória
network
storage
pods
nodes
restarts
deployments
HPA
```

Também:

``` text
PostgreSQL
Kafka
RabbitMQ
Redis
```

Ferramentas:

``` text
Prometheus
Grafana
OpenTelemetry
```

Alertas importantes:

``` text
pod reiniciando continuamente
OOMKilled
CPU elevada
memória elevada
node indisponível
PVC próximo do limite
HPA no máximo
deployment sem réplicas disponíveis
```

------------------------------------------------------------------------

# 31. Escalabilidade para o Dia D

A carga da plataforma é previsível em horário.

Isso permite combinar:

``` text
autoscaling
+
pre-scaling
```

Antes do início da prova, serviços críticos podem ser escalados
antecipadamente.

Exemplo conceitual:

``` text
11:00
answer-service = 5 pods

11:30
answer-service = 15 pods

12:00
answer-service = 30 pods
```

Serviços prioritários durante a aplicação:

``` text
auth-service
application-service
answer-service
exam-service
question-service
```

No pós-prova:

``` text
scoring-service
performance-service
ranking-service
orchestrator-service
```

podem receber maior capacidade.

A infraestrutura deve acompanhar as diferentes fases do evento.

------------------------------------------------------------------------

# 32. Resiliência e recuperação

Falha de um pod:

``` text
Pod falha
   |
   v
Kubernetes detecta
   |
   v
Pod removido
   |
   v
Novo pod criado
```

Falha de dependência:

``` text
timeout
retry controlado
circuit breaker quando aplicável
fila
DLQ
```

Serviços síncronos não devem realizar retries agressivos que ampliem uma
indisponibilidade.

------------------------------------------------------------------------

# 33. Backup e recuperação

Dados persistentes devem possuir política de backup.

PostgreSQL:

``` text
backup completo
backup incremental / WAL quando aplicável
retenção
restore testado
```

Não basta gerar backups.

O processo de restauração deve ser testado.

Devem existir objetivos definidos:

``` text
RPO
Recovery Point Objective

RTO
Recovery Time Objective
```

Exemplo conceitual:

``` text
RPO -> quanto dado pode ser perdido

RTO -> quanto tempo pode levar para recuperar
```

Os valores definitivos dependem dos requisitos operacionais da
plataforma.

------------------------------------------------------------------------

# 34. Ambiente local

O desenvolvimento pode utilizar Docker Compose para dependências.

Exemplo:

``` yaml
services:

  postgres:
    image: postgres

  redis:
    image: redis

  kafka:
    image: kafka

  rabbitmq:
    image: rabbitmq:management
```

Fluxo local:

``` text
Aplicações Java
      |
      +--> PostgreSQL
      +--> Redis
      +--> Kafka
      +--> RabbitMQ
```

Não é necessário reproduzir toda a infraestrutura de produção no
computador do desenvolvedor.

------------------------------------------------------------------------

# 35. Estrutura de repositório e manifests

Exemplo:

``` text
application-service/
|
+-- src/
+-- pom.xml
+-- Dockerfile
+-- README.md
|
+-- k8s/
    |
    +-- base/
    |   +-- deployment.yaml
    |   +-- service.yaml
    |   +-- hpa.yaml
    |
    +-- overlays/
        |
        +-- dev/
        +-- qa/
        +-- prod/
```

Em uma abordagem GitOps, os manifests também podem permanecer em
repositório específico.

``` text
dia-d-infrastructure/
|
+-- application-service/
+-- answer-service/
+-- scoring-service/
+-- kafka/
+-- rabbitmq/
+-- observability/
```

------------------------------------------------------------------------

# 36. Fluxo completo de deploy

``` text
Developer
   |
   v
Git Push
   |
   v
Pull Request
   |
   v
CI
   |
   +--> Compile
   +--> Unit Tests
   +--> Integration Tests
   +--> Static Analysis
   |
   v
Merge
   |
   v
Docker Build
   |
   v
Container Registry
   |
   v
Deploy DEV
   |
   v
Validation
   |
   v
Deploy QA
   |
   v
Tests
   |
   v
Approval / Release
   |
   v
Deploy PROD
   |
   v
Rolling Update
   |
   v
Readiness Check
   |
   v
Smoke Test
   |
   v
Monitoring
```

------------------------------------------------------------------------

# 37. Checklist de produção

Antes do evento:

``` text
[ ] réplicas mínimas configuradas
[ ] HPA validado
[ ] testes de carga executados
[ ] PostgreSQL dimensionado
[ ] pool de conexões validado
[ ] Kafka validado
[ ] consumer lag monitorado
[ ] RabbitMQ validado
[ ] DLQs vazias
[ ] Redis operacional
[ ] dashboards disponíveis
[ ] alertas testados
[ ] backups confirmados
[ ] restore testado
[ ] health checks funcionando
[ ] certificados TLS válidos
[ ] secrets configurados
[ ] imagens corretas em produção
[ ] migrations executadas
[ ] rollback validado
```

Pouco antes da prova:

``` text
[ ] pre-scaling executado
[ ] equipe acompanhando dashboards
[ ] serviços críticos saudáveis
[ ] banco sem saturação
[ ] Kafka sem lag anormal
[ ] RabbitMQ sem backlog
[ ] nenhuma DLQ inesperada
```

------------------------------------------------------------------------

# 38. Resumo da infraestrutura

A infraestrutura do **Dia D Simulation** é baseada em workloads
containerizados e distribuídos.

``` text
Git
 |
 v
CI/CD
 |
 v
Container Registry
 |
 v
Kubernetes
 |
 +-------------------------------+
 |                               |
 v                               v
REST Services              Async Services
 |                               |
 +---------------+---------------+
                 |
     +-----------+-----------+
     |           |           |
     v           v           v
PostgreSQL     Redis      Messaging
                         /         \
                      Kafka      RabbitMQ
```

Princípios principais:

-   Docker para empacotamento;
-   Kubernetes para orquestração;
-   serviços stateless sempre que possível;
-   PostgreSQL para persistência relacional;
-   Redis para cache e dados temporários;
-   Kafka para eventos;
-   RabbitMQ para comandos;
-   configuração externa;
-   gerenciamento seguro de secrets;
-   health checks;
-   autoscaling;
-   alta disponibilidade;
-   deploy automatizado;
-   Rolling Update;
-   rollback;
-   Flyway para evolução do banco;
-   observabilidade integrada;
-   backup e recuperação;
-   pre-scaling para a janela crítica da prova.

A infraestrutura deve ser preparada para a característica central do
**Dia D Simulation**: uma grande quantidade de usuários executando as
mesmas etapas em períodos muito próximos, exigindo escalabilidade
previsível, resiliência e acompanhamento operacional em tempo real.
