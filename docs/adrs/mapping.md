# Mapeamento da Arquitetura do Código — Order Management System (OMS)
Gerado em: 2026-10-26 23:42:00

## Visão Geral do Projeto
O **Order Management System (OMS)** (`order-management-api`) é uma aplicação backend RESTful corporativa para gestão de pedidos, estoque, catálogo de produtos, clientes e controle de acesso baseado em cargos (RBAC). O sistema foi concebido como um **Monólito Modular** em Node.js com TypeScript e resolução ESM nativa, utilizando persistência relacional em MySQL 8.0 gerenciada via Prisma ORM. 

O repositório base conta com aproximadamente 40 arquivos de código fonte em `src/`, além de testes de integração com Vitest e Supertest. A feature em desenvolvimento (Sistema de Webhooks de Notificação de Pedidos) introduz uma extensão de mensageria assíncrona baseada no padrão Transactional Outbox e um processo worker desacoplado.

---

## Stack Tecnológico
| Camada | Tecnologia | Versão | Evidência (arquivo) |
| :--- | :--- | :--- | :--- |
| **Runtime** | Node.js (ESM nativo) | `>= 20.0.0` | `package.json` (`"type": "module"`, `"engines"`) |
| **Linguagem** | TypeScript | `5.6.3` | `package.json`, `tsconfig.json`, `tsconfig.build.json` |
| **Framework Web** | Express | `4.21.1` | `package.json`, `src/app.ts`, `src/server.ts` |
| **ORM / Banco de Dados** | Prisma ORM / MySQL | `5.22.0` / `8.0` | `package.json`, `prisma/schema.prisma`, `docker-compose.yml` |
| **Autenticação & Criptografia** | JWT (`jsonwebtoken`) / `bcrypt` | `9.0.2` / `5.1.1` | `package.json`, `src/modules/auth/auth.service.ts` |
| **Validação de Esquemas** | Zod | `3.23.8` | `package.json`, `src/middlewares/validate.middleware.ts` |
| **Logging Estruturado** | Pino / Pino-HTTP | `9.5.0` / `10.3.0` | `package.json`, `src/shared/logger/index.ts` |
| **Identificadores Únicos** | UUID (v4) | `11.0.3` | `package.json`, `prisma/schema.prisma` |
| **Testes Automatizados** | Vitest / Supertest | `2.1.4` / `7.0.0` | `package.json`, `vitest.config.ts`, `tests/setup.ts` |
| **Execução em Desenvolvimento** | TSX (TypeScript Execute) | `4.19.2` | `package.json` (`tsx watch --env-file=.env`) |

---

## Padrões Arquiteturais
- **Monólito Modular em Camadas:** Cada domínio de negócio reside em `src/modules/<dominio>/` estruturado rigorosamente em Controller, Service, Repository, Routes e Schemas.
- **Injeção Manual de Dependências (Composition Root):** As classes recebem suas dependências explicitamente via construtor, instanciadas centralizadamente em `src/app.ts` (`buildControllers()`), sem uso de decorators reflexivos ou containers IoC mágicos.
- **Transactional Outbox Pattern (Feature Webhooks):** Persistência atômica de eventos em tabela relacional (`webhook_outbox`) na mesma transação SQL de alteração de pedidos (`prisma.$transaction`), desacoplando a escrita de banco do despacho HTTP.
- **Dedicated Worker (Polling Consumer):** Processo Node.js independente (`src/worker.ts`) em loop de polling a cada 2 segundos, executando requisições com timeout estrito de 10s via `AbortController`.
- **Máquina de Estados Explícita:** O ciclo de vida de pedidos é governado por regras formais de transição (`src/modules/orders/order.status.ts`), impedindo alterações de status ilegais e orquestrando movimentações transacionais de estoque.
- **Envelope Padronizado de Respostas e Erros:** Todas as listagens utilizam paginação uniforme (`paginated()`) e todas as exceções de domínio estendem `AppError`, tratadas centralizadamente em `src/middlewares/error.middleware.ts`.

---

## Módulos do Sistema

| ID | Nome | Caminhos | Arquivos (estim.) | Descrição |
| :--- | :--- | :--- | :---: | :--- |
| **AUTH** | Autenticação e Credenciais | `src/modules/auth/` | 4 | Registro de contas, login, geração e validação de tokens JWT Bearer |
| **USERS** | Usuários e Controle de Acesso | `src/modules/users/` | 5 | Gestão de operadores e administradores, hashing de senhas e sanitização `PublicUser` |
| **CUSTOMERS** | Cadastro de Clientes | `src/modules/customers/` | 5 | Gestão de clientes B2B, documentos (CPF/CNPJ) e endereços estruturados em JSON |
| **PRODUCTS** | Catálogo e Inventário | `src/modules/products/` | 5 | Gestão de itens vendáveis, precificação em centavos inteiros (`priceCents`) e estoque |
| **ORDERS** | Gestão de Pedidos e Estoque | `src/modules/orders/` | 6 | Criação de pedidos, numeração sequencial atômica, máquina de estados e auditoria |
| **WEBHOOKS** | Notificações Outbound (Novo) | `src/modules/webhooks/`, `src/worker.ts` | 6 | Cadastro de endpoints, outbox, worker assíncrono, HMAC-SHA256, retries e DLQ |
| **CORE** | Infraestrutura e Servidor HTTP | `src/config/`, `src/routes/`, `src/app.ts`, `src/server.ts` | 5 | Ponto de inicialização, Composition Root, pool Prisma e configuração fail-fast de ambiente |
| **CROSSCUTTING** | Componentes Transversais | `src/middlewares/`, `src/shared/` | 9 | Middlewares de autenticação, validação, tratamento de erros, logger Pino e helpers HTTP |

