# ADR-003: Política de Retentativas com Backoff Exponencial e Dead Letter Queue

**Status:** Aceito
**Data:** 2026-10-26
**ADRs Relacionados:** Depende de [ADR-002: Worker Desacoplado em Processo Separado com Polling](./ADR-002-worker-separado-em-polling.md); Relacionado a [ADR-001: Padrão Outbox no MySQL para Notificações de Pedidos](./ADR-001-padrao-outbox-no-mysql.md); Relacionado a [ADR-005: Garantia de Entrega At-Least-Once com Header de Idempotência](./ADR-005-garantia-at-least-once-idempotencia.md)

## Contexto e Definição do Problema
O disparo de notificações para servidores externos de clientes B2B está sujeito a falhas transitórias de conectividade, indisponibilidades momentâneas de rede e janelas planejadas de manutenção que, no ecossistema corporativo, chegam a durar entre 1 e 2 horas. Se o sistema realizar apenas tentativas imediatas ou com intervalos muito curtos, o envio falhará em definitivo antes do restabelecimento do cliente.

Por outro lado, adotar retentativas infinitas sobrecarregaria a tabela de outbox com registros de clientes desativados ou com URLs inválidas, represando o loop do worker e consumindo processamento inútil. É necessário um protocolo balanceado de resiliência com encerramento determinístico e retenção de falhas permanentes para diagnóstico e intervenção.

## Fatores de Decisão
- Prover cobertura de resiliência com janela temporal suficiente para superar manutenções programadas de clientes (> 12 horas).
- Evitar o represamento indefinido de mensagens falhas na fila de processamento da outbox principal.
- Estabelecer timeout rigoroso por chamada HTTP para impedir que conexões travadas bloqueiem o worker.
- Disponibilizar mecanismo auditável e seguro para que administradores consigam reprocessar manualmente eventos que esgotaram as tentativas.

## Opções Consideradas
1. Cinco retentativas com backoff exponencial (1m, 5m, 30m, 2h, 12h) e isolamento em tabela `webhook_dead_letter` com replay restrito a perfil `ADMIN`.
2. Três retentativas em intervalos curtos (1m, 5m, 15m) com marcação de falha na própria tabela outbox.
3. Retentativas contínuas e indefinidas com intervalo fixo de repetição sem Dead Letter Queue.

## Resultado da Decisão
Opção escolhida: "Cinco retentativas com backoff exponencial e tabela de Dead Letter Queue dedicada", porque equilibra alta resiliência e proteção dos recursos computacionais. A progressão de intervalos (1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas) estende a janela de tolerância para aproximadamente 15 horas entre o primeiro erro e a última tentativa.

Além disso, ao atingir a quinta falha consecutiva, o evento é transferido para a tabela `webhook_dead_letter` com diagnóstico detalhado (payload original, último código HTTP e motivo da falha). Isso mantém a tabela `webhook_outbox` limpa para consultas de alta performance e permite o replay manual auditado através do endpoint `POST /admin/webhooks/dead-letter/:id/replay`.

## Prós e Contras das Opções

### Cinco retentativas com backoff exponencial e DLQ dedicada
- Prós: Janela de resiliência ampla (~15 horas); isolamento físico de eventos mortos; tabela outbox desimpedida; governança e rastreabilidade através de endpoint de replay exclusivo para `ADMIN`.
- Contras: Exige persistência de timestamps (`next_retry_at`) e modelagem de tabela adicional para a DLQ.

### Três retentativas curtas (1m, 5m, 15m)
- Prós: Resolução rápida do ciclo de vida da mensagem.
- Contras: Insuficiente para cobrir manutenções de rotina de parceiros corporativos (que costumam durar 1 a 2 horas), gerando falsos positivos de falhas permanentes.

### Retentativas indefinidas sem DLQ
- Prós: Nenhuma mensagem é descartada automaticamente.
- Contras: Fila eternamente poluída por clientes que desativaram seus servidores; degradação contínua do polling no banco relacional.

## Consequências
- Efeitos positivos: Resiliência garantida para a grande maioria das falhas transitórias; intervenção administrativa controlada para incidentes permanentes; sem perda silenciosa de dados.
- Efeitos negativos: O cliente receptor pode receber eventos horas após a ocorrência caso seu ambiente fique instável, reforçando a necessidade do cabeçalho de timestamp.
- Efeitos operacionais: Monitoramento de métricas de descarte em DLQ (`webhook_dead_letter_total`) para acionar o suporte ao cliente caso ocorram falhas contínuas.

## Referências
- `docs/FDD.md:270-295` - Especificação técnica da máquina de retentativas e DLQ.
- `docs/RFC.md:74-76` - Regra de negócio de 5 tentativas e backoff exponencial.
- `TRANSCRICAO.md:90-116` - Deliberação técnica entre Diego, Bruno e Sofia definindo as 5 etapas.
