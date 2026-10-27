# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

* **Versão:** 1.0.0
* **Data:** 2026-10-26
* **Status:** Aprovado para Engenharia
* **Autor / Responsável:** Marcos (Product Manager), em colaboração com Larissa (Tech Lead) e Sofia (Segurança)
* **Sistema Alvo:** `order-management-api` (Order Management System — OMS REST API)

---

## 1. Resumo e Contexto da Feature

Esta iniciativa introduz um mecanismo nativo e corporativo de **Webhooks de Saída (Outbound Webhooks)** no Order Management System (OMS). A solução tem como finalidade notificar sistemas externos de clientes B2B em tempo hábil sempre que houver alteração confirmada no ciclo de vida de seus pedidos (`PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`). 

O OMS foi concebido como um monólito modular em Node.js e TypeScript com banco de dados MySQL 8.0 gerenciado via Prisma ORM. Atualmente, a plataforma opera sem qualquer infraestrutura de mensageria assíncrona, filas ou eventos. Para acompanhar o progresso de faturamento e entrega, clientes corporativos estratégicos (como Atlas Comercial, Max Distribuição e Nova Cargo) executam consultas periódicas em lote no endpoint de leitura `GET /orders`. Esse modelo de polling satura o pool de conexões do banco relacional, eleva o consumo de CPU e memória, e introduz latência na cadeia de suprimentos dos clientes.

A feature preenche essa lacuna através do padrão arquitetural **Transactional Outbox**, persistindo o snapshot do evento na mesma transação SQL que altera o status do pedido, e desacoplando o despacho HTTP através de um processo worker independente em Node.js com ciclo de polling de 2 segundos. A comunicação externa é blindada com assinaturas criptográficas **HMAC-SHA256** por endpoint, garantia de entrega *at-least-once* com identificador de desduplicação (`X-Event-Id`), política de 5 retentativas com backoff exponencial e retenção de falhas em **Dead Letter Queue (DLQ)** com replay restrito a administradores.

---

## 2. Problema e Motivação

### 2.1. Dores Críticas Identificadas
1. **Sobrecarga de Infraestrutura por Polling Excessivo:** Clientes parceiros realizam requisições repetitivas no endpoint `GET /orders` em intervalos de poucos segundos para detectar alterações de status. Isso gera contenção no pool de conexões do MySQL 8.0, concorre com as transações de escrita do método `OrderService.changeStatus` e eleva custos operacionais desnecessários.
2. **Defasagem Operacional e Latência Percebida:** O cliente corporativo só toma ciência de que um pedido foi pago ou despachado no próximo ciclo de polling de seu agendador local. Em operações logísticas de alta rotação, essa defasagem atrasa o faturamento, a emissão de notas fiscais e o carregamento de cargas.
3. **Risco Imediato de Churn e Perda de Receita:** O cliente estratégico Atlas Comercial formalizou risco crítico de rescisão contratual e migração para a concorrência caso o OMS não disponibilize notificações outbound em tempo real até o fechamento do trimestre corrente.
4. **Falta de Segurança e Rastreabilidade em Integrações Outbound:** Transmitir notificações sem assinatura digital ou controle de integridade exporia os receptores dos clientes a ataques de personificação (falsificação de eventos) e tornaria impossível auditar incidentes de entrega.

---

## 3. Público-Alvo e Cenários de Uso

### 3.1. Personas e Público-Alvo
* **Desenvolvedores e Engenheiros de Integração dos Clientes B2B:** Profissionais responsáveis por configurar os receptores HTTPS nos sistemas de ERP/WMS de clientes corporativos (Atlas Comercial, Max Distribuição, Nova Cargo).
* **Operadores e Gestores de Atendimento do OMS:** Equipe interna que acompanha a volumetria de vendas e necessita de diagnósticos transparentes sobre a saúde das integrações.
* **Administradores de TI e Engenharia de Segurança:** Usuários com privilégio `ADMIN` responsáveis por auditar incidentes graves na Dead Letter Queue e autorizar o reprocessamento manual de mensagens.