---

### AUTH — Autenticação e Credenciais
- **Escopo:** Endpoints públicos e protegidos sob `/api/v1/auth`.
- **Responsabilidades:** Autenticar operadores e administradores via email/senha, emitir tokens stateless JWT assinados com `JWT_SECRET` e validar integridade de sessões.
- **Dependências de outros módulos:** `USERS` (consulta de credenciais), `CROSSCUTTING` (`AppError`, schemas Zod).
- **Tecnologias-chave:** `jsonwebtoken`, `bcrypt`, Zod.

### USERS — Usuários e Controle de Acesso
- **Escopo:** Endpoints de gestão de usuários sob `/api/v1/users`.
- **Responsabilidades:** Criação e listagem de usuários com papéis `ADMIN` e `OPERATOR`. Garantir que `passwordHash` nunca seja exposto via API através da conversão para `PublicUser`.
- **Dependências de outros módulos:** `CORE` (PrismaClient), `CROSSCUTTING` (AppError, Zod).
- **Tecnologias-chave:** Prisma ORM, bcrypt, Zod.

### CUSTOMERS — Cadastro de Clientes
- **Escopo:** Endpoints sob `/api/v1/customers`.
- **Responsabilidades:** Manutenção dos dados cadastrais dos clientes compradores e parceiros B2B (Atlas Comercial, MaxDistribuição, Nova Cargo). Validação de documento único e armazenamento de endereços em formato JSON.
- **Dependências de outros módulos:** `CORE` (PrismaClient), `CROSSCUTTING` (AppError, Zod, helpers de paginação).
- **Tecnologias-chave:** Prisma ORM (MySQL JSON type), Zod.

### PRODUCTS — Catálogo e Inventário
- **Escopo:** Endpoints sob `/api/v1/products`.
- **Responsabilidades:** Controle de catálogo, unicidade de SKU, estado ativo de produtos, quantidade de estoque (`stockQuantity`) e garantia monetária usando centavos inteiros (`priceCents`).
- **Dependências de outros módulos:** `CORE` (PrismaClient), `CROSSCUTTING` (AppError, Zod, helpers de paginação).
- **Tecnologias-chave:** Prisma ORM, Zod.

### ORDERS — Gestão de Pedidos e Estoque
- **Escopo:** Endpoints sob `/api/v1/orders`.
- **Responsabilidades:** Orquestração do ciclo de vida do pedido. Geração de código incremental (`ORD-000001`) via `OrderNumberSequence`, transições de status (`canTransition`), congelamento do preço unitário em `OrderItem`, débito/reposição transacional de estoque em `Product` e histórico imutável em `OrderStatusHistory`.
- **Dependências de outros módulos:** `CUSTOMERS`, `PRODUCTS`, `USERS`, `CORE` (PrismaClient), `CROSSCUTTING` (AppError, Zod).
- **Tecnologias-chave:** Prisma ORM (`$transaction` interativa), Zod.

### WEBHOOKS — Notificações Outbound (Em Implementação)
- **Escopo:** Endpoints `/api/v1/webhooks`, `/admin/webhooks` e processo em background `src/worker.ts`.
- **Responsabilidades:** Registro de endpoints HTTPS de clientes com filtros de eventos, enfileiramento atômico na `webhook_outbox` durante o `changeStatus`, consumo via polling de 2s, cálculo de assinatura HMAC-SHA256, disparo HTTP com timeout de 10s, retentativas exponenciais em 5 etapas, descarte em Dead Letter Queue (`webhook_dead_letter`), rotação de chaves com carência de 24h e replay administrativo auditado.
- **Dependências de outros módulos:** `ORDERS` (gatilho de mudança de status), `CUSTOMERS` (vínculo de cliente), `CORE` (PrismaClient), `CROSSCUTTING` (AppError, middlewares de auth e error, Pino Logger).
- **Tecnologias-chave:** Node.js native fetch, `node:crypto`, Prisma ORM, Zod, AbortController.

