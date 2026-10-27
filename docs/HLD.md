### HLD: Order Management System (OMS) Modulo de Webhooks de Notificacao de Pedidos

Versao: 1.0.0
Data: 2026-10-26
Responsavel: Larissa (Tech Lead) / Diego (Engenheiro Senior de Plataforma)

---

### Objetivo tecnico
Implementar uma arquitetura confiavel, desacoplada e resiliente para notificacao de eventos de mudanca de status de pedidos para sistemas externos de clientes B2B. A solucao substitui o modelo ineficiente de polling continuo em GET /orders atraves do padrao Transactional Outbox em MySQL, garantindo entrega em menos de 10 segundos, integridade de dados at-least-once com HMAC-SHA256 e isolamento operacional entre a API e o despacho de rede.

Dependencias com outros sistemas
- Endpoints receptores HTTPS gerenciados pelos clientes corporativos B2B (Atlas Comercial, MaxDistribuicao, Nova Cargo)
- Banco de dados relacional MySQL 8.0 como fonte unica da verdade e mecanismo de persistencia transacional
- Runtime Node.js (versao >= 20) com suporte a ESM nativo e ecossistema Express 4.21 com Prisma ORM 5.22

---

### Arquitetura geral
A solucao adota uma arquitetura baseada no padrao Monolito Modular, desacoplando o recebimento de requisicoes HTTP da execucao assincrona de chamadas de rede externas. O enfileiramento dos eventos ocorre de forma atomica dentro da transacao SQL de alteracao de status de pedido, enquanto um processo independente (Worker) consome os eventos e executa as notificacoes de saida.

Ambiente de implantacao
- Implantacao em containers OCI (Docker) em ambiente Cloud/Hybrid
- Separacao de processos no mesmo ambiente de banco: processo de API (src/server.ts) e processo de Worker (src/worker.ts) compartilhando a mesma rede interna de banco MySQL 8.0

Tecnologias principais
- Node.js >= 20 com ECMAScript Modules (ESM)
- TypeScript 5.6 em modo estrito
- Express 4.21 para roteamento HTTP e composicao de controladores
- MySQL 8.0 gerenciado via Prisma ORM 5.22
- Zod 3.23 para validacao fail-fast de esquemas e contratos de entrada
- Pino 9.5 para logging estruturado com redacao automatica de dados sensiveis
- Modulo nativo node:crypto para calculo de assinaturas HMAC-SHA256

Padroes adotados
- Transactional Outbox Pattern para garantia de consistencia entre pedido e evento
- Polling Consumer Pattern em processo dedicado com intervalo de 2 segundos
- At-Least-Once Delivery com header de idempotencia (X-Event-Id) no cliente
- Assinatura digital simetrica (HMAC-SHA256) com segredo exclusivo por endpoint
- Exponential Backoff Retry com Dead Letter Queue (DLQ) para tolerancia a falhas

---

### Componentes e responsabilidades
| Componente | Responsabilidades | Dependencias |
| ----------- | ----------------- | ------------ |
| API Monolith (Express) | Roteamento HTTP, autenticacao JWT, controle de acesso baseado em cargos, validacao de esquemas Zod e tratamento centralizado de erros | Node.js, Express, Middlewares (auth, error, validate) |
| OrderService Extension | Execucao da maquina de estados de pedidos, baixa/devolucao de estoque e gravacao atomica do evento na outbox sob transacao SQL | PrismaClient, OrderStatus, ProductRepository, WebhookOutbox |
| Webhooks Module | Gerenciamento de endpoints de clientes, validacao de URLs HTTPS, geracao e rotacao de secrets (grace period 24h) e historico de entregas | PrismaClient, Zod, AppError, Pino Logger |
| Dedicated Outbound Worker | Loop de polling a cada 2s, leitura de eventos PENDING, calculo de HMAC-SHA256, envio HTTP (timeout 10s), retentativas exponenciais e descarte para DLQ | Processo Node.js dedicado, PrismaClient proprio, fetch nativo |
| MySQL 8.0 Database | Persistencia transacional de tabelas de negocio (orders, products) e tabelas de webhooks (outbox, endpoints, deliveries, dead_letter) | Sistema de armazenamento em disco e pool de conexoes relacional |

---

