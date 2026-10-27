# ADR-001: Padrão Outbox no MySQL para Notificações de Pedidos

**Status:** Aceito
**Data:** 2026-10-26
**ADRs Relacionados:** Usado por [ADR-002: Worker Desacoplado em Processo Separado com Polling](./ADR-002-worker-separado-em-polling.md); Usado por [ADR-005: Garantia de Entrega At-Least-Once com Header de Idempotência](./ADR-005-garantia-at-least-once-idempotencia.md); Relacionado a [ADR-006: Reuso dos Padrões de Código e Arquitetura Existentes](./ADR-006-reuso-padroes-arquiteturais-existentes.md)

## Contexto e Definição do Problema
O Order Management System (OMS) opera a gestão de pedidos com máquina de estados rigorosa e controle transacional de estoque. Clientes corporativos B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) necessitam de notificações em tempo real sobre transições de status de seus pedidos (`PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`). Atualmente, esses clientes praticam polling contínuo via `GET /orders`, degradando o desempenho da API e do banco de dados relacional.

A emissão síncrona de chamadas HTTP externas durante o processamento do pedido bloquearia o pool de conexões do MySQL e criaria dependência temporal inaceitável em relação à disponibilidade dos servidores dos clientes. Caso o endpoint de um cliente estivesse fora do ar ou apresentasse latência excessiva, a transação de mudança de status sofreria timeout ou forçaria um rollback indevido de operações comerciais válidas (como pagamentos e baixas de estoque).

## Fatores de Decisão
- Assegurar consistência atômica entre a alteração de status do pedido e a emissão do evento de notificação.
- Desacoplar a thread de execução do banco de dados relacional das chamadas de rede externas para a internet.
- Evitar o problema arquitetural de consistência Dual-Write entre bancos de dados e mensagerias externas.
- Minimizar a sobrecarga operacional evitando a introdução de novos clusters de infraestrutura (como Kafka ou RabbitMQ).

## Opções Consideradas
1. Padrão Transactional Outbox utilizando tabela dedicada no MySQL 8.0 existente sob a mesma transação do Prisma.
2. Disparo HTTP síncrono diretamente dentro do método `OrderService.changeStatus`.
3. Publicação direta de eventos em broker de mensageria externo dedicado (Redis Streams, RabbitMQ ou Apache Kafka).

## Resultado da Decisão
Opção escolhida: "Padrão Transactional Outbox no MySQL 8.0", porque permite persistir o evento na tabela `webhook_outbox` dentro do mesmo bloco atômico `prisma.$transaction(async (tx) => { ... })` que atualiza a entidade `Order`, o histórico `OrderStatusHistory` e o inventário em `Product`.

Se a transação do pedido for confirmada com sucesso (`COMMIT`), o evento de notificação está garantido em armazenamento durável. Caso ocorra qualquer erro de negócio ou falha relacional, a transação sofre rollback integral e o evento não é gerado, eliminando qualquer risco de inconsistência de dados.

## Prós e Contras das Opções

### Padrão Transactional Outbox no MySQL 8.0
- Prós: Consistência transacional estrita (propriedades ACID garantidas); zero adição de infraestrutura externa; simplicidade operacional para a equipe atual; snapshot do payload congelado na origem.
- Contras: Aumenta a volumetria de escrita no MySQL; exige um processo consumidor assíncrono operando em polling na tabela `webhook_outbox`.

### Disparo HTTP síncrono no OrderService
- Prós: Implementação trivial sem necessidade de tabelas intermediárias ou workers assíncronos.
- Contras: Acoplamento temporal severo; travamento de conexões de banco por clientes lentos; falhas externas forçam rollback de mudanças legítimas de pedidos.

### Mensageria externa dedicada (Kafka / RabbitMQ / Redis Streams)
- Prós: Alta vazão para milhões de mensagens por segundo e recursos avançados de streaming de eventos.
- Contras: Complexidade operacional desproporcional (*overengineering*) para o volume atual; necessidade de coordenar consistência de duas fases (Dual-Write) entre o MySQL e o broker externo.

## Consequências
- Efeitos positivos: O sistema ganha capacidade de notificação orientada a eventos sem risco de estados inconsistentes; os clientes B2B recebem atualizações em tempo hábil sem sobrecarregar a API com consultas de polling.
- Efeitos negativos: Necessidade de manutenção de índices compostos em `(status, next_retry_at, created_at)` para manter a leitura da outbox eficiente e criação futura de rotina de expurgo de registros antigos com status `DELIVERED`.
- Efeitos operacionais: Monitoramento da contagem de registros pendentes na tabela `webhook_outbox` para detecção precoce de represamento.

## Referências
- `src/modules/orders/order.service.ts:131-178` - Bloco transacional atômico onde a outbox é gravada.
- `docs/RFC.md:44-65` - Sequência transacional formalizada da proposta técnica.
- `TRANSCRICAO.md:44-53` - Discussão técnica sobre outbox e descarte de mensageria externa.