### CORE — Infraestrutura e Servidor HTTP
- **Escopo:** Raiz técnica do projeto (`src/server.ts`, `src/app.ts`, `src/config/`, `src/routes/`).
- **Responsabilidades:** Bootstrap da aplicação Express, gerenciamento fail-fast de variáveis de ambiente via Zod (`src/config/env.ts`), singleton e fábrica do PrismaClient (`src/config/database.ts`), graceful shutdown nos sinais `SIGINT`/`SIGTERM` e montagem do agregador de rotas `/api/v1` e `/health`.
- **Dependências de outros módulos:** Todos os módulos de domínio.
- **Tecnologias-chave:** Express, Prisma, Zod, Node.js process events.

### CROSSCUTTING — Componentes Transversais
- **Escopo:** `src/middlewares/` e `src/shared/`.
- **Responsabilidades:** Middlewares globais de autenticação (`authenticate`, `requireRole`), interceptador de logging Pino com geração de `X-Request-Id`, validador de esquemas Zod (`validate`), manipulador global de erros (`error.middleware.ts`), hierarquia de erros (`AppError`) e helpers padronizados de resposta paginada (`paginated()`).
- **Dependências de outros módulos:** Nenhuma (camada base reutilizada por todos os módulos).
- **Tecnologias-chave:** Pino, Pino-HTTP, jsonwebtoken, Zod.

---

## Preocupações Transversais
- **Segurança e Isolamento:** Autenticação stateless via JWT Bearer em todas as rotas de negócio. Chaves secretas geradas com entropia criptográfica e assinadas com HMAC-SHA256 nas notificações outbound. Proteção estrita RBAC com bloqueio de operadores em rotas administrativas (`ADMIN`).
- **Integridade Transacional de Dados:** Operações críticas que afetam múltiplas tabelas (ex.: baixa de estoque + atualização de pedido + inserção de histórico + inserção na outbox) são executadas sob transações ACID gerenciadas pelo Prisma (`$transaction`). Valores monetários sempre modelados como inteiros (`Int` em centavos) para evitar imprecisões de ponto flutuante IEEE 754.
- **Observabilidade e Rastreabilidade:** Tracing distribuído e rastreabilidade ponta a ponta com `X-Request-Id` gerado em cada requisição HTTP e `X-Event-Id` único por evento de webhook. Logs estruturados em formato JSON com Pino, com sanitização ativa de cabeçalhos de autorização e campos de segredo.
- **Resiliência e Desacoplamento:** Separação física de processos para garantir que travamentos de rede em endpoints de clientes externos nunca afetem a disponibilidade das APIs HTTP do OMS.

---

## Notas de Contexto
A partir da análise detalhada da reunião técnica (`TRANSCRICAO.md`) e dos documentos de design técnico elaborados (`docs/PRD.md`, `docs/RFC.md`, `docs/HLD.md` e `docs/FDD.md`), identificou-se que o vácuo de mensageria assíncrona do OMS levou clientes corporativos a praticarem polling excessivo, criando gargalos de I/O e risco de cancelamento contratual (Atlas Comercial).

A decisão colegiada optou por não adicionar brokers externos (Kafka, RabbitMQ, Redis Streams) no momento, priorizando simplicidade operacional e reuso do banco MySQL 8.0 através do padrão Transactional Outbox.

---

## Diretrizes para a Fase 2 (Identificação de ADRs)

### Ordem Sugerida de Análise dos Módulos:
1. **WEBHOOKS & ORDERS (Prioridade Máxima):** Concentra a maior densidade de decisões arquiteturais críticas (Transactional Outbox no MySQL, Worker desacoplado em polling, garantia at-least-once com `X-Event-Id`, política de retentativas com DLQ e autenticação HMAC-SHA256 com secret por endpoint).
2. **CROSSCUTTING & CORE (Prioridade Alta):** Reuso de padrões arquiteturais corporativos existentes (hierarquia `AppError`, injeção manual de dependências sem IoC externo, logs estruturados com sanitização no Pino, validação fail-fast de contratos com Zod).
3. **PRODUCTS & CUSTOMERS (Prioridade Média):** Modelo monetário em centavos inteiros (`priceCents`), validações de unicidade de SKU/documento e armazenamento de endereços em formato JSON relacional.

### Áreas com Maior Probabilidade de Decisões Arquiteturais:
- **Padrão Outbox no MySQL:** Justificativa da escolha frente a mensageria externa (Kafka/RabbitMQ/Redis) e disparo síncrono.
- **Worker em Processo Separado com Polling:** Desacoplamento de threads e isolamento de pools de conexão.
- **Política de Retry com Backoff e DLQ:** Tolerância a falhas em 5 níveis e isolamento permanente para replay administrativo.
- **Autenticação HMAC-SHA256 com Secret por Endpoint:** Padrão de segurança simétrica e rotação com 24h de tolerância.
- **Garantia At-Least-Once com Header de Idempotência:** Transferência da responsabilidade de desduplicação via `X-Event-Id`.
- **Reuso dos Padrões de Código e Arquitetura Existentes:** Alinhamento com a estrutura modular, `AppError` e middlewares globais.
