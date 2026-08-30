# Arquitetura - Dia D Simulation Platform

## Sumário

1. [Objetivo](#1---objetivo)
2. [Princípios arquiteturais](#2---princípios-arquiteturais)
3. [Visão arquitetural](#3---visão-arquitetural)
4. [Bounded Contexts](#4---bounded-contexts)
5. [Fluxo arquitetural da aplicação](#5---fluxo-arquitetural-da-aplicação)
6. [Arquitetura interna dos microservices](#6---arquitetura-interna-dos-microservices)
7. [Autoridade e propriedade de domínio](#7---autoridade-e-propriedade-de-domínio)
8. [Dependências entre serviços](#8---dependências-entre-serviços)
9. [Comunicação síncrona e assíncrona](#9---comunicação-síncrona-e-assíncrona)
10. [Consistência distribuída](#10---consistência-distribuída)
11. [Controle de tempo](#11---controle-de-tempo)
12. [Estado e ciclo de vida da aplicação](#12---estado-e-ciclo-de-vida-da-aplicação)
13. [Isolamento de dados](#13---isolamento-de-dados)
14. [Idempotência](#14---idempotência)
15. [Resiliência e tratamento de falhas](#15---resiliência-e-tratamento-de-falhas)
16. [Escalabilidade e concorrência](#16---escalabilidade-e-concorrência)
17. [Evolução das regras de prova](#17---evolução-das-regras-de-prova)
18. [Fronteiras de segurança](#18---fronteiras-de-segurança)
19. [Observabilidade arquitetural](#19---observabilidade-arquitetural)
20. [Decisões arquiteturais](#20---decisões-arquiteturais)

---

## 1 - Objetivo

Este documento descreve as decisões arquiteturais da **Dia D Simulation Platform**.

O objetivo não é repetir a apresentação funcional, tecnologias ou responsabilidades resumidas no `README.md`, mas detalhar como os componentes da plataforma devem se relacionar e quais regras arquiteturais deverão orientar sua implementação.

A arquitetura deve permitir que diferentes tipos de provas sejam executados sobre a mesma plataforma sem que o domínio fique acoplado a uma instituição, banca ou exame específico.

Além da separação funcional, existe uma preocupação particular com o comportamento da plataforma durante aplicações simultâneas.

Em um mesmo período, milhares de candidatos poderão estar:

- entrando em salas virtuais;
- iniciando sessões;
- recebendo questões;
- enviando e alterando respostas;
- produzindo eventos de segurança;
- consultando o tempo restante;
- finalizando provas;
- tendo resultados processados;
- recebendo relatórios e classificações.

Por isso, a arquitetura deverá ser preparada para preservar regras críticas mesmo diante de concorrência elevada, falhas parciais e processamento distribuído.

---

## 2 - Princípios arquiteturais

A evolução da plataforma deverá respeitar os seguintes princípios.

### 2.1 - Domínio como núcleo

As regras de negócio não devem depender de frameworks ou mecanismos de infraestrutura.

O domínio não deverá conhecer diretamente:

```text
Spring
JPA
Kafka
RabbitMQ
Redis
HTTP
PostgreSQL/MySQL
APIs externas
```

Essas tecnologias são detalhes externos ao domínio.

### 2.2 - Responsabilidade por contexto

Cada serviço deverá possuir uma responsabilidade de negócio claramente delimitada.

A criação de um microservice deverá representar uma capacidade relevante da plataforma, e não simplesmente uma tabela ou entidade diferente.

### 2.3 - Autoridade única

Cada informação crítica deverá possuir um único contexto responsável por decidir seu estado.

Exemplos:

```text
Application Service → estado da aplicação
Answer Service      → respostas
Scoring Service     → pontuação
Performance Service → análise de desempenho
Ranking Service     → classificação
Security Service    → eventos de integridade
```

### 2.4 - Contratos explícitos

A comunicação entre contextos deverá ocorrer por contratos conhecidos:

```text
REST
Eventos
Mensagens
```

Nenhum serviço poderá utilizar diretamente a implementação interna ou o banco de outro serviço.

### 2.5 - Falhas parciais são esperadas

A indisponibilidade temporária de um serviço não deverá necessariamente provocar falha global da aplicação.

O sistema deverá ser projetado considerando:

```text
timeout
retry
mensagem duplicada
consumer indisponível
serviço reiniciado
pod substituído
conexão perdida
evento processado novamente
```

---

## 3 - Visão arquitetural

## 3.1 Visão Geral
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/fea41e8e-d95b-493a-a7e8-34820d0c5f5d" />

A entrada externa da plataforma será centralizada no API Gateway.

Os microservices permanecem isolados atrás dessa fronteira.

```text
┌─────────────────┐
│    diad-web     │
│   React / TS    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   API Gateway   │
└────────┬────────┘
         │
         ▼
┌───────────────────────────────────────────────┐
│                Microservices                  │
│                                               │
│ Auth          Candidate      Exam             │
│ Application   Question       Answer           │
│ Security      Scoring        Performance      │
│ Ranking       Communication                   │
└───────────────────────┬───────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────┐
│         Dados / Mensageria / Cache            │
│                                               │
│ Databases │ Kafka │ RabbitMQ │ Redis          │
└───────────────────────────────────────────────┘
```

O Gateway será uma fronteira técnica.

Ele poderá executar tarefas como:

```text
validação de autenticação
roteamento
rate limiting
CORS
propagação de correlationId
filtros técnicos
```

O Gateway não poderá decidir regras pertencentes aos domínios.

Exemplos de decisões que não pertencem ao Gateway:

```text
o candidato pode iniciar a prova?
o tempo mínimo foi cumprido?
a sessão pode ser finalizada?
o candidato pode acessar o gabarito?
a resposta ainda pode ser alterada?
```

---

## 4 - Bounded Contexts

A plataforma será dividida em contextos de negócio.

A separação existe para reduzir acoplamento semântico entre capacidades diferentes.

```text
                    DIA D
                      │
       ┌──────────────┼──────────────┐
       │              │              │
       ▼              ▼              ▼
  IDENTIDADE       APLICAÇÃO      CONTEÚDO
       │              │              │
   Auth           Application      Question
 Candidate           Exam
                      │
                      ▼
                    Answer
                      │
            ┌─────────┴─────────┐
            ▼                   ▼
         ANÁLISE            INTEGRIDADE
            │                   │
         Scoring             Security
       Performance
         Ranking
            │
            ▼
       COMUNICAÇÃO
            │
     Communication
```

### 4.1 - Identidade

Representa quem utiliza a plataforma.

Autenticação e dados do candidato permanecem separados das regras operacionais da prova.

### 4.2 - Aplicação

Representa o núcleo operacional do dia da prova.

É responsável por coordenar as condições necessárias para uma sessão existir e evoluir.

### 4.3 - Conteúdo

Representa as questões e suas classificações acadêmicas.

O conteúdo existe independentemente de uma aplicação específica.

### 4.4 - Análise

Recebe informações produzidas pela aplicação e transforma esses dados em:

```text
nota
desempenho
evolução
classificação
```

### 4.5 - Integridade

Representa eventos relacionados ao comportamento da sessão.

Não deve ser confundido com autenticação.

### 4.6 - Comunicação

Transforma acontecimentos do domínio em comunicações externas.

O envio de uma notificação não deverá fazer parte da transação crítica que finaliza uma prova.

---

## 5 - Fluxo arquitetural da aplicação

O `diad-application-service` representa o centro operacional da experiência.

Ele não precisa executar todas as operações, mas mantém a autoridade sobre o estado da aplicação.

```text
Candidato
    │
    ▼
Inscrição
    │
    ▼
REGISTERED
    │
    ▼
Sala de espera
    │
    ▼
WAITING
    │
    ▼
Horário autorizado
    │
    ▼
IN_PROGRESS
    │
    ├────────────► Answer Service
    │
    ├────────────► Security Service
    │
    └────────────► Controle de tempo
    │
    ▼
Finalização
    │
    ▼
FINISHED
    │
    ▼
ApplicationFinished
    │
    ├────────────► Scoring
    ├────────────► Performance
    ├────────────► Ranking
    └────────────► Communication
```

A finalização deverá representar uma transição definitiva.

Depois de:

```text
IN_PROGRESS → FINISHED
```

o candidato não poderá continuar modificando respostas.

O processamento posterior poderá continuar de forma assíncrona sem manter a sessão de prova aberta.

---

## 6 - Arquitetura interna dos microservices

Cada serviço deverá utilizar Ports and Adapters.

O desenho básico será:

```text
             ENTRADAS
                │
      ┌─────────┴─────────┐
      ▼                   ▼
REST Controller      Message Consumer
      │                   │
      └─────────┬─────────┘
                ▼
           INPUT PORT
                │
                ▼
             USE CASE
                │
                ▼
              DOMAIN
                │
                ▼
           OUTPUT PORT
      ┌─────────┼─────────┐
      ▼         ▼         ▼
 Repository   Producer   Client
 Adapter      Adapter    Adapter
```

### 6.1 - Core

O `core` representa a parte independente da aplicação.

```text
core/
├── common/
├── domain/
│   ├── model/
│   ├── exception/
│   └── valueobject/
├── port/
│   ├── input/
│   └── output/
└── usecase/
```

### 6.2 - Adapters

Os adapters traduzem tecnologias externas para contratos compreendidos pelo core.

```text
app/
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

### 6.3 - Direção das dependências

A dependência deverá apontar para dentro.

```text
Framework / Infra
       │
       ▼
    Adapters
       │
       ▼
     Ports
       │
       ▼
   Use Cases
       │
       ▼
    Domain
```

O domínio não poderá inverter essa relação.

---

## 7 - Autoridade e propriedade de domínio

A plataforma deverá evitar situações nas quais dois serviços acreditam possuir autoridade sobre o mesmo estado.

```text
┌───────────────────────┬────────────────────────────┐
│ Contexto              │ Autoridade                 │
├───────────────────────┼────────────────────────────┤
│ Application Service   │ sessão e aplicação         │
│ Answer Service        │ respostas                  │
│ Scoring Service       │ pontuação                  │
│ Performance Service   │ análise de desempenho      │
│ Ranking Service       │ classificação              │
│ Security Service      │ eventos de integridade     │
│ Communication Service │ entrega de comunicação     │
└───────────────────────┴────────────────────────────┘
```

Exemplo:

O `scoring-service` poderá conhecer o tempo que o candidato permaneceu na prova caso essa informação seja necessária para alguma política de cálculo.

Entretanto, ele não poderá alterar o tempo de permanência ou decidir se a sessão estava válida.

Essa autoridade continua pertencendo ao contexto de aplicação.

---

## 8 - Dependências entre serviços

Serviços não poderão compartilhar banco de dados.

### Incorreto

```text
Application Service
        │
        ▼
   scoring_db
```

### Correto

```text
Application Service
        │
        ├──── REST ────► Outro serviço
        │
        └──── EVENT ───► Kafka
```

Cada serviço controla seu próprio estado:

```text
Application Service ───► application_db

Answer Service      ───► answer_db

Scoring Service     ───► scoring_db

Performance Service ───► performance_db

Ranking Service     ───► ranking_db
```

Um serviço também não deverá importar diretamente classes internas de outro serviço.

Contratos compartilhados devem ser limitados aos elementos realmente necessários para interoperabilidade.

---

## 9 - Comunicação síncrona e assíncrona

A escolha entre comunicação síncrona e assíncrona deverá considerar a necessidade real de resposta.

### 9.1 - Síncrona

Indicada quando o solicitante precisa do resultado imediatamente.

```text
Client
  │
  ▼
Gateway
  │
  ▼
Service
  │
  ▼
Response
```

Exemplos:

```text
consultar aplicação
iniciar sessão
consultar questão
registrar resposta
consultar resultado já processado
```

### 9.2 - Assíncrona

Indicada para comunicar fatos já ocorridos.

```text
Application Service
        │
        ▼
ApplicationFinished
        │
        ▼
      Kafka
     /  |   \
    /   |    \
   ▼    ▼     ▼
Score  Audit  Outros consumidores
```

O produtor publica um fato.

Ele não deve precisar conhecer todos os consumidores.

Isso reduz acoplamento entre serviços.

### 9.3 - Evento x comando

Eventos representam fatos:

```text
ApplicationFinished
ScoreCalculated
PerformanceReportGenerated
```

Comandos representam uma intenção:

```text
SendExamReminder
SendResultNotification
GenerateDocument
```

Kafka deverá ser priorizado para propagação de eventos de domínio.

RabbitMQ poderá ser utilizado para filas de trabalho, processamento dirigido e retry/DLQ.

---

## 10 - Consistência distribuída

A plataforma não deverá depender de transações ACID envolvendo bancos de vários serviços.

Cada contexto controla sua própria transação local.

Exemplo de finalização:

```text
Application Service

┌───────────────────────────────┐
│        TRANSACTION            │
│                               │
│ Application = FINISHED        │
│                               │
│ OutboxEvent =                 │
│ ApplicationFinished           │
└───────────────────────────────┘
```

Depois:

```text
Outbox
   │
   ▼
Publisher
   │
   ▼
Kafka
```

O estado local e a intenção de publicação são persistidos na mesma transação.

Isso reduz a possibilidade de ocorrer:

```text
Application = FINISHED

mas

ApplicationFinished nunca publicado
```

Entre serviços, a plataforma trabalhará com consistência eventual.

---

## 11 - Controle de tempo

O controle de tempo é uma das decisões mais importantes da plataforma.

O navegador do candidato não poderá ser autoridade para:

```text
início
término
permanência mínima
expiração
liberação do gabarito
```

O backend deverá manter informações como:

```text
serverTime
startedAt
endsAt
minimumStayUntil
sessionExpiresAt
answerKeyAvailableAt
```

Fluxo simplificado:

```text
Backend
   │
   │ serverTime
   │ endsAt
   ▼
Frontend
   │
   ▼
Exibe cronômetro
```

O cronômetro exibido pelo React é uma representação visual.

A decisão continua no servidor.

```text
Frontend:
00:00:01 restante

Backend:
currentTime >= endsAt
        │
        ▼
Finalizar sessão
```

Alterar o relógio local do computador não deverá alterar nenhuma regra da aplicação.

---

## 12 - Estado e ciclo de vida da aplicação

O estado da aplicação deverá ser modelado explicitamente.

Exemplo:

```text
REGISTERED
    │
    ▼
WAITING
    │
    ▼
AVAILABLE
    │
    ▼
IN_PROGRESS
    │
    ├────────► FINISHED
    │
    ├────────► EXPIRED
    │
    └────────► DISQUALIFIED
```

Nem toda transição será permitida.

Exemplo inválido:

```text
FINISHED → IN_PROGRESS
```

Exemplo válido:

```text
WAITING → AVAILABLE
AVAILABLE → IN_PROGRESS
IN_PROGRESS → FINISHED
```

A validação das transições deverá permanecer no domínio.

Controller ou repository não deverão decidir mudanças de estado.

---

## 13 - Isolamento de dados

Será utilizado o princípio de banco por serviço.

Não serão permitidos JOINs entre bancos de contextos diferentes.

Quando um serviço precisar de dados pertencentes a outro contexto, deverá utilizar contrato explícito.

### Consulta imediata

```text
Service A
   │
   │ REST
   ▼
Service B
```

### Replicação orientada por evento

```text
Service A
   │
   ▼
DomainEvent
   │
   ▼
Kafka
   │
   ▼
Service B
   │
   ▼
Representação local
```

A segunda estratégia poderá ser utilizada quando consultas frequentes justificarem uma projeção local.

Essa representação será derivada.

A autoridade continuará pertencendo ao serviço de origem.

---

## 14 - Idempotência

Em sistemas distribuídos, a mesma requisição ou mensagem poderá ser recebida mais de uma vez.

Operações críticas deverão ser idempotentes.

### Eventos

Todo evento relevante deverá possuir `eventId`.

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "ApplicationFinished",
  "aggregateId": "application-id",
  "correlationId": "correlation-id",
  "occurredAt": "2026-08-30T15:30:00Z",
  "version": 1
}
```

Consumer:

```text
Evento recebido
      │
      ▼
eventId já existe?
      │
   ┌──┴──┐
   │     │
  SIM   NÃO
   │     │
   ▼     ▼
Ignora Processa
```

O Inbox Pattern poderá ser utilizado nos fluxos em que a duplicação gere impacto relevante.

### Requisições

Operações sensíveis, como finalização de prova, também deverão considerar repetição.

Duas chamadas consecutivas de finalização não poderão produzir duas finalizações independentes.

---

## 15 - Resiliência e tratamento de falhas

Falhas parciais serão tratadas como parte normal da operação.

Exemplo:

```text
Application
    │
    ▼
ApplicationFinished
    │
    ▼
Kafka
    │
    ▼
Scoring
    │
    X
  falha
```

A falha no scoring não deverá fazer a aplicação retornar para:

```text
IN_PROGRESS
```

Ela continuará:

```text
FINISHED
```

O cálculo poderá ser repetido posteriormente.

### Retry

Retry deverá ser utilizado apenas para falhas potencialmente transitórias.

Exemplos:

```text
timeout
indisponibilidade temporária
falha momentânea de rede
```

### DLQ

Falhas não recuperadas após a política de retry deverão ser encaminhadas para DLQ quando o fluxo utilizar fila compatível com essa estratégia.

```text
Processamento
     │
     X
     │
     ▼
   Retry
     │
     X
     │
     ▼
    DLQ
```

Erros de negócio não deverão ser reprocessados indefinidamente.

---

## 16 - Escalabilidade e concorrência

A carga não será distribuída igualmente entre os serviços.

Durante uma prova, `answer-service` e `application-service` poderão receber volume significativamente maior que serviços administrativos.

Por isso, cada serviço deverá ser escalável de forma independente.

```text
                Load Balancer
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
    Answer #1    Answer #2    Answer #3
```

O estado necessário para continuar uma sessão não poderá depender exclusivamente da memória de uma instância.

Exemplo:

```text
Request 1
   │
   ▼
Application Pod #1

Request 2
   │
   ▼
Application Pod #4
```

As duas requisições deverão observar estado consistente.

### Concorrência de respostas

O sistema deverá considerar situações como:

```text
duplo clique
retry do frontend
reenvio após timeout
duas requisições concorrentes
```

Operações deverão possuir mecanismos adequados de consistência, versionamento ou idempotência de acordo com o caso.

### Pico de início

O início de uma aplicação poderá produzir um padrão conhecido como **thundering herd**:

```text
13:30:00

10.000 candidatos
        │
        ▼
tentando iniciar simultaneamente
```

A arquitetura deverá ser validada posteriormente com testes de carga específicos para:

```text
abertura da prova
salvamento simultâneo de respostas
autosave
finalização próxima ao horário limite
consulta massiva de resultado
```

---

## 17 - Evolução das regras de prova

O núcleo não deverá possuir condicionais espalhadas por tipo de exame.

Exemplo a evitar:

```kotlin
if (examType == ENEM) {
    ...
} else if (examType == OAB) {
    ...
} else if (examType == CEBRASPE) {
    ...
}
```

O comportamento variável deverá ser representado por políticas e estratégias.

```text
                  Exam
                   │
                   ▼
          Application Policies
        ┌──────────┼──────────┐
        ▼          ▼          ▼
      Start       Exit     MinimumStay
      Policy      Policy      Policy
                   │
                   ▼
             ScoringPolicy
```

Exemplo:

```text
ENEM
├── EnemApplicationPolicy
├── EnemMinimumStayPolicy
└── TriScoringPolicy

CEBRASPE
├── CebraspeApplicationPolicy
├── CebraspeExitPolicy
└── NegativeScoringPolicy
```

Quando possível, diferenças simples deverão ser configuráveis por dados.

Código específico deverá existir apenas quando o comportamento realmente exigir implementação diferente.

---

## 18 - Fronteiras de segurança

Autenticação e integridade são problemas diferentes.

```text
Auth Service
    │
    └── Quem é o usuário?

Security Service
    │
    └── O que ocorreu na sessão?
```

Eventos de integridade poderão incluir:

```text
TAB_CHANGED
WINDOW_BLURRED
FULLSCREEN_EXIT
COPY_ATTEMPT
PASTE_ATTEMPT
MULTIPLE_LOGIN
IP_CHANGED
SESSION_DUPLICATED
CONNECTION_LOST
```

Um evento não deverá significar automaticamente fraude.

```text
Security Event
      │
      ▼
Security Policy
      │
 ┌────┼───────────┐
 ▼    ▼           ▼
Log  Warning   Disqualification
```

A consequência dependerá das regras configuradas para a aplicação.

A plataforma deverá assumir que um ambiente web não consegue impedir completamente o uso de outro dispositivo ou outros meios externos.

O objetivo técnico será detectar, registrar, dificultar e aplicar políticas possíveis sem tratar mecanismos de frontend como garantia absoluta de inviolabilidade.

---

## 19 - Observabilidade arquitetural

Uma operação distribuída deverá ser rastreável de ponta a ponta.

Exemplo:

```text
POST /applications/{id}/finish

correlationId = ABC-999
```

Propagação:

```text
Gateway
  │
  │ ABC-999
  ▼
Application Service
  │
  │ ABC-999
  ▼
Kafka
  │
  │ ABC-999
  ▼
Scoring Service
  │
  │ ABC-999
  ▼
Performance Service
```

Os contratos deverão preservar, quando aplicável:

```text
eventId
correlationId
traceId
spanId
aggregateId
occurredAt
```

Isso permitirá investigar fluxos que atravessam vários serviços sem depender exclusivamente de logs isolados.

A documentação específica de observabilidade deverá definir métricas, dashboards, tracing, logs e alertas em maior detalhe.

---

## 20 - Decisões arquiteturais

As principais decisões serão registradas como ADRs.

### ADR-001 - Microservices por capacidade de negócio

**Decisão**

Utilizar serviços independentes delimitados por responsabilidade de domínio.

**Motivação**

A plataforma possui capacidades com ciclos de evolução e perfis de carga diferentes.

**Consequência**

Existe maior complexidade operacional e de consistência distribuída em comparação com uma aplicação monolítica.

---

### ADR-002 - Arquitetura Hexagonal

**Decisão**

Aplicar Ports and Adapters dentro dos serviços.

**Motivação**

Proteger regras de negócio contra dependências de frameworks, persistência e mensageria.

**Consequência**

A implementação possui mais contratos e mapeamentos, porém mantém separação arquitetural explícita.

---

### ADR-003 - Database per Service

**Decisão**

Cada serviço será proprietário do próprio armazenamento.

**Motivação**

Evitar compartilhamento de persistência e acoplamento entre contextos.

**Consequência**

Consultas que atravessam domínios deverão utilizar APIs, eventos ou projeções locais.

---

### ADR-004 - Event-driven Architecture

**Decisão**

Mudanças relevantes de estado serão propagadas através de eventos.

**Motivação**

Reduzir dependências síncronas e permitir processamento independente.

**Consequência**

A plataforma deverá lidar explicitamente com consistência eventual, duplicação e rastreabilidade.

---

### ADR-005 - Backend como autoridade de tempo

**Decisão**

O servidor será a autoridade para todas as regras temporais da aplicação.

**Motivação**

O relógio do dispositivo do candidato não é uma fonte confiável.

**Consequência**

O frontend representa o cronômetro, mas não determina validade temporal.

---

### ADR-006 - Consistência eventual entre serviços

**Decisão**

Não utilizar transações distribuídas entre bancos de diferentes microservices.

**Motivação**

Evitar alto acoplamento e fragilidade operacional.

**Consequência**

Algumas informações poderão levar um pequeno intervalo para convergir entre contextos.

---

### ADR-007 - Outbox e idempotência

**Decisão**

Fluxos críticos de publicação utilizarão Outbox e consumidores deverão tratar duplicação.

**Motivação**

Reduzir perda de eventos e efeitos colaterais duplicados.

**Consequência**

Existe persistência e processamento adicional para controle dos eventos.

---

### ADR-008 - Regras específicas através de Policies e Strategies

**Decisão**

Comportamentos variáveis entre provas serão implementados através de configuração, políticas e estratégias.

**Motivação**

Evitar acoplamento da plataforma a ENEM, OAB, concursos ou bancas específicas.

**Consequência**

Novos exames poderão ser adicionados com menor impacto no núcleo.

---

### ADR-009 - Processamento pós-prova desacoplado

**Decisão**

Pontuação, análise de desempenho, ranking e comunicação deverão ser desacoplados da transação que finaliza a aplicação sempre que possível.

**Motivação**

A finalização da prova é uma operação crítica e não deverá depender da disponibilidade imediata de processos posteriores.

Fluxo:

```text
Finalizar prova
      │
      ▼
FINISHED
      │
      ▼
ApplicationFinished
      │
      ▼
     Kafka
   ┌──┼──────────┐
   ▼  ▼          ▼
Score Performance Ranking
                  │
                  ▼
            Communication
```

**Consequência**

O candidato poderá visualizar temporariamente um estado como:

```text
Prova finalizada.
Resultado em processamento.
```

sem que isso represente falha na aplicação.

---

## Visão consolidada

A arquitetura pode ser representada de forma simplificada por:

```text
                    ┌─────────────┐
                    │    WEB      │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ API GATEWAY │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         APPLICATION    QUESTION      ANSWER
              │                         │
              └────────────┬────────────┘
                           │
                           ▼
                        EVENTS
                           │
                           ▼
                         KAFKA
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           SCORING    PERFORMANCE    RANKING
                           │
                           ▼
                    COMMUNICATION
```

A regra arquitetural central da **Dia D Simulation Platform** será:

> Cada serviço possui autoridade apenas sobre seu próprio contexto, protege seu domínio de detalhes externos e comunica mudanças relevantes através de contratos explícitos.

Essa regra deverá orientar novas funcionalidades e novos serviços adicionados à plataforma.
