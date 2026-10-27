# ADR-006: Reuso dos Padrões de Código e Arquitetura Existentes

**Status:** Aceito
**Data:** 2026-10-26
**ADRs Relacionados:** Relacionado a [ADR-001: Padrão Outbox no MySQL para Notificações de Pedidos](./ADR-001-padrao-outbox-no-mysql.md); Relacionado a [ADR-004: Autenticação Criptográfica HMAC-SHA256 com Secret por Endpoint](./ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md)

## Contexto e Definição do Problema
O Order Management System (`order-management-api`) possui uma arquitetura em Monólito Modular madura e padronizada em camadas. Cada módulo é decomposto em Controller, Service, Repository, Routes e Schemas. O ecossistema compartilha infraestruturas transversais essenciais: hierarquia de exceções de domínio estendendo `AppError`, middleware central de tratamento de erros com envelope JSON consistente, validação estrita fail-fast com Zod, autenticação stateless JWT com RBAC via middleware `requireRole`, injeção manual de dependências no Composition Root e logging estruturado com Pino com sanitização de segredos.

Ao introduzir a nova feature de Webhooks de Notificação de Pedidos, havia o risco de engenheiros adotarem bibliotecas utilitárias externas adicionais (como frameworks de filas em memória ou loggers paralelos), ou criarem exceções aos padrões de tratamento de erro e validação, resultando em fragmentação arquitetural, aumento desnecessário de dependências no `package.json` e maior esforço de manutenção.

## Fatores de Decisão
- Preservar a consistência estilística, arquitetural e conceitual em 100% da base de código do OMS.
- Reutilizar o pipeline de middlewares existente (autenticação, autorização, validação de schemas e erro global).
- Evitar a introdução de novas dependências externas de infraestrutura ou bibliotecas utilitárias redundantes.
- Facilitar a revisão de código, testes de integração e manutenibilidade contínua pelo time de engenharia.

## Opções Consideradas
1. Reuso integral dos padrões arquiteturais e classes transversais existentes da base de código.
2. Adoção de biblioteca externa de filas ou background jobs em memória (ex.: BullMQ ou Agenda) para gerenciar o worker.
3. Criação de padrão isolado e descentralizado de tratamento de erros e respostas exclusivo para o módulo de webhooks.

## Resultado da Decisão
Opção escolhida: "Reuso integral dos padrões arquiteturais e classes transversais existentes", porque assegura que o novo módulo `src/modules/webhooks/` seja um cidadão de primeira classe perfeitamente alinhado ao restante da aplicação.

Todas as novas rotas utilizam a fábrica de roteamento `buildWebhooksRouter`, validação de parâmetros e corpos via `validate` com esquemas Zod, e proteção por tokens JWT com `authenticate`. O endpoint sensível de replay de Dead Letter Queue aproveita diretamente o middleware `requireRole('ADMIN')`. Todos os erros do novo módulo especializam classes da família `AppError` (`NotFoundError`, `ValidationError`, `ConflictError`) e utilizam códigos de erro padronizados com o prefixo `WEBHOOK_`, sendo formatados de maneira uniforme pelo `src/middlewares/error.middleware.ts`. O worker e os serviços utilizam a mesma instância configurada do Pino Logger, herdando automaticamente a sanitização de segredos e credenciais.

## Prós e Contras das Opções

### Reuso integral dos padrões existentes
- Prós: Coerência visual e estrutural absoluta; curva de aprendizado zero para a equipe; tratamento de erros, autorização e logging unificados sem código duplicado; aproveitamento dos testes existentes com Vitest/Supertest.
- Contras: Acoplamento deliberado às convenções estabelecidas pela arquitetura da aplicação existente.

### Adoção de framework externo de filas (BullMQ / Agenda)
- Prós: Recursos prontos de interface de fila e agendamento.
- Contras: Dependência obrigatória de infraestrutura adicional (Redis ou MongoDB); inconsistência arquitetural com a decisão de outbox relacional em MySQL.

### Tratamento isolado por módulo
- Prós: Independência temporária na modelagem de respostas.
- Contras: Quebra do envelope corporativo de erros; inconsistência na experiência de consumo dos desenvolvedores clientes; logs fragmentados.

## Consequências
- Efeitos positivos: O módulo de webhooks e o worker integram-se harmonicamente ao monólito modular; desenvolvedores navegam e mantêm o código com facilidade; auditoria e sanitização de segurança mantidas ativas.
- Efeitos negativos: Necessidade de seguir estritamente as regras de tipagem e convenções das camadas existentes.
- Efeitos operacionais: Nenhum custo adicional de licenciamento, monitoramento ou bibliotecas no `package.json`.

## Referências
- `src/shared/errors/app-error.ts` e `src/middlewares/error.middleware.ts` - Hierarquia de erros e interceptador global.
- `src/middlewares/auth.middleware.ts` - Middlewares de autenticação e RBAC `requireRole`.
- `src/shared/logger/index.ts` - Instância do Pino Logger com sanitização de campos sensíveis.
- `TRANSCRICAO.md:160-180` - Acordo conduzido por Bruno e Larissa estabelecendo o reuso máximo de padrões.
