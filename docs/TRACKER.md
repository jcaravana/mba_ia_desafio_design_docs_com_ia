# Tracker de Rastreabilidade 

Abaixo está a tabela de referência cruzada que garante que cada elemento dos documentos produzidos até o momento possui origem comprovada, eliminando qualquer tipo de alucinação.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PRD-OBJ-01** | `docs/PRD.md` | Objetivo / Métrica | Notificar clientes em quase tempo real com latência inferior a 10s | TRANSCRICAO | `[09:02] Marcos` |
| **PRD-OBJ-02** | `docs/PRD.md` | Objetivo / Métrica | Reduzir em mais de 80% o volume de polling no endpoint GET /orders | TRANSCRICAO | `[09:00] Marcos` |
| **PRD-OBJ-03** | `docs/PRD.md` | Objetivo / Métrica | Reter 100% dos clientes B2B que solicitaram a feature (Atlas, MaxDistribuição) | TRANSCRICAO | `[09:00] Marcos` |
| **PRD-FORA-01** | `docs/PRD.md` | Fora de Escopo | Notificações de alerta via e-mail descartadas para a primeira fase | TRANSCRICAO | `[09:37] Larissa` |
| **PRD-FORA-02** | `docs/PRD.md` | Fora de Escopo | Rate limiting de saída postergado para observação em produção | TRANSCRICAO | `[09:39] Larissa` |
| **PRD-FORA-03** | `docs/PRD.md` | Fora de Escopo | Dashboard visual ou painel gráfico de webhooks no frontend fora de escopo | TRANSCRICAO | `[09:40] Larissa` |
| **PRD-FR-01** | `docs/PRD.md` | Requisito Funcional | Cadastro de endpoints de webhook via POST com seleção de eventos | TRANSCRICAO | `[09:31] Marcos` |
| **PRD-FR-02** | `docs/PRD.md` | Requisito Funcional | Filtragem proativa de eventos no momento da inserção na outbox | TRANSCRICAO | `[09:34] Bruno` |
| **PRD-FR-03** | `docs/PRD.md` | Requisito Funcional | Snapshot congelado do payload serializado em JSON na inserção | TRANSCRICAO | `[09:52] Larissa` |
| **PRD-FR-04** | `docs/PRD.md` | Requisito Funcional | Identificadores padronizados gerados estritamente em UUID | TRANSCRICAO | `[09:51] Larissa` |
| **PRD-FR-05** | `docs/PRD.md` | Requisito Funcional | Processo worker dedicado rodando separadamente da API HTTP | TRANSCRICAO | `[09:11] Larissa` |
| **PRD-FR-06** | `docs/PRD.md` | Requisito Funcional | Polling assíncrono do worker executado em loop a cada 2 segundos | TRANSCRICAO | `[09:09] Diego` |
| **PRD-FR-07** | `docs/PRD.md` | Requisito Funcional | Timeout de conexão e resposta das chamadas HTTP outbound em 10 segundos | TRANSCRICAO | `[09:42] Diego` |
| **PRD-FR-08** | `docs/PRD.md` | Requisito Funcional | Política de 5 retentativas com backoff exponencial (1m, 5m, 30m, 2h, 12h) | TRANSCRICAO | `[09:17] Diego` |
| **PRD-FR-09** | `docs/PRD.md` | Requisito Funcional | Persistência de mensagens falhas em tabela de Dead Letter Queue (DLQ) | TRANSCRICAO | `[09:18] Diego` |
| **PRD-FR-10** | `docs/PRD.md` | Requisito Funcional | Endpoint de replay manual de eventos da DLQ restrito à role ADMIN | TRANSCRICAO | `[09:36] Sofia` |
| **PRD-FR-11** | `docs/PRD.md` | Requisito Funcional | Consulta paginada do histórico de entregas via GET /webhooks/:id/deliveries | TRANSCRICAO | `[09:34] Marcos` |
| **PRD-FR-12** | `docs/PRD.md` | Requisito Funcional | Assinatura HMAC-SHA256 no header X-Signature com secret por endpoint | TRANSCRICAO | `[09:20] Sofia` |
| **PRD-FR-13** | `docs/PRD.md` | Requisito Funcional | Rotação de chave secreta com grace period de convivência de 24 horas | TRANSCRICAO | `[09:21] Sofia` |
| **PRD-FR-14** | `docs/PRD.md` | Requisito Funcional | Semântica de entrega at-least-once com header X-Event-Id para idempotência | TRANSCRICAO | `[09:25] Diego` |
| **PRD-FR-15** | `docs/PRD.md` | Requisito Funcional | Inclusão dos headers de entrega X-Timestamp e X-Webhook-Id | TRANSCRICAO | `[09:44] Diego` |
| **PRD-RNF-01** | `docs/PRD.md` | Requisito Não Funcional | Teto máximo de tamanho de payload serializado limitado a 64 KB | TRANSCRICAO | `[09:24] Diego` |
| **PRD-RNF-02** | `docs/PRD.md` | Requisito Não Funcional | Exigência estrita de URLs utilizando o protocolo HTTPS | TRANSCRICAO | `[09:23] Sofia` |
| **PRD-RNF-03** | `docs/PRD.md` | Requisito Não Funcional | Padronização de códigos de erro de domínio com prefixo WEBHOOK_ | TRANSCRICAO | `[09:28] Bruno` |
| **PRD-RNF-04** | `docs/PRD.md` | Requisito Não Funcional | Logs estruturados utilizando a instância compartilhada do Pino Logger | TRANSCRICAO | `[09:29] Bruno` |
| **RFC-PROP-01** | `docs/RFC.md` | Decisão Arquitetural | Adoção do padrão Transactional Outbox em MySQL | TRANSCRICAO | `[09:06] Diego` |
| **RFC-PROP-02** | `docs/RFC.md` | Integração de Código | Ponto de extensão no método changeStatus sob transação atômica | CODIGO | `src/modules/orders/order.service.ts` |
| **RFC-PROP-03** | `docs/RFC.md` | Integração de Código | Validação da máquina de estados no ciclo de vida de pedidos | CODIGO | `src/modules/orders/order.status.ts` |
| **RFC-PROP-04** | `docs/RFC.md` | Integração de Código | Middleware de controle de acesso para proteção de endpoints de admin | CODIGO | `src/middlewares/auth.middleware.ts` |
| **RFC-PROP-05** | `docs/RFC.md` | Integração de Código | Middleware centralizado de tratamento de AppError e erros relacionais | CODIGO | `src/middlewares/error.middleware.ts` |
| **RFC-PROP-06** | `docs/RFC.md` | Integração de Código | Sanitização ativa de segredos e credenciais nos logs estruturados | CODIGO | `src/shared/logger/index.ts` |
| **RFC-ALT-01** | `docs/RFC.md` | Alternativa Descartada | Disparo HTTP síncrono descartado por acoplamento temporal e risco de rollback | TRANSCRICAO | `[09:04] Bruno` |
| **RFC-ALT-02** | `docs/RFC.md` | Alternativa Descartada | Mensageria externa (Redis/Kafka) descartada por overengineering operacional | TRANSCRICAO | `[09:07] Diego` |
| **RFC-ALT-03** | `docs/RFC.md` | Alternativa Descartada | Triggers nativas descartadas por falta de suporte a LISTEN/NOTIFY no MySQL | TRANSCRICAO | `[09:09] Diego` |
| **RFC-OPEN-01** | `docs/RFC.md` | Questão em Aberto | Avaliação de rate limiting de saída após medição em ambiente produtivo | TRANSCRICAO | `[09:39] Larissa` |
| **RFC-OPEN-02** | `docs/RFC.md` | Questão em Aberto | Avaliação de necessidade futura de canais de notificação proativa de falha | TRANSCRICAO | `[09:37] Marcos` |
| **RFC-OPEN-03** | `docs/RFC.md` | Questão em Aberto | Estratégia de particionamento e locks para escala com múltiplos workers | TRANSCRICAO | `[09:13] Diego` |
| **RFC-RISK-01** | `docs/RFC.md` | Risco e Mitigação | Sobrecarga de polling no MySQL mitigada com índice (status, created_at) | TRANSCRICAO | `[09:08] Diego` |
| **RFC-RISK-02** | `docs/RFC.md` | Risco e Mitigação | Janela de 2 dias úteis reservada para revisão de segurança da Sofia | TRANSCRICAO | `[09:46] Sofia` |
| **FDD-INT-01** | `docs/FDD.md` | Integração de Código | Invocação pura de publishWebhookEvent sob prisma.$transaction | CODIGO | `src/modules/orders/order.service.ts` |
| **FDD-INT-02** | `docs/FDD.md` | Integração de Código | Reuso do enum OrderStatus no schema Zod de eventos | CODIGO | `src/modules/orders/order.status.ts` |
| **FDD-INT-03** | `docs/FDD.md` | Integração de Código | Especialização de classes de erro com prefixo WEBHOOK_ via AppError | CODIGO | `src/shared/errors/app-error.ts` |
| **FDD-INT-04** | `docs/FDD.md` | Integração de Código | Proteção de rota de replay de DLQ com requireRole('ADMIN') | CODIGO | `src/middlewares/auth.middleware.ts` |
| **FDD-INT-05** | `docs/FDD.md` | Integração de Código | Formatação padronizada de erros de domínio via middleware central | CODIGO | `src/middlewares/error.middleware.ts` |
| **FDD-INT-06** | `docs/FDD.md` | Integração de Código | Redação e sanitização de secrets e tokens no Pino Logger | CODIGO | `src/shared/logger/index.ts` |
| **FDD-CTR-01** | `docs/FDD.md` | Contrato Público | Endpoint POST /api/v1/webhooks para cadastro com retorno de secret | TRANSCRICAO | `[09:31] Marcos` |
| **FDD-CTR-02** | `docs/FDD.md` | Contrato Público | Endpoint GET /api/v1/webhooks com paginação por cliente | TRANSCRICAO | `[09:33] Bruno` |
| **FDD-CTR-03** | `docs/FDD.md` | Contrato Público | Endpoint POST /api/v1/webhooks/:id/rotate-secret com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| **FDD-CTR-04** | `docs/FDD.md` | Contrato Público | Endpoint POST /admin/webhooks/dead-letter/:id/replay para perfil ADMIN | TRANSCRICAO | `[09:35] Diego` |
| **FDD-CTR-05** | `docs/FDD.md` | Contrato Público | Despacho outbound com X-Signature, X-Event-Id, X-Timestamp, X-Webhook-Id | TRANSCRICAO | `[09:44] Diego` |
| **FDD-RES-01** | `docs/FDD.md` | Resiliência Técnica | Timeout estrito de 10s via AbortController nas requisições do worker | TRANSCRICAO | `[09:42] Diego` |
| **FDD-RES-02** | `docs/FDD.md` | Resiliência Técnica | Progressão de retries em 5 níveis (1m, 5m, 30m, 2h, 12h) | TRANSCRICAO | `[09:17] Diego` |
| **FDD-RES-03** | `docs/FDD.md` | Resiliência Técnica | Isolamento definitivo na tabela webhook_dead_letter após 5 falhas | TRANSCRICAO | `[09:18] Diego` |
| **FDD-OBS-01** | `docs/FDD.md` | Observabilidade | Métricas RED, contagem de outbox e alerta para acúmulo de eventos pendentes | TRANSCRICAO | `[09:29] Bruno` |
| **ADR-001** | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Decisão Arquitetural | Padrão Outbox no MySQL para Notificações de Pedidos | TRANSCRICAO | `[09:06] Diego` |
| **ADR-002** | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão Arquitetural | Worker Desacoplado em Processo Separado com Polling de 2s | TRANSCRICAO | `[09:11] Larissa` |
| **ADR-003** | `docs/adrs/ADR-003-politica-retry-backoff-dlq.md` | Decisão Arquitetural | Política de Retentativas com Backoff Exponencial e DLQ | TRANSCRICAO | `[09:17] Diego` |
| **ADR-004** | `docs/adrs/ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md` | Decisão Arquitetural | Autenticação Criptográfica HMAC-SHA256 com Secret por Endpoint | TRANSCRICAO | `[09:20] Sofia` |
| **ADR-005** | `docs/adrs/ADR-005-garantia-at-least-once-idempotencia.md` | Decisão Arquitetural | Garantia de Entrega At-Least-Once com Header de Idempotência | TRANSCRICAO | `[09:25] Diego` |
| **ADR-006** | `docs/adrs/ADR-006-reuso-padroes-arquiteturais-existentes.md` | Decisão Arquitetural | Reuso dos Padrões de Código e Arquitetura Existentes | CODIGO | `src/shared/errors/app-error.ts` |
