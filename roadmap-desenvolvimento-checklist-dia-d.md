# Roadmap de Desenvolvimento --- Dia D Simulation

## Sumário

1.  [Fundação](#1-fundação)
2.  [Core da prova](#2-core-da-prova)
3.  [Fluxo completo da prova](#3-fluxo-completo-da-prova)
4.  [Eventos assíncronos](#4-eventos-assíncronos)
5.  [Orquestração e pós-prova](#5-orquestração-e-pós-prova)
6.  [Serviços complementares](#6-serviços-complementares)
7.  [Observabilidade](#7-observabilidade)
8.  [Infraestrutura e Deploy](#8-infraestrutura-e-deploy)
9.  [Testes e preparação final](#9-testes-e-preparação-final)

------------------------------------------------------------------------

# 1. Fundação

-   [ ] Criar repositórios
-   [ ] Configurar Java 21 + Spring Boot
-   [ ] Definir arquitetura hexagonal padrão
-   [ ] Configurar PostgreSQL
-   [ ] Configurar Flyway
-   [ ] Configurar profiles `local`, `test` e `prod`
-   [ ] Criar tratamento global de exceções
-   [ ] Criar contrato padrão de erros
-   [ ] Implementar `Correlation ID`
-   [ ] Configurar testes com JUnit + Mockito
-   [ ] Configurar Testcontainers
-   [ ] Criar Docker Compose local com PostgreSQL, Redis, Kafka e
    RabbitMQ

------------------------------------------------------------------------

# 2. Core da prova

## Auth Service

-   [ ] Criar projeto
-   [ ] Criar autenticação
-   [ ] Implementar JWT
-   [ ] Implementar Refresh Token
-   [ ] Criar `/auth/login`
-   [ ] Criar `/auth/refresh`
-   [ ] Criar `/auth/me`
-   [ ] Criar testes

## Candidate Service

-   [ ] Criar projeto
-   [ ] Criar entidade Candidate
-   [ ] Criar migrations
-   [ ] Criar cadastro
-   [ ] Criar consulta
-   [ ] Criar atualização
-   [ ] Validar documento e e-mail únicos
-   [ ] Criar testes

## Exam Service

-   [ ] Criar projeto
-   [ ] Criar entidade Exam
-   [ ] Criar status do exame
-   [ ] Criar migrations
-   [ ] Criar cadastro de exame
-   [ ] Criar consulta de exames
-   [ ] Criar regras de data e horário
-   [ ] Criar testes

## Question Service

-   [ ] Criar projeto
-   [ ] Criar Question
-   [ ] Criar Alternative
-   [ ] Criar KnowledgeArea
-   [ ] Criar migrations
-   [ ] Associar questões ao exame
-   [ ] Criar consulta de questões
-   [ ] Garantir que gabarito não seja exposto
-   [ ] Criar testes

## Application Service

-   [ ] Criar projeto
-   [ ] Criar Application
-   [ ] Criar ApplicationStatus
-   [ ] Criar School
-   [ ] Criar Room
-   [ ] Criar Allocation
-   [ ] Criar migrations
-   [ ] Criar inscrição no simulado
-   [ ] Criar alocação virtual
-   [ ] Criar início da prova
-   [ ] Criar controle de sessão
-   [ ] Criar controle de tempo
-   [ ] Criar finalização
-   [ ] Bloquear início/finalização duplicados
-   [ ] Criar testes

## Answer Service

-   [ ] Criar projeto
-   [ ] Criar Answer
-   [ ] Criar migrations
-   [ ] Criar registro de resposta
-   [ ] Criar alteração de resposta
-   [ ] Criar remoção de resposta
-   [ ] Criar consulta de respostas
-   [ ] Bloquear alterações após finalização
-   [ ] Criar testes

------------------------------------------------------------------------

# 3. Fluxo completo da prova

-   [ ] Cadastrar candidato
-   [ ] Autenticar candidato
-   [ ] Consultar exame
-   [ ] Inscrever candidato
-   [ ] Gerar escola e sala
-   [ ] Iniciar prova
-   [ ] Carregar questões
-   [ ] Registrar respostas
-   [ ] Alterar respostas
-   [ ] Recuperar respostas salvas
-   [ ] Consultar tempo restante
-   [ ] Finalizar prova
-   [ ] Bloquear alterações após finalização
-   [ ] Criar testes ponta a ponta desse fluxo

> **Só avançar após todo o fluxo acima estar funcionando.**

------------------------------------------------------------------------

# 4. Eventos assíncronos

## Kafka

-   [ ] Configurar Kafka
-   [ ] Criar contrato padrão de eventos
-   [ ] Implementar `eventId`
-   [ ] Implementar `eventVersion`
-   [ ] Propagar `correlationId`
-   [ ] Implementar Outbox
-   [ ] Implementar Inbox/idempotência
-   [ ] Publicar `ApplicationStarted`
-   [ ] Publicar `AnswerSubmitted`
-   [ ] Publicar `ApplicationFinished`
-   [ ] Criar testes de producer/consumer

## RabbitMQ

-   [ ] Configurar RabbitMQ
-   [ ] Criar exchange `dia-d.commands`
-   [ ] Criar exchange de retry
-   [ ] Criar DLX
-   [ ] Criar contrato padrão de comandos
-   [ ] Implementar `commandId`
-   [ ] Criar retry
-   [ ] Criar DLQ
-   [ ] Criar testes de comandos

------------------------------------------------------------------------

# 5. Orquestração e pós-prova

## Orchestrator Service

-   [ ] Criar projeto
-   [ ] Criar `orchestrator_db`
-   [ ] Criar entidade Workflow
-   [ ] Criar controle de etapas
-   [ ] Consumir `ApplicationFinished`
-   [ ] Criar workflow pós-prova
-   [ ] Criar recuperação de workflow
-   [ ] Criar idempotência
-   [ ] Criar testes

## Scoring Service

-   [ ] Criar projeto
-   [ ] Criar Score
-   [ ] Criar engine de correção
-   [ ] Consumir `ProcessScoring`
-   [ ] Persistir resultado
-   [ ] Publicar `ScoringFinished`
-   [ ] Criar consulta de score
-   [ ] Criar testes

## Performance Service

-   [ ] Criar projeto
-   [ ] Criar Performance
-   [ ] Criar desempenho por área
-   [ ] Criar percentil
-   [ ] Consumir comando de cálculo
-   [ ] Publicar `PerformanceCalculated`
-   [ ] Criar consulta
-   [ ] Criar testes

## Ranking Service

-   [ ] Criar projeto
-   [ ] Criar Ranking
-   [ ] Criar cálculo de classificação
-   [ ] Consumir comando de geração
-   [ ] Publicar `RankingUpdated`
-   [ ] Criar ranking geral
-   [ ] Criar posição individual
-   [ ] Criar testes

## Answer Key Service

-   [ ] Criar projeto
-   [ ] Criar AnswerKey
-   [ ] Criar versionamento
-   [ ] Criar publicação do gabarito
-   [ ] Publicar `AnswerKeyReleased`
-   [ ] Criar consulta
-   [ ] Criar testes

## Result Service

-   [ ] Criar projeto
-   [ ] Consolidar Score
-   [ ] Consolidar Performance
-   [ ] Consolidar Ranking
-   [ ] Criar resultado final
-   [ ] Criar status `PROCESSING`
-   [ ] Criar status `AVAILABLE`
-   [ ] Publicar `ResultAvailable`
-   [ ] Criar testes

------------------------------------------------------------------------

# 6. Serviços complementares

## Communication Service

-   [ ] Criar projeto
-   [ ] Criar envio de e-mail
-   [ ] Consumir `SendCommunication`
-   [ ] Criar retry
-   [ ] Criar DLQ
-   [ ] Publicar `CommunicationSent`
-   [ ] Publicar `CommunicationFailed`
-   [ ] Criar histórico
-   [ ] Criar testes

## Audit Service

-   [ ] Criar projeto
-   [ ] Consumir eventos auditáveis
-   [ ] Persistir auditoria
-   [ ] Consultar por `correlationId`
-   [ ] Consultar por `aggregateId`
-   [ ] Criar testes

------------------------------------------------------------------------

# 7. Observabilidade

-   [ ] Configurar logs estruturados
-   [ ] Configurar OpenTelemetry
-   [ ] Propagar `traceId`
-   [ ] Propagar `spanId`
-   [ ] Instrumentar REST/HTTP
-   [ ] Instrumentar Kafka
-   [ ] Instrumentar RabbitMQ
-   [ ] Instrumentar banco
-   [ ] Instrumentar Orchestrator
-   [ ] Configurar Prometheus
-   [ ] Configurar Grafana
-   [ ] Criar dashboard geral
-   [ ] Criar dashboard do Dia D
-   [ ] Criar dashboard Kafka
-   [ ] Criar dashboard RabbitMQ
-   [ ] Criar dashboard dos workflows
-   [ ] Criar alertas

------------------------------------------------------------------------

# 8. Infraestrutura e Deploy

## Docker

-   [ ] Criar Dockerfile de cada serviço
-   [ ] Configurar usuário não-root
-   [ ] Configurar variáveis de ambiente
-   [ ] Configurar health checks
-   [ ] Gerar imagens versionadas

## Kubernetes

-   [ ] Criar namespaces
-   [ ] Criar ConfigMaps
-   [ ] Criar Secrets
-   [ ] Criar Deployments
-   [ ] Criar Services
-   [ ] Criar Ingress
-   [ ] Criar readiness probes
-   [ ] Criar liveness probes
-   [ ] Criar resource requests/limits
-   [ ] Criar HPA
-   [ ] Criar PodDisruptionBudget
-   [ ] Criar Network Policies

## CI/CD

-   [ ] Configurar build
-   [ ] Executar testes automaticamente
-   [ ] Criar imagem Docker
-   [ ] Executar scan
-   [ ] Publicar imagem
-   [ ] Deploy DEV
-   [ ] Deploy QA
-   [ ] Deploy PROD
-   [ ] Configurar smoke tests
-   [ ] Configurar rollback

------------------------------------------------------------------------

# 9. Testes e preparação final

-   [ ] Testar fluxo completo candidato → resultado
-   [ ] Testar fluxo Kafka
-   [ ] Testar fluxo RabbitMQ
-   [ ] Testar Orchestrator
-   [ ] Testar retries
-   [ ] Testar DLQs
-   [ ] Testar idempotência
-   [ ] Testar recuperação de workflow
-   [ ] Testar indisponibilidade de serviços
-   [ ] Testar indisponibilidade do banco
-   [ ] Testar indisponibilidade Kafka
-   [ ] Testar indisponibilidade RabbitMQ
-   [ ] Executar testes de carga
-   [ ] Testar pico de login
-   [ ] Testar pico de início de prova
-   [ ] Testar volume de respostas
-   [ ] Testar pico de finalização
-   [ ] Testar processamento pós-prova
-   [ ] Ajustar réplicas
-   [ ] Ajustar HPA
-   [ ] Ajustar pool de conexões
-   [ ] Ajustar partições Kafka
-   [ ] Ajustar consumers
-   [ ] Validar backup
-   [ ] Validar restore
-   [ ] Validar rollback
-   [ ] Executar simulação completa
-   [ ] Corrigir pendências
-   [ ] Congelar versão
-   [ ] Fazer pre-scaling
-   [ ] Executar checklist final do Dia D