### 3.2. Cenários de Uso Chave
* **Cenário 1 — Notificação Automática de Faturamento:** Um pedido do cliente Atlas Comercial transiciona do status `PENDING` para `PAID`. Atomicamente, o evento é gravado na outbox. Em menos de 4 segundos, o worker despacha uma requisição HTTP POST assinada com HMAC-SHA256 para o endpoint cadastrado da Atlas, que imediatamente libera a separação de estoque no seu WMS.
* **Cenário 2 — Configuração e Gestão de Endpoints:** O desenvolvedor da Max Distribuição cadastra um endpoint HTTPS via API (`POST /api/v1/webhooks`) selecionando apenas os eventos `SHIPPED` e `DELIVERED`, recebendo no ato de criação a secret exclusiva para validação de assinaturas.
* **Cenário 3 — Rotação Preventiva de Segredo Criptográfico:** A equipe de segurança da Nova Cargo solicita a rotação periódica de credenciais. Através do endpoint `POST /api/v1/webhooks/:id/rotate-secret`, o sistema gera uma nova secret e mantém a secret anterior válida por 24 horas (*grace period*), permitindo que a troca seja efetuada no cliente sem parada técnica.
* **Cenário 4 — Resiliência diante de Manutenção Externa e Replay de DLQ:** O servidor receptor de um cliente fica offline por 14 horas durante manutenção de datacenter. O worker do OMS executa 5 retentativas com backoff exponencial (1m, 5m, 30m, 2h, 12h) e, ao esgotar as tentativas, move o evento com segurança para a tabela `webhook_dead_letter`. Assim que o cliente reestabelece o ambiente, um administrador do OMS aciona `POST /admin/webhooks/dead-letter/:id/replay`, reenfileirando a mensagem para entrega sem perda de dados.

---

## 4. Objetivos e Métricas de Sucesso

| Objetivo de Negócio / Técnico | Métrica Quantitativa | Meta Operacional |
| :--- | :--- | :--- |
| **Notificar clientes B2B em quase tempo real** | Latência fim a fim entre o commit transacional de `OrderService.changeStatus` e o envio da requisição HTTP pelo worker | Menor que 10 segundos no p95 (meta média operacional entre 2 e 4 segundos) |
| **Eliminar o desperdício de requisições de leitura** | Volume de requisições `GET /orders` originadas por clientes habilitados com webhook | Redução mínima de 80% nas chamadas de polling |
| **Garantir a retenção de clientes corporativos chave** | Taxa de retenção dos clientes Atlas Comercial, Max Distribuição e Nova Cargo ao final do trimestre | 100% de retenção (zero churn) |
| **Confiabilidade e taxa de sucesso na entrega** | Percentual de eventos confirmados com HTTP 2xx (diretos ou via retentativas) sobre o total de eventos válidos gerados | Mínimo de 99,5% de sucesso |

---

## 5. Escopo (Incluso e Fora de Escopo)

