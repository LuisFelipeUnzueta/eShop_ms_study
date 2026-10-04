# Lista de Decisões (ADR) — Order

> **Instruções para a LLM**
>
> - [x] Substitua os placeholders `{...}` pelos dados reais.
> - [x] Registre decisões reais do projeto: problema, opções e motivo da escolha.
> - [x] **Evite** listar tecnologias soltas ("Usa Kafka", "Usa DDD", "Usa repository").
> - [x] **Prefira** explicar o porquê: "O Kafka foi usado para desacoplar a confirmação do pedido do processo de notificação, evitando que uma indisponibilidade do serviço de notificações impeça a confirmação."
> - [x] Numere os ADRs sequencialmente: ADR-001, ADR-002, ...
> - [x] Se, ao registrar uma decisão, você perceber que ela já apareceu em outro projeto estudado, adicione uma nota em "Decisões recorrentes" (fim deste arquivo) apontando para `../../licoes-aprendidas.md`.

## Etapa 1 — Identificar decisões durante a leitura

- [x] Sempre que algo poderia ter sido implementado de outra forma, registre uma decisão.

- [x] Método no agregado ou no handler? → **Agregado** (regras em `Order`/`OrderItem`), handler só orquestra.
- [x] REST ou mensageria? → **REST para criar, mensageria para efeitos colaterais**.
- [x] Evento de domínio ou integração? → **Ambos**: domínio dentro do processo, integração no bus.
- [x] Publicar diretamente ou usar Outbox? → **Outbox** (tabela `IntegrationEventLog` na mesma transação).
- [x] SSE ou polling? → SSE para notificação de status (no `WebApp`, consumidor do evento).
- [x] EF Core ou Dapper? → **EF Core** (leitura/escrita com o mesmo DbContext; queries dedicadas usam Dapper no `OrderQueries`).
- [x] Monólito ou microsserviço? → **Microsserviços** (Ordering.API, Basket.API, Catalog.API, PaymentProcessor, OrderProcessor, WebApp, Webhooks.API).
- [x] Estado calculado ou persistido? → **Persistido** (`OrderStatus`/`Description` são propriedades do agregado).
- [x] Estratégia de idempotência escolhida? → **Chave de idempotência `x-requestid` persistida em `ClientRequest`**.
- [x] Estratégia de controle de concorrência escolhida? → **Otimista** (Value Object `PaymentMethod.ObjectVersion`; sem versão explícita no `Order`).
- [x] {Outra pergunta de decisão específica do projeto} → Command imutável (`CreateOrderCommand`) com setters privados.

## Etapa 2 — Perguntas fixas (responda para cada decisão)

- [x] 1. Qual problema precisava ser resolvido?
- [x] 2. Qual decisão foi tomada?
- [x] 3. Quais alternativas existiam?
- [x] 4. Quais benefícios foram obtidos?
- [x] 5. Quais custos ou limitações surgiram?
- [x] 6. Em quais condições essa decisão deixaria de fazer sentido?

## Etapa 3 — Modelo ADR reduzido

- [x] Duplique este modelo para cada decisão registrada.

## ADR-001 — Idempotência via chave `x-requestid` persistida em `ClientRequest`

### Contexto

- [x] Um cliente pode duplicar a requisição de criação de pedido (retry de rede, duplo clique no checkout). Sem proteção, o mesmo carrinho geraria dois pedidos e o carrinho seria apagado duas vezes.

### Decisão

- [x] O endpoint exige o header `x-requestid`; o comando é envolvido em `IdentifiedCommand<CreateOrderCommand, bool>` e o `IdentifiedCommandHandler` grava o GUID na tabela `ClientRequest` antes de executar. Duplicatas retornam sucesso sem reprocessar (`CreateOrderIdentifiedCommandHandler.CreateResultForDuplicateRequest()` → `true`). Implementado em `IdentifiedCommandHandler.cs:39-104` e `RequestManager.cs:13-37`.

### Motivo

- [x] Solução simples no nível de aplicação, transparente para o cliente e que protege tanto o banco quanto os efeitos colaterais (publicação de eventos).

### Alternativas consideradas

