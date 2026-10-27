# ADR-002: Worker Desacoplado em Processo Separado com Polling

**Status:** Aceito
**Data:** 2026-10-26
**ADRs Relacionados:** Depende de [ADR-001: Padrão Outbox no MySQL para Notificações de Pedidos](./ADR-001-padrao-outbox-no-mysql.md); Usado por [ADR-003: Política de Retentativas com Backoff Exponencial e Dead Letter Queue](./ADR-003-politica-retry-backoff-dlq.md); Relacionado a [ADR-004: Autenticação Criptográfica HMAC-SHA256 com Secret por Endpoint](./ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md); Relacionado a [ADR-005: Garantia de Entrega At-Least-Once com Header de Idempotência](./ADR-005-garantia-at-least-once-idempotencia.md)

## Contexto e Definição do Problema
Com a adoção do padrão Transactional Outbox (ADR-001), os eventos de alteração de pedidos são enfileirados como registros relacionais na tabela `webhook_outbox`. É indispensável um mecanismo de software responsável por ler esses registros, computar assinaturas criptográficas e realizar os disparos de rede via HTTP POST para os endpoints externos dos clientes.

O envio de requisições pela internet para servidores externos apresenta latências imprevisíveis e riscos de travamento de sockets. Se essa rotina fosse acoplada dentro do mesmo processo Node.js da API Express (por exemplo, via `setInterval` compartilhado), reinicializações da API interromperiam o envio de notificações, e o consumo de conexões e event loop poderia degradar a performance das rotas de pedidos e produtos atendidas pelo OMS.

## Fatores de Decisão
- Isolar o ciclo de vida e a memória do despachante assíncrono em relação à API HTTP pública.
- Garantir que instabilidades e lentidões de rede em clientes externos não afetem o atendimento do OMS.
- Manter a latência total de notificação abaixo do teto de 10 segundos no p95 (meta operacional entre 2 e 4 segundos).
- Prover pool de conexões Prisma exclusivo e isolado para o consumidor de eventos.

## Opções Consideradas
1. Processo Node.js dedicado (`src/worker.ts`), executado via script `npm run worker`, operando em polling com intervalo de 2 segundos.
2. Agendador em memória ou rotina `setInterval` executada na mesma instância do processo da API Express.
3. Função serverless sob demanda (AWS Lambda / Cloud Functions) acionada por agendador externo.

## Resultado da Decisão
Opção escolhida: "Processo Node.js dedicado com polling a cada 2 segundos", porque proporciona isolamento físico completo entre a camada de apresentação web e as tarefas de background. O worker opera em loop infinito consultando a tabela `webhook_outbox` ordenando por `created_at` e atualizando o status para `PROCESSING`.

Essa abordagem garante que a API Express permaneça stateless e focada em responder com baixa latência, enquanto o worker gerencia seu próprio ciclo de conexões, timeouts e retentativas sem interferência mútua.

## Prós e Contras das Opções

### Processo Node.js dedicado com polling a cada 2 segundos
- Prós: Isolamento de falhas (a queda ou restart da API não afeta o worker); cota isolada de conexões no banco; simplicidade de desenvolvimento reutilizando o mesmo ecossistema TypeScript/Prisma; atende com folga ao requisito de latência < 10s.
- Contras: Exige o gerenciamento e monitoramento de um processo ou container Node.js adicional no pipeline de deploy; polling regular gera consultas a cada 2s no banco de dados.

### Rotina na mesma instância da API (setInterval / cron interno)
- Prós: Dispensa a configuração de novos processos ou containers de inicialização.
- Contras: Acoplamento de ciclo de vida; risco de contenção no event loop do Node.js; se a API reiniciar por deploy ou crash, o envio de notificações é paralisado.

### Função serverless / Lambda orientada a eventos
- Prós: Desligamento quando ocioso e escala automática elástica.
- Contras: Complexidade de infraestrutura desnecessária; latência de cold start; dificuldade de teste em ambiente local; custo desproporcional.

## Consequências
- Efeitos positivos: Estabilidade da API preservada; resiliência nas entregas; facilidade de auditoria e escalabilidade futura independente entre API e worker.
- Efeitos negativos: O ambiente de produção precisa orquestrar dois serviços distintos (`npm start` para o servidor web e `npm run worker` para o despachante de webhooks).
- Efeitos operacionais: Implementação de política de restart automático (Docker restart policy ou PM2) e métricas de heartbeat para alertar caso o worker pare de consumir.

## Referências
- `src/server.ts:1-40` - Ponto de entrada atual da API Express preservado sem background jobs.
- `package.json:10-21` - Scripts do projeto onde o novo comando `worker` é registrado.
- `TRANSCRICAO.md:58-76` - Discussão sobre separação de processos e polling de 2 segundos.
