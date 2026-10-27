# ADR-004: Autenticação Criptográfica HMAC-SHA256 com Secret por Endpoint

**Status:** Aceito
**Data:** 2026-10-26
**ADRs Relacionados:** Relacionado a [ADR-002: Worker Desacoplado em Processo Separado com Polling](./ADR-002-worker-separado-em-polling.md); Relacionado a [ADR-006: Reuso dos Padrões de Código e Arquitetura Existentes](./ADR-006-reuso-padroes-arquiteturais-existentes.md)

## Contexto e Definição do Problema
As notificações de pedidos são enviadas pela internet pública para servidores externos de clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo). Esses eventos contêm informações comerciais relevantes, como identificador de pedido, status de pagamento e valores agregados em centavos.

Sem um mecanismo confiável de autenticação e integridade, os endpoints receptores dos clientes estariam vulneráveis a ataques de personificação (onde invasores enviam notificações forjadas de pedidos pagos para liberar mercadorias) ou ataques *man-in-the-middle* (onde dados do pedido são adulterados em trânsito). Ademais, utilizar uma chave secreta global compartilhada por toda a plataforma representaria um risco catastrófico: o vazamento de credenciais por parte de um único cliente comprometeria a segurança de toda a base de clientes do OMS.

## Fatores de Decisão
- Assegurar a autenticidade da origem (comprovar que a mensagem partiu exclusivamente do OMS).
- Garantir a integridade estrita do payload (detectar qualquer adulteração de campos no trânsito).
- Conter o raio de explosão (*blast radius*) associado a vazamentos de segredos via isolamento por endpoint.
- Fornecer suporte nativo a rotação de chaves sem indisponibilidade de serviço através de período de tolerância (*grace period* de 24 horas).
- Adotar padrão consolidado e amplamente suportado na indústria de software corporativo (como Stripe e GitHub).

## Opções Consideradas
1. Assinatura digital simétrica HMAC-SHA256 enviada no cabeçalho `X-Signature` com chave secreta exclusiva por endpoint e carência de 24 horas na rotação.
2. Chave secreta única global da plataforma compartilhada entre todos os clientes.
3. Criptografia assimétrica com par de chaves pública e privada (assinatura digital via RSA ou ECDSA).

## Resultado da Decisão
Opção escolhida: "Assinatura simétrica HMAC-SHA256 com secret única por endpoint e rotação com carência de 24 horas", porque entrega o mais alto nível de segurança operacional com baixa complexidade de integração para os clientes parceiros.

O worker calcula o hash HMAC-SHA256 sobre o corpo serializado do JSON utilizando a secret do webhook e o envia no cabeçalho `X-Signature`. Cada endpoint possui sua própria secret gerada com entropia criptográfica (`crypto.randomBytes`). Quando o cliente solicita rotação via `POST /api/v1/webhooks/:id/rotate-secret`, a secret anterior permanece válida por 24 horas (`previous_secret`), permitindo ao cliente atualizar seus servidores sem perder notificações durante a janela de transição.

## Prós e Contras das Opções

### HMAC-SHA256 com secret individual por endpoint e carência de 24 horas
- Prós: Padrão consagrado da indústria; contenção total de danos em vazamentos acidentais; integridade e autenticidade comprovadas; rotação suave sem parada de serviço; biblioteca de validação nativa em qualquer linguagem backend dos clientes.
- Contras: Custo computacional de computar hash HMAC em cada disparo no worker; necessidade de gerenciar `previous_secret` e timestamp de expiração no modelo de dados.

### Secret estática global da plataforma
- Prós: Facilidade de configuração sem persistência de chaves múltiplas no banco de dados.
- Contras: Risco de segurança inaceitável; se um cliente registrar sua secret em logs públicos, todos os webhooks de todos os clientes ficam vulneráveis a forjamento.

### Assinatura assimétrica com RSA/ECDSA
- Prós: O cliente só precisa armazenar a chave pública do OMS, sem compartilhar segredos.
- Contras: Maior sobrecarga computacional de assinatura e verificação; maior complexidade para desenvolvedores terceiros implementarem a validação em seus receptores.

## Consequências
- Efeitos positivos: Proteção máxima contra ataques de spoofing e adulteração; conformidade com requisitos de segurança corporativa; autonomia para os clientes rotacionarem credenciais pela API.
- Efeitos negativos: O cliente deve obrigatoriamente implementar o algoritmo de validação HMAC-SHA256 em seu receptor.
- Efeitos operacionais: Revisão mandatória do código de criptografia e rotação conduzida pela engenheira de segurança Sofia antes da liberação em produção.

## Referências
- `docs/PRD.md:58-61` - Requisitos de segurança, HMAC-SHA256 e rotação de 24h.
- `docs/FDD.md:162-180` - Contrato técnico do endpoint de rotação de chaves.
- `TRANSCRICAO.md:118-135` - Decisão conduzida por Sofia estabelecendo HMAC-SHA256 por endpoint.