### Fluxo de requisicoes e de dados
**Fluxo de requisicao**
- 1. Cliente ou Operador envia requisicao PATCH /api/v1/orders/:id/status para alterar o status do pedido
- 2. OrderController autentica a chamada e encaminha para OrderService.changeStatus()
- 3. Dentro de prisma.$transaction, o servico valida a transicao, atualiza estoque em products, atualiza orders e registra em order_status_history
- 4. O servico consulta webhooks ativos para o customer_id com filtro compativel com o novo status
- 5. O servico insere registro na tabela webhook_outbox com status PENDING e payload serializado congelado (<= 64 KB)
- 6. A transacao SQL e confirmada (COMMIT) e a API responde 200 OK
- 7. O processo Dedicated Outbound Worker consulta a tabela webhook_outbox a cada 2 segundos buscando registros PENDING
- 8. O Worker marca registros como PROCESSING, calcula assinatura HMAC-SHA256 com a secret do endpoint e envia HTTP POST com timeout de 10s
- 9. Ao receber HTTP 2xx do cliente, o Worker marca o evento como DELIVERED e registra log na tabela webhook_delivery_history

**Fluxo de dados**
- Requisicao de negocio de pedido (OrderInput) -> Validacao e persistencia relacional -> Snapshot JSON congelado (WebhookPayload) -> Insercao atomica na webhook_outbox -> Despacho assincrono com HMAC-SHA256 e headers de rastreabilidade (X-Signature, X-Event-Id, X-Timestamp, X-Webhook-Id) -> Endpoint HTTPS do cliente corporativo

---

### Modelo de dados (alto nivel)
Entidades principais
- WebhookEndpoint: cadastro do destino HTTPS contendo customer_id, url, secret ativa, previous_secret, expires_at, status_filter e flag is_active
- WebhookOutbox: fila relacional contendo event_id (UUID), webhook_id, payload congelado em JSON, status (PENDING, PROCESSING, DELIVERED, FAILED), attempt_count, next_retry_at e timestamps
- WebhookDeliveryHistory: registro auditavel de cada tentativa de envio contendo webhook_id, event_id, http_status, duration_ms, error_message e timestamp
- WebhookDeadLetter: tabela de isolamento contendo webhook_id, payload original, ultimo codigo HTTP, motivo da falha, total de tentativas e flag is_reprocessed

Relacoes
- Customer (1) -> (N) WebhookEndpoint: um cliente corporativo pode cadastrar multiplos endpoints de notificacao
- WebhookEndpoint (1) -> (N) WebhookOutbox: um endpoint possui multiplos eventos enfileirados
- WebhookEndpoint (1) -> (N) WebhookDeliveryHistory: um endpoint possui historico de tentativas de entrega
- WebhookEndpoint (1) -> (N) WebhookDeadLetter: eventos em falha definitiva sao vinculados ao endpoint correspondente
- Order (1) -> (N) WebhookOutbox: associacao logica preservada dentro do payload congelado via order_id

Fonte de verdade
- Banco de dados relacional MySQL 8.0 gerenciado pelo Prisma ORM

---

### Interfaces publicas
| Nome | Tipo | Protocolo | Exposicao | SLAs/Limites |
| ---- | ---- | ---------- | --------- | ------------- |
| POST /api/v1/webhooks | API | REST / JSON | Externa (Autenticada) | p95 < 150 ms (URL HTTPS obrigatoria) |
| GET /api/v1/webhooks | API | REST / JSON | Externa (Autenticada) | p95 < 100 ms (Listagem paginada por cliente) |
| PATCH /api/v1/webhooks/:id | API | REST / JSON | Externa (Autenticada) | p95 < 150 ms (Atualizacao de URL e eventos) |
| DELETE /api/v1/webhooks/:id | API | REST / JSON | Externa (Autenticada) | p95 < 150 ms (Remocao de cadastro) |
| POST /api/v1/webhooks/:id/rotate-secret | API | REST / JSON | Externa (Autenticada) | p95 < 150 ms (Rotacao de chave com 24h grace period) |
| GET /api/v1/webhooks/:id/deliveries | API | REST / JSON | Externa (Autenticada) | p95 < 150 ms (Historico paginado de envios) |
| POST /admin/webhooks/dead-letter/:id/replay | API | REST / JSON | Interna (Role ADMIN) | p95 < 200 ms (Replay auditado de eventos da DLQ) |
| Outbound Webhook Event Dispatch | Webhook | HTTPS POST | Externa (Outbound) | Timeout estrito 10s, Payload <= 64 KB |

---

### Consideracoes de escalabilidade e disponibilidade
Abordagem geral
- Desacoplamento fisico entre o processamento de requisicoes de entrada da API e as chamadas de rede externas, impedindo que lentidoes de clientes degradem as rotas do OMS

Tecnicas aplicadas
- Indice composto otimizado na tabela webhook_outbox cobrindo (status, next_retry_at, created_at)
- Consumo em lotes pequenos com limites de busca (batch de 20 a 50 registros por ciclo de polling)
- Pools de conexoes PrismaClient isolados entre o processo Express e o processo Worker
- Politica de backoff exponencial com espacamento crescente para conter tempestades de retentativas
- Timeout rigido de 10 segundos em todas as chamadas HTTP externas via fetch com AbortController