- [x] Chave natural de negócio (ex.: carrinho + usuário) para upsert.
- [x] Idempotência no banco via constraint única.
- [x] Sem proteção (retry manual).

### Consequências positivas

- [x] Retry seguro e idempotente para o cliente.
- [x] Registro auditável de requisições recebidas (`ClientRequest`).
- [x] Mecanismo reutilizado por `cancel` e `ship`.

### Consequências negativas

- [x] O GUID precisa ser gerado pelo cliente (responsabilidade adicionada).
- [x] Linhas em `ClientRequest` crescem sem política de expurgo (não encontrada nesta versão).
- [x] GUID vazio precisa de checagem explícita no endpoint (senão cria 400 manual).

### Quando reconsiderar

- [x] Se houver proxy/edge que já garanta deduplicação, ou se a tabela exigir política de retenção.

---

## ADR-002 — Publicação de eventos de integração via Outbox na mesma transação

### Contexto

- [x] A criação do pedido precisa notificar `Basket.API` (apagar carrinho) e `WebApp` (atualizar status). Publicar o evento ao RabbitMQ antes do commit poderia gerar "evento fantasma" se a transação falhasse; publicar depois do commit arrisca perder o evento se o bus cair.

### Decisão

- [x] Eventos de integração são gravados na tabela `IntegrationEventLog` (outbox) dentro da mesma transação do banco (`OrderingIntegrationEventService.AddAndSaveEventAsync`, usando `GetCurrentTransaction()`). Após o commit, `TransactionBehavior` chama `PublishEventsThroughEventBusAsync(transactionId)` que lê os eventos pendentes e os publica, marcando `InProgress` → `Published`/`PublishedFailed`. Implementado em `OrderingIntegrationEventService.cs:13-41`, `TransactionBehavior.cs:52` e `IntegrationEventLogService.cs`.

### Motivo

- [x] Garante consistência entre a persistência do pedido e a intenção de notificar, com a mesma atomicidade do banco; publicação acontece após o commit (consistência eventual entre microsserviços).

### Alternativas consideradas

- [x] Publicar direto no RabbitMQ no handler (síncrono antes/depois do commit).
- [x] Publicar após o commit sem registro em tabela (fire-and-forget).
- [x] Usar transação distribuída (2PC / saga orquestrada).

### Consequências positivas

- [x] Sem eventos fantasmas: nada é publicado se o pedido não foi persistido.
- [x] Eventos pendentes ficam registrados (rastreabilidade e base para reconciliação).
- [x] Cliente recebe `200 OK` assim que a transação é commitada, sem depender do bus.

### Consequências negativas

- [x] Nesta versão não há worker de redispatch automático — eventos marcados `PublishedFailed` ficam pendentes de reconciliação manual.
- [x] Camada extra de tabela (`IntegrationEventLog`) e lógica no `TransactionBehavior`.
- [x] Publicação ocorre na sequência da transação (single-process), limitando throughput se o volume for alto.

### Quando reconsiderar

- [x] Se o volume exigir publicador assíncrono dedicado/background, ou se o broker oferecer transações com o banco (ex.: RabbitMQ Streams + hooks).

---

## ADR-003 — Regras de negócio no agregado, não no handler nem no endpoint

### Contexto

- [x] A criação do pedido precisa garantir invariantes: itens com unidades válidas, desconto coerente, mescla de itens repetidos. Essas regras poderiam ser validadas no endpoint ou no handler.

### Decisão

- [x] As regras vivem no agregado `Order`/`OrderItem` (construtor e `AddOrderItem`), e o `CreateOrderCommandHandler` apenas orquestra: cria `Address`, `Order`, chama `AddOrderItem` e persiste. Implementado em `Order.cs:52-91` e `OrderItem.cs:23-62`.

### Motivo

- [x] O agregado é a fronteira de consistência: garante que qualquer canal que crie pedidos (API, teste, script) passe pelas mesmas regras.

### Alternativas consideradas

- [x] Validar no endpoint (`CreateOrderRequest`).
- [x] Validar no handler com `if`s.
- [x] Serviço de domínio separado.

### Consequências positivas