### 5.1. Escopo Incluso
* **Persistência Transacional (Transactional Outbox):** Criação da tabela `webhook_outbox` no MySQL 8.0, populada atomicamente dentro do bloco `prisma.$transaction` do método `OrderService.changeStatus`.
* **Snapshot Congelado de Dados:** Gravação do corpo do evento como JSON serializado no instante da mutação, com teto rígido de tamanho fixado em 64 KB.
* **Filtragem Pró-Ativa de Eventos:** Gravação de linhas na outbox realizada somente se o cliente tiver endpoint ativo configurado com interesse no status de destino (`events`).
* **Worker Assíncrono Desacoplado:** Processo Node.js independente (`src/worker.ts`, disparado via `npm run worker`) executando polling regular a cada 2 segundos.
* **Assinatura Digital Criptográfica HMAC-SHA256:** Assinatura enviada no cabeçalho `X-Signature` calculada sobre o payload bruto com secret individual por endpoint gerada com alta entropia (`crypto.randomBytes`).
* **Rotação Segura com Período de Convivência:** Endpoint de rotação que preserva a secret anterior ativa por exatamente 24 horas (`previous_secret` e `expires_at`).
* **Cabeçalhos de Rastreabilidade e Idempotência:** Envio mandatório dos cabeçalhos `X-Signature`, `X-Event-Id` (UUID v4), `X-Timestamp` (ISO 8601) e `X-Webhook-Id`.
* **Protocolo de Retentativas com Backoff Exponencial:** Máquina de retentativas em 5 etapas escalonadas (1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas) e timeout estrito de 10 segundos por requisição outbound via `AbortController`.
* **Isolamento em Dead Letter Queue (DLQ):** Tabela `webhook_dead_letter` para armazenamento de eventos que falharem após a 5ª tentativa, contendo payload congelado, código de erro HTTP e diagnóstico.
* **Reprocessamento Manual Restrito a Administradores:** Endpoint `POST /admin/webhooks/dead-letter/:id/replay` protegido pelo middleware `requireRole('ADMIN')` com emissão de log de auditoria.
* **APIs de Gerenciamento de Webhooks e Histórico:** Endpoints REST em `/api/v1/webhooks` para cadastro, listagem, atualização, exclusão e consulta paginada de histórico de entregas (`/api/v1/webhooks/:id/deliveries`).
* **Validação Estrita via Zod:** Rejeição mandatória de URLs com protocolo HTTP inseguro, exigindo HTTPS com validação fail-fast.

### 5.2. Fora de Escopo (Explicitamente Descartados ou Adiados)
1. **Disparo de Notificações de Alerta por E-mail:** Envio automatizado de e-mails para operadores do cliente em caso de falhas consecutivas de webhook (descartado para a primeira fase; o cliente deve monitorar seus próprios logs e status de entrega).
2. **Algoritmo Ativo de Rate Limiting de Saída:** Controle ativo de taxa de despacho (Token Bucket ou Leaky Bucket) por cliente (adiado para avaliação após observação da volumetria real em produção).
3. **Painel Visual / Dashboard no Frontend:** Criação de interface gráfica no backoffice web para visualização de webhooks (gestão estritamente restrita a endpoints REST da API nesta fase).
4. **Escalonamento Horizontal Concorrente com Particionamento:** Coordenação distribuída entre múltiplos workers simultâneos (a fase inicial operará em arquitetura *single-worker* com ordenação garantida por `created_at`).
5. **Webhooks de Entrada (Inbound Webhooks):** Recepção de requisições disparadas por sistemas externos para dentro do OMS (a feature contempla exclusivamente webhooks de saída).

---

## 6. Requisitos Funcionais

### RF-001: Cadastro de Endpoint de Webhook
* **Descrição:** Permite registrar um novo endpoint HTTPS associado a um cliente com seleção dos status de pedidos a serem monitorados.
* **Fluxo Principal:**
  1. O usuário autenticado envia requisição `POST /api/v1/webhooks` informando `customerId`, `url` (HTTPS obrigatória), `events` (array de `OrderStatus`) e `description` opcional.
  2. O sistema valida os campos via esquema Zod, assegurando formato UUID no `customerId` e protocolo HTTPS na URL.
  3. O sistema verifica a existência do `customerId` na tabela `customers`.
  4. O sistema gera uma chave secreta criptográfica aleatória de 32 bytes (`whsec_...`).
  5. O sistema persiste o registro na tabela `webhook_endpoints` com status `isActive = true`.
  6. O sistema retorna HTTP 201 Created exibindo o identificador UUID do webhook, URL, eventos cadastrados e a chave secreta gerada em texto aberto (exibição única).
* **Exceções e Erros:**
  * URL informada com protocolo `http://`: retorna HTTP 400 Bad Request com código `WEBHOOK_INVALID_URL`.
  * `customerId` inexistente: retorna HTTP 404 Not Found com código `WEBHOOK_CUSTOMER_NOT_FOUND`.
  * Status informados inválidos: retorna HTTP 400 Bad Request com detalhes de validação Zod.