Meta de disponibilidade
- 99.5 por cento de disponibilidade mensal para a API e para o servico de despacho de webhooks

---

### Seguranca
Autenticacao
- Requisicoes na API REST exigem token JWT Bearer no cabecalho Authorization
- Requisicoes outbound enviadas aos clientes sao assinadas digitalmente via HMAC-SHA256 no cabecalho X-Signature

Autorizacao
- Controle de acesso baseado em cargos via middleware requireRole, reservando o endpoint de replay de DLQ estritamente para a role ADMIN

Protecao de dados
- Exigencia obrigatoria de TLS 1.2 ou superior (recusa de URLs HTTP no cadastro)
- Sanitizacao ativa no Logger Pino para nao registrar segredos, tokens ou senhas em texto claro
- Limite maximo de 64 KB por evento para prevencao contra ataques de exaustao de memoria ou buffer overflow
- Snapshot enxuto sem exposicao de dados de pagamento ou itens detalhados

Gestao de segredos
- Chaves secretas geradas com entropia criptografica (crypto.randomBytes)
- Segredo unico por endpoint (o comprometimento de uma chave nao afeta outros clientes)
- Suporte a chave anterior (previous_secret) valida por 24 horas apos a solicitacao de rotacao

---

### Observabilidade
Logs
- Emissao de logs estruturados em formato JSON utilizando o Pino Logger compartilhado
- Campos contextuais obrigatorios: request_id, event_id, webhook_id, customer_id, duration_ms e status_code

Metricas
- Taxa de eventos inseridos na outbox por minuto
- Metricas RED do worker: taxa de entrega, taxa de erros HTTP externos e duracao das chamadas em milissegundos
- Indicador de volumetria de registros na tabela webhook_outbox com status PENDING e FAILED
- Contagem total de eventos isolados na Dead Letter Queue (DLQ)

Tracing
- Propagacao do cabecalho X-Event-Id como identificador unico de correlacao ponta a ponta
- Cabecalho X-Request-Id rastreando as transacoes originadas na API REST

Dashboards e alertas
- Alerta para acumulacao anormal de eventos PENDING na outbox (> 100 eventos por mais de 5 minutos)
- Alerta para parada de execucao ou queda do processo Worker (ausencia de heartbeat)
- Alerta para novos registros persistidos na tabela webhook_dead_letter

---

### Riscos arquiteturais e mitigacao
#### Sobrecarga do MySQL decorrente de polling continuo do Worker
- **Probabilidade:** media
- **Impacto:** medio
- **Mitigacao:**
  - Criacao de indice composto especifico cobrindo status, next_retry_at e created_at
  - Instancia dedicada de PrismaClient para o Worker com cota de pool de conexoes controlada
  - Consultas em lote com limit pequeno (20 a 50 itens)
- **Plano de contingencia:** Ajuste em tempo de execucao da cadencia de polling de 2 segundos para 5 segundos via variavel de configuracao.

#### Acumulo de retentativas decorrente de indisponibilidade prolongada de clientes
- **Probabilidade:** media
- **Impacto:** medio
- **Mitigacao:**
  - Backoff exponencial com janela crescente (1m, 5m, 30m, 2h, 12h)
  - Limite estrito de 5 tentativas por evento
  - Transferencia definitiva para a tabela de DLQ liberando a fila principal
- **Plano de contingencia:** Eventos esgotados sao retidos na DLQ para posterior replay manual pelo administrador quando o cliente restabelecer o ambiente.

---

### ADRs e proximos passos
ADRs associados
- ADR-001 - Padrao Outbox no MySQL
- ADR-002 - Worker Desacoplado em Processo Separado com Polling
- ADR-003 - Politica de Retry com Backoff Exponencial e Dead Letter Queue
- ADR-004 - Autenticacao Criptografica HMAC-SHA256 com Secret por Endpoint
- ADR-005 - Garantia de Entrega At-Least-Once com Header de Idempotencia
- ADR-006 - Reuso dos Padroes de Codigo e Arquitetura Existentes

Decisoes pendentes
- Avaliacao de necessidade de limitador de taxa (Rate Limiting) de saida por cliente apos medicao de trafego em producao
- Avaliacao de implementacao de canal de alertas proativos por e-mail para falhas persistentes de webhooks em fase futura
- Modelagem de particionamento e uso de SELECT ... FOR UPDATE SKIP LOCKED caso seja necessario escalar para multiplos workers paralelos

Proximos passos
- Elaboracao do documento detalhado de implementacao (FDD - Feature Design Document)
- Geracao formal dos registros de decisao arquitetural (ADRs) individuais no diretorio docs/adrs/
- Atualizacao da matriz de rastreabilidade e do diario de bordo no arquivo docs/TRACKER.md
