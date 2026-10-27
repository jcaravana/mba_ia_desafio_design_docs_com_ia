# ADR-005: Garantia de Entrega At-Least-Once com Header de Idempotência

**Status:** Aceito
**Data:** 2026-10-26
**ADRs Relacionados:** Depende de [ADR-001: Padrão Outbox no MySQL para Notificações de Pedidos](./ADR-001-padrao-outbox-no-mysql.md); Relacionado a [ADR-002: Worker Desacoplado em Processo Separado com Polling](./ADR-002-worker-separado-em-polling.md); Relacionado a [ADR-003: Política de Retentativas com Backoff Exponencial e Dead Letter Queue](./ADR-003-politica-retry-backoff-dlq.md)

## Contexto e Definição do Problema
Em redes de computadores abertas e não confiáveis, falhas de conectividade podem ocorrer após o recebimento e processamento com sucesso de uma requisição pelo servidor do cliente, mas antes que a resposta HTTP 200 alcance o remetente (timeout de rede no retorno). Nesses cenários, o despachante de webhooks não tem como confirmar se o evento foi processado ou se foi perdido no caminho.

Para manter a confiabilidade do sistema e evitar a perda de eventos críticos de pagamento e faturamento de pedidos, o worker é obrigado a disparar retentativas (ADR-003). Como consequência inevitável, o receptor do cliente pode receber a mesma notificação mais de uma vez. Garantir exatamente-uma-entrega (*exactly-once delivery*) em protocolos HTTP distribuídos exigiria coordenação de duas fases complexa e frágil, inadequada para integrações heterogêneas com múltiplos clientes corporativos.

## Fatores de Decisão
- Impedir terminantemente a perda silenciosa de notificações de mudanças de status de pedidos.
- Fornecer identificador imutável e determinístico para viabilizar desduplicação pelo cliente.
- Adotar um contrato de idempotência simples, transparente e padronizado na indústria.
- Evitar a complexidade arquitetural impraticável de transações distribuídas bilaterais em HTTP.

## Opções Consideradas
1. Garantia de entrega pelo menos uma vez (*at-least-once*) com cabeçalho de idempotência `X-Event-Id` (UUID v4) e carimbo temporal `X-Timestamp`.
2. Garantia de entrega no máximo uma vez (*at-most-once*), sem qualquer retentativa em caso de falha de confirmação.
3. Protocolo de transação distribuída bilateral para tentativa de garantia exatamente-uma-vez (*exactly-once*).

## Resultado da Decisão
Opção escolhida: "Garantia at-least-once com cabeçalho de idempotência X-Event-Id", porque garante que nenhum evento confirmado no banco de dados seja abandonado ou perdido por instabilidades transitórias de rede.

No momento em que a alteração de status do pedido é persistida, a outbox gera um UUID v4 imutável para aquele evento. Em todos os despachos e eventuais retentativas daquela notificação, esse identificador é transmitido no cabeçalho `X-Event-Id`. Os clientes parceiros assumem a responsabilidade de verificar se o `X-Event-Id` já foi processado em suas bases locais antes de aplicar mutações em seus sistemas, descartando duplicidades com segurança.

## Prós e Contras das Opções

### Garantia at-least-once com header X-Event-Id
- Prós: Resiliência absoluta contra perda de eventos; padrão adotado universalmente por APIs de webhook (como Stripe, GitHub e Shopify); arquitetura do despachante desacoplada e escalável.
- Contras: O cliente receptor precisa obrigatoriamente persistir os IDs de eventos recebidos para desduplicação idempotente em seu banco de dados.

### Garantia at-most-once (disparo único sem retentativas)
- Prós: Nenhuma possibilidade de entrega duplicada.
- Contras: Perda inaceitável de dados; quedas momentâneas de meio segundo na internet fariam clientes perderem notificações de pagamentos para sempre.

### Tentativa de exactly-once bilateral
- Prós: Eliminação teórica de duplicidades do lado do cliente.
- Contras: Complexidade computacional e de rede extrema; pontos únicos de falha e acoplamento temporal que inviabilizariam a integração com sistemas legados de clientes.

## Consequências
- Efeitos positivos: Nenhuma notificação é perdida; os clientes B2B conseguem implementar filtros de idempotência simples em suas tabelas relacionais baseados em chave única sobre `event_id`.
- Efeitos negativos: Possibilidade de requisições repetidas chegarem ao endpoint do cliente em momentos de lentidão de rede.
- Efeitos operacionais: A documentação do portal de desenvolvedores deve enfatizar e exemplificar como os clientes devem validar o `X-Event-Id` para desduplicação.

## Referências
- `docs/PRD.md:61-62` - Requisito de inclusão de `X-Event-Id` para desduplicação.
- `TRANSCRICAO.md:146-158` - Discussão liderada por Diego e Sofia fundamentando o padrão at-least-once.
- `docs/RFC.md:146-158` - Formalização da entrega at-least-once e descarte de exactly-once.