* **Prioridade:** Alta

### RF-002: Consulta e Listagem de Webhooks
* **Descrição:** Permite listar os webhooks cadastrados para um determinado cliente ou obter os detalhes de um webhook individual.
* **Fluxo Principal:**
  1. O usuário envia `GET /api/v1/webhooks?customerId=:id&page=1&pageSize=20` ou `GET /api/v1/webhooks/:id`.
  2. O sistema consulta a tabela `webhook_endpoints` e formata a resposta no envelope padronizado de paginação (`paginated`).
  3. O sistema omite rigorosamente o campo `secret` e `previous_secret` em qualquer listagem ou consulta.
* **Prioridade:** Média

### RF-003: Atualização e Desativação de Webhook
* **Descrição:** Permite alterar a URL de destino, os eventos monitorados ou alternar o estado de ativação (`isActive`).
* **Fluxo Principal:**
  1. O usuário envia `PATCH /api/v1/webhooks/:id` com os campos a modificar.
  2. O sistema valida os dados de entrada (reaplicando validação estrita de HTTPS caso a URL seja informada).
  3. O sistema atualiza o registro e responde com HTTP 200 OK.
* **Exceções e Erros:**
  * Webhook não localizado: retorna HTTP 404 Not Found com código `WEBHOOK_NOT_FOUND`.
* **Prioridade:** Média

### RF-004: Remoção Física de Webhook
* **Descrição:** Permite a exclusão de um cadastro de webhook.
* **Fluxo Principal:**
  1. O usuário envia `DELETE /api/v1/webhooks/:id`.
  2. O sistema exclui o registro na tabela `webhook_endpoints`.
  3. O sistema responde com HTTP 204 No Content.
* **Prioridade:** Baixa

### RF-005: Rotação Criptográfica de Segredo com Período de Convivência
* **Descrição:** Gera uma nova chave secreta para assinatura, mantendo a chave anterior temporariamente aceita para evitar descontinuidade na validação do cliente.
* **Fluxo Principal:**
  1. O usuário autenticado envia `POST /api/v1/webhooks/:id/rotate-secret`.
  2. O sistema recupera a secret atual e a transfere para a coluna `previous_secret`.
  3. O sistema grava o carimbo de data limite em `previous_secret_expires_at` somando exatamente 24 horas à data/hora atual.
  4. O sistema gera uma nova secret de 32 bytes e a persiste na coluna `secret`.
  5. O sistema responde com HTTP 200 OK exibindo a nova secret em texto aberto e o timestamp ISO 8601 de expiração da chave anterior.
* **Prioridade:** Alta

### RF-006: Enfileiramento Transacional Atômico (Transactional Outbox)
* **Descrição:** Garante que eventos de mudança de status de pedidos sejam persistidos exclusivamente se a transação do pedido for confirmada com sucesso.
* **Fluxo Principal:**
  1. O método `OrderService.changeStatus` inicia a transação `this.prisma.$transaction(async (tx) => { ... })`.
  2. Após validar regras de negócio, debitar estoque e gravar o histórico de status, o serviço invoca a função `publishWebhookEvent(tx, order, fromStatus, toStatus)`.
  3. A função consulta na conexão transacional `tx` se existem webhooks ativos para o `customerId` do pedido com interesse no `toStatus`.
  4. Caso existam, monta o snapshot do evento em JSON contendo: `eventId`, `eventType: order.status_changed`, `timestamp`, `data` (com `orderId`, `orderNumber`, `customerId`, `fromStatus`, `toStatus`, `totalCents`, `discountCents`).
  5. Valida que o JSON serializado não excede o teto de 64 KB.
  6. Insere os registros na tabela `webhook_outbox` com status `PENDING`, `attemptCount = 0` e `nextRetryAt = null`.
  7. A transação SQL é confirmada (`COMMIT`), liberando atomicamente a alteração do pedido e o evento na outbox.