- [x] Regra centralizada e difícil de burlar.
- [x] Testes unitários simples (`OrderAggregateTest`).
- [x] O agregado comunica a intenção de negócio.

### Consequências negativas

- [x] O agregado concentra comportamento; cresce com cada novo fluxo de status.
- [x] A persistência precisa reconstruir o estado fielmente (EF Core config das entidades).

### Quando reconsiderar

- [x] Se uma regra passar a depender de múltiplos agregados ou de serviços externos.

---

## ADR-004 — Dispatch de domain events antes do `SaveChanges` (mesma transação)

### Contexto

- [x] Ao criar o pedido, o `OrderStartedDomainEvent` precisa atualizar `Buyer`/`PaymentMethod` antes do commit — e esses dados participam da mesma unidade de consistência.

### Decisão

- [x] `OrderingContext.SaveEntitiesAsync` chama `mediator.DispatchDomainEventsAsync(this)` ANTES de `SaveChangesAsync`, executando os handlers de domínio dentro da mesma transação (Opção A do comentário em `OrderingContext.cs:49-55`). Implementado em `OrderingContext.cs:47-62` e `MediatorExtension.cs:5-20`.

### Motivo

- [x] Side effects de domínio (criar Buyer, associar PaymentMethod) devem commitar junto com o pedido — uma única transação, sem necessidade de compensação.

### Alternativas consideradas

- [x] Dispatch após o commit (Opção B) — múltiplas transações e consistência eventual com compensação.
- [x] Executar a lógica do Buyer diretamente no handler do command.

### Consequências positivas

- [x] Pedido + Buyer + PaymentMethod + eventos de integração commitam atomicamente.
- [x] Sem compensação manual em falhas de domínio.

### Consequências negativas

- [x] Handler de domínio executa `SaveEntitiesAsync` aninhado (reentrância no mesmo `DbContext`), frágil se a ordem de geração de IDs mudar (ver `REVIEW` em `ValidateOrAddBuyerAggregateWhenOrderStartedDomainEventHandler.cs:31-32`).

### Quando reconsiderar

- [x] Se o handler de domínio passar a fazer I/O externo lento ou se o acoplamento a IDs gerados (HiLo) quebrar.

---

## ADR-005 — Command imutável com setters privados

### Contexto

- [x] Commands que carregam dados sensíveis (número de cartão) e dados de negócio trafegam pelo pipeline do MediatR e pelo serializador; mutação acidental poderia corromper o caso de uso.

### Decisão

- [x] `CreateOrderCommand` é imutável: setters privados, coleção de itens `IEnumerable` exposta como leitura, construção única pelo construtor (`CreateOrderCommand.cs`). O número do cartão é mascarado antes de entrar no command (`OrdersApi.cs:140`).

### Motivo

- [x] Commands representam intenções pontuais; imutabilidade reduz bugs de pipeline e segue a recomendação de CQRS.

### Alternativas consideradas

- [x] Record com `init` (usado em eventos de integração).
- [x] DTO mutável.

### Consequências positivas

- [x] Predição de comportamento em qualquer estágio do pipeline.
- [x] Segurança por redução de superfície (cartão não é logado).

### Consequências negativas

- [x] Boilerplate de construtor com muitos parâmetros.
- [x] Dados que precisam de derivação (ex.: itens) exigem lógica de conversão externa (`ToOrderItemsDTO`).

### Quando reconsiderar

- [x] Se o framework de serialização exigir setters públicos (ex.: System.Text.Json com source-gen restrito).

---

## Decisões recorrentes (cross-project)

> Preencha apenas se, durante este estudo, você notou que uma decisão já havia aparecido em projeto anterior.

| ADR deste estudo | Já visto em | Nota |
|-------------------|-------------|------|
| ADR-002 (Outbox) | — | Ver `../../licoes-aprendidas.md#outbox-pattern` |
| ADR-004 (Domain events antes do SaveChanges) | — | Ver `../../licoes-aprendidas.md#domain-events-no-savechanges` |
| ADR-001 (Idempotência com request-id) | — | Ver `../../licoes-aprendidas.md#idempotencia-request-id` |