* **Exceções e Erros:**
  * Falha na gravação na outbox acarreta o *rollback* imediato de toda a transação (o pedido e o estoque não são alterados).
  * Payload superior a 64 KB rejeita a transação com HTTP 422 Unprocessable Entity e código `WEBHOOK_PAYLOAD_TOO_LARGE`.
* **Prioridade:** Crítica

### RF-007: Despacho Assíncrono e Assinatura pelo Worker
* **Descrição:** Processo em segundo plano que consome mensagens da outbox e dispara as notificações HTTP.
* **Fluxo Principal:**
  1. O worker executa loop a cada 2 segundos consultando registros com status `PENDING` ou registros em retentativa cujo `next_retry_at <= NOW()`.
  2. Marca o lote selecionado como `PROCESSING`.
  3. Para cada registro, calcula o hash HMAC-SHA256 sobre o JSON serializado utilizando a secret ativa do endpoint.
  4. Executa requisição HTTP POST para a URL do cliente com timeout de 10 segundos configurado via `AbortController`.
  5. Inclui os cabeçalhos: `Content-Type: application/json`, `X-Signature: sha256={hash}`, `X-Event-Id: {uuid}`, `X-Timestamp: {iso}`, `X-Webhook-Id: {uuid}`.
  6. Se o endpoint do cliente responder com status HTTP 2xx, o worker marca o registro como `DELIVERED` e grava o resultado na tabela `webhook_delivery_history`.
* **Prioridade:** Crítica

### RF-008: Máquina de Retentativas com Backoff Exponencial
* **Descrição:** Reprocessamento escalonado de eventos diante de falhas temporárias no endpoint receptor.
* **Fluxo Principal:**
  1. Caso a chamada retorne erro de rede, timeout (> 10s) ou status HTTP fora da faixa 2xx (4xx ou 5xx), o worker incrementa `attempt_count`.
  2. Se `attempt_count < 5`, calcula o intervalo de espera baseado na tentativa:
     * Tentativa 1: +1 minuto
     * Tentativa 2: +5 minutos
     * Tentativa 3: +30 minutos
     * Tentativa 4: +2 horas
     * Tentativa 5: +12 horas
  3. Atualiza `next_retry_at` com o timestamp calculado e redefine o status para `PENDING`.
  4. O registro é ignorado nas leituras do worker até que o relógio atinja `next_retry_at`.
* **Prioridade:** Alta

### RF-009: Descarte e Isolamento em Dead Letter Queue (DLQ)
* **Descrição:** Encaminhamento seguro de mensagens cuja entrega falhou em definitivo após 5 tentativas.
* **Fluxo Principal:**
  1. Quando a 5ª retentativa falha, o worker detecta que o limite máximo de tentativas foi atingido.
  2. Em uma transação, insere um registro na tabela `webhook_dead_letter` contendo: identificador original, `webhookId`, payload JSON original congelado, código de erro HTTP final, motivo da falha e contagem de tentativas (5).
  3. Atualiza o status do registro na tabela `webhook_outbox` para `FAILED`.
  4. Emite log estruturado de erro alertando a falha definitiva de entrega.
* **Prioridade:** Alta

### RF-010: Replay Manual de DLQ por Administrador
* **Descrição:** Permite que operadores com permissão de nível `ADMIN` reenfileirem eventos retidos na DLQ após o cliente corrigir seu ambiente.
* **Fluxo Principal:**
  1. O administrador envia `POST /admin/webhooks/dead-letter/:id/replay`.
  2. O middleware `requireRole('ADMIN')` valida os privilégios do token JWT.
  3. O sistema recupera o registro na tabela `webhook_dead_letter`.
  4. Recria o registro na tabela `webhook_outbox` com status `PENDING`, `attempt_count = 0` e `next_retry_at = null`.
  5. Marca o registro na DLQ como reprocessado (`replayedAt = NOW()`).
  6. Grava log de auditoria estruturado com o `userId` do administrador, timestamp e IDs envolvidos.
  7. Retorna HTTP 200 OK com confirmação da operação.
* **Exceções e Erros:**
  * Usuário com perfil `OPERATOR`: retorna HTTP 403 Forbidden com código `FORBIDDEN_ROLE`.
  * Evento de DLQ inexistente: retorna HTTP 404 Not Found com código `WEBHOOK_DLQ_NOT_FOUND`.
  * Evento já reprocessado: retorna HTTP 409 Conflict com código `WEBHOOK_DLQ_ALREADY_REPROCESSED`.
* **Prioridade:** Média

### RF-011: Consulta de Histórico de Entregas
* **Descrição:** Permite consultar o histórico detalhado de tentativas de disparo de um webhook.
* **Fluxo Principal:**
  1. O usuário envia `GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=20`.
  2. O sistema consulta a tabela `webhook_delivery_history` e retorna a listagem paginada informando: `eventId`, `attemptNumber`, `httpStatus`, `responseDurationMs`, `success`, `errorMessage` e `createdAt`.
  3. Responde com HTTP 200 OK no envelope padronizado.
* **Prioridade:** Média

---

## 7. Requisitos Não Funcionais

### 7.1. Performance e Latência
* **RNF-001 (Latência de Despacho):** A latência total entre o commit da transação no banco e a chegada da requisição HTTP ao endpoint do cliente deve ser inferior a 10 segundos no p95 (com meta média entre 2 e 4 segundos em condições normais de rede).
* **RNF-002 (Timeout Rigoroso):** Cada chamada HTTP realizada pelo worker deve possuir timeout estrito de 10 segundos configurado com `AbortController` nativo do Node.js.

### 7.2. Segurança e Controle de Acesso
* **RNF-003 (HTTPS Obrigatório):** O cadastro de webhooks deve rejeitar obrigatoriamente qualquer URL que não utilize o protocolo seguro `https://`.
* **RNF-004 (Autenticidade e Integridade HMAC):** Toda requisição externa deve ser autenticada via hash HMAC-SHA256 calculado sobre o payload bruto e assinado com secret de 32 bytes exclusiva do endpoint.
* **RNF-005 (Privacidade de Segredos):** As credenciais de webhook (`secret` e `previous_secret`) devem ser exibidas exclusivamente no momento da criação ou rotação, sendo omitidas em qualquer outra consulta ou listagem e mascaradas nos logs estruturados do Pino.
* **RNF-006 (Controle Baseado em Cargos — RBAC):** O acesso a rotas de replay da DLQ exige estritamente a claim de autorização de cargo `ADMIN`.
* **RNF-007 (Teto de Tamanho de Payload):** O corpo do evento serializado em JSON não pode ultrapassar 64 KB.

### 7.3. Confiabilidade e Tolerância a Falhas
* **RNF-008 (Garantia At-Least-Once):** Nenhum evento confirmado no banco relacional pode ser perdido. A garantia de entrega é do tipo *at-least-once*, sendo acompanhada do cabeçalho `X-Event-Id` (UUID v4) para permitir desduplicação idempotente pelo receptor.
* **RNF-009 (Consistência Transacional Estrita):** A outbox deve ser populada dentro do mesmo bloco `prisma.$transaction` do pedido. Se a atualização de status ou estoque falhar, o evento na outbox não deve existir.

### 7.4. Observabilidade e Auditoria
* **RNF-010 (Logs Estruturados):** Toda a operação de outbox, worker e retries deve emitir logs estruturados em JSON via biblioteca Pino, contendo `requestId`, `eventId`, `webhookId`, `customerId`, `attemptCount`, `httpStatus` e `durationMs`.
* **RNF-011 (Métricas Operacionais):** Exposição de contadores e medidores para monitorar a volumetria da outbox (`webhook_outbox_pending_gauge`), taxa de entrega e eventos descartados na DLQ.

---

## 8. Decisões e Trade-offs Principais

### Decisão 1: Transactional Outbox em MySQL em vez de Broker de Mensageria Dedicado (Kafka / RabbitMQ / Redis)
* **Justificativa:** A infraestrutura existente roda com MySQL 8.0 estável e a equipe de engenharia é enxuta. A outbox no MySQL assegura atomicidade com a transação de pedidos sem introduzir novos clusters de infraestrutura ou complexidade de consistência Dual-Write.
* **Trade-off:** O worker realiza consultas regulares de polling no banco a cada 2 segundos, demandando índices compostos otimizados para evitar concorrência com as tabelas de pedidos.

### Decisão 2: Worker Desacoplado em Processo Node.js Independente (`src/worker.ts`)
* **Justificativa:** Isola o consumo e o tráfego de rede para a internet das threads do servidor HTTP Express, garantindo que indisponibilidades ou lentidões de clientes não travem o atendimento de novas requisições da API.
* **Trade-off:** Exige a orquestração e monitoramento de um processo ou container adicional no ambiente de deploy (`npm run worker`).

### Decisão 3: Snapshot de Payload Serializado em JSON no Momento da Mutação
* **Justificativa:** Congela o estado exato do pedido no instante da transição de status, impedindo que mutações comerciais posteriores distorçam os dados transmitidos no webhook.
* **Trade-off:** Eleva o consumo de espaço em disco na tabela `webhook_outbox` em comparação com o armazenamento de apenas chaves estrangeiras.

### Decisão 4: Autenticação Criptográfica HMAC-SHA256 com Secret por Endpoint e Carência de 24h
* **Justificativa:** Padrão consolidado adotado por referências globais de API (Stripe, GitHub), provendo integridade e contendo o raio de explosão em caso de vazamento. O período de carência de 24 horas garante rotação de credenciais sem indisponibilidade de serviço.
* **Trade-off:** Requer armazenamento de colunas adicionais para chave anterior e expiração na tabela de configuração.

---

## 9. Dependências

### 9.1. Dependências Organizacionais e de Segurança
* **Revisão Formal de Segurança (Engenheira Sofia):** Bloqueio obrigatório de 2 dias úteis na Sprint 3 para auditoria formal do gerador de segredos criptográficos, rotina de HMAC e permissões de DLQ antes da liberação em produção.

### 9.2. Dependências Externas
* **Homologação e Piloto com Clientes B2B:** Disponibilização da especificação técnica e credenciais em ambiente de homologação (*staging*) para os times de engenharia da Atlas Comercial, Max Distribuição e Nova Cargo configurarem seus receptores.

### 9.3. Dependências Técnicas e de Código Base
* **Refatoração Cirúrgica em `src/modules/orders/order.service.ts`:** Exposição controlada do cliente transacional `tx` no bloco `prisma.$transaction` para acoplamento limpo da função `publishWebhookEvent`.
* **Reuso da Infraestrutura Transversal:** Alinhamento direto com as classes de erro `AppError`, logger Pino configurado e middleware `requireRole`.

---

## 10. Riscos e Mitigação

| Risco Identificado | Probabilidade | Impacto | Estratégia de Mitigação | Plano de Contingência |
| :--- | :--- | :--- | :--- | :--- |
| **Cliente corporativo com servidor fora do ar por tempo prolongado sobrecarregando retries** | Média | Médio | Aplicação de backoff exponencial em 5 níveis com intervalo total de ~15h e descarte automático para a DLQ após a 5ª tentativa. | Eventos retidos na tabela `webhook_dead_letter` podem ser reprocessados manualmente por administradores via endpoint dedicado quando o cliente normalizar seu servidor. |
| **Vazamento acidental de chave secreta em logs do cliente B2B** | Média | Alto | Geração de secrets exclusivas por endpoint (o vazamento em um cliente não afeta outros) e disponibilização de endpoint de autoatendimento para rotação imediata de chaves com período de carência de 24h. | Revogação emergencial da secret via patch administrativo desativando temporariamente o endpoint até a substituição definitiva. |
| **Degradação de performance do MySQL devido ao polling contínuo do worker a cada 2s** | Baixa | Alto | Criação de índice composto cobrindo `(status, next_retry_at, created_at)` na tabela `webhook_outbox`, leituras em lotes pequenos (20 a 50 registros) e pool de conexões Prisma exclusivo para o worker. | Parametrização dinâmica do intervalo de polling via variável de ambiente (`POLLING_INTERVAL_MS`), permitindo elevar o intervalo para 5 segundos sem necessidade de novo deploy. |
| **Queda silenciosa do processo worker deixando notificações represadas** | Baixa | Alto | Monitoramento de métricas operacionais e configuração de política de reinício automático no orquestrador (Docker restart policy ou PM2). | Emissão de alerta operacional caso a métrica `webhook_outbox_pending_gauge > 100` por mais de 5 minutos, disparando acionamento da equipe de plantão. |

---

## 11. Critérios de Aceitação

* [x] A tabela `webhook_outbox` é populada exclusivamente dentro da transação SQL do método `OrderService.changeStatus`.
* [x] Se a transação de mudança de status sofrer rollback por qualquer motivo de negócio ou erro relacional, nenhum registro de webhook persiste no banco de dados.
* [x] O processo worker executa de forma desacoplada em `src/worker.ts` consumindo registros pendentes a cada 2 segundos.
* [x] O cabeçalho `X-Signature` contém a assinatura HMAC-SHA256 calculada sobre o payload bruto do JSON utilizando a secret do endpoint.
* [x] O cadastro de webhooks rejeita URLs que utilizem o protocolo HTTP inseguro, retornando código `WEBHOOK_INVALID_URL`.
* [x] O ciclo de retentativas respeita estritamente os 5 intervalos exponenciais (1m, 5m, 30m, 2h, 12h) com timeout de 10 segundos por chamada HTTP.
* [x] Eventos que ultrapassam 5 tentativas de entrega são transferidos para a tabela `webhook_dead_letter` com motivo detalhado e payload preservado.
* [x] O endpoint `POST /admin/webhooks/dead-letter/:id/replay` aceita requisições exclusivamente com cargo `ADMIN`, rejeitando `OPERATOR` com HTTP 403 Forbidden.
* [x] A rotação de chave secreta gera uma nova secret e mantém a chave anterior aceita para validação por exatamente 24 horas.
* [x] O histórico de entregas (`GET /api/v1/webhooks/:id/deliveries`) retorna o tempo de resposta em milissegundos e o código de status HTTP de cada tentativa.
* [x] Payloads montados cujo tamanho serializado ultrapasse 64 KB são rejeitados antes do despacho.
* [x] Todos os novos códigos de erro de domínio do módulo adotam o prefixo padronizado `WEBHOOK_`.

---

## 12. Estratégia de Testes e Validação

### 12.1. Tipos de Teste Obrigatórios
* **Testes Unitários:** Validação dos esquemas Zod de entrada, cálculo do hash HMAC-SHA256, verificação da tolerância de 24 horas de rotação e cálculo dos intervalos de backoff exponencial.
* **Testes de Integração com Banco MySQL (Vitest + Supertest):** Validação da persistência transacional atômica entre `OrderService.changeStatus` e a gravação na `webhook_outbox`, confirmando a inexistência de registros órfãos em caso de rollback forçado.
* **Testes de Contratos de API:** Validação dos endpoints de CRUD de webhooks, autenticação JWT, bloqueio de perfil `OPERATOR` no replay de DLQ e formatação padronizada de erros de domínio via `AppError`.
* **Testes de Simulação do Worker:** Execução ponta a ponta do loop do worker simulando servidores receptores externos (cenários de sucesso 200, falhas com retry 500, timeouts de rede acima de 10s e transferência para a DLQ após 5 falhas).
* **Testes de Segurança:** Tentativa de cadastro de URLs HTTP em texto aberto e injeção de caracteres para assegurar sanitização no logger Pino.

### 12.2. Homologação e Revisão
* **Revisão de Código de Segurança:** Validação formal dos módulos criptográficos conduzida pela engenheira Sofia.
* **Teste Piloto em Staging:** Integração prévia assistida com as equipes de tecnologia da Atlas Comercial, Max Distribuição e Nova Cargo antes da ativação em produção.
