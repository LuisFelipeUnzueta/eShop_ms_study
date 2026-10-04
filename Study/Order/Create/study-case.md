# Caso de Estudo — Order (Create)

> **Instruções para a LLM**
>
> - [x] Substitua todos os placeholders `{...}` pelos dados reais do projeto analisado.
> - [x] Preencha os checklists apenas com informações encontradas no código; não invente.
> - [x] Marque itens não aplicáveis como `N/A`, nunca deixe em branco.
> - [x] Se não conseguir confirmar algo com certeza no código, **não preencha por inferência** — registre em "Perguntas em aberto" (Etapa 8).
> - [x] Este arquivo é o sumário: deve permitir leitura isolada, sem abrir o código.
> - [x] A tabela de rastreio de componentes vive em `diagrama-fluxo.md` — não a duplique aqui, apenas referencie.
> - [x] Ao concluir o checklist final, adicione uma linha em `../../_INDEX.md` e, se aplicável, uma entrada em `../../licoes-aprendidas.md`.

## Metadados do estudo

- [x] **Projeto:** eShop (referência .NET — monorepo `eShop_ms_study`)
- [x] **Repositório/commit analisado:** `https://github.com/dotnet/eShop` (cópia local) @ `9b4f943`
- [x] **Data do estudo:** 2026-08-04
- [x] **Stack:** .NET 10 / ASP.NET Core Minimal APIs / MediatR / EF Core + Npgsql (PostgreSQL) / FluentValidation / RabbitMQ (EventBus) / Aspire / Asp.Versioning
- [x] **Entidade:** Order (agregado `Order` em `Ordering.Domain`)
- [x] **Endpoint:** Criar pedido
- [x] **Método HTTP:** POST
- [x] **Rota:** `POST /api/orders` (grupo `api/orders`, versão 1.0)
- [x] **Versão da API:** 1.0 (Asp.Versioning, cabeçalho `api-supported-versions`)

## Objetivo

- [x] Descrever o objetivo do estudo: receber os itens do carrinho + dados de entrega e pagamento do cliente e criar um novo pedido no estado `Submitted`, persistindo Order/OrderItems/Buyer/PaymentMethod e publicando eventos de integração que limpam o carrinho e notificam a WebApp.
- [x] Qual problema de negócio ele resolve? Transforma um carrinho confirmado (checkout) em um pedido persistente com vínculo de comprador e forma de pagamento, iniciando a máquina de estados do pedido.
- [x] Quais outros componentes dependem dele? `Basket.API` (consome `OrderStartedIntegrationEvent` para apagar o carrinho), `WebApp` (consome `OrderStatusChangedToSubmittedIntegrationEvent` para notificação via SSE/OrderStatusNotificationService). Downstream, `Catalog.API` valida estoque quando o pedido avança para `AwaitingValidation`.

## Entrada

- [x] **Body:** `CreateOrderRequest` (`OrdersApi.cs:171`): `UserId`, `UserName`, `City`, `Street`, `State`, `Country`, `ZipCode`, `CardNumber`, `CardHolderName`, `CardExpiration`, `CardSecurityNumber`, `CardTypeId`, `Buyer`, `Items: List<BasketItem>` (`Id`, `ProductId`, `ProductName`, `UnitPrice`, `OldUnitPrice`, `Quantity`, `PictureUrl`).
- [x] **Query params:** N/A
- [x] **Path params:** N/A
- [x] **Headers obrigatórios:** `x-requestid: Guid` (chave de idempotência) — obrigatório e não pode ser `Guid.Empty`.
- [x] **Autenticação/Autorização:** JWT Bearer — grupo exige `.RequireAuthorization()` (`Program.cs:22`), autenticação padrão via `AddDefaultAuthentication()`.

### Pré-condições

- [x] Estado anterior exigido da Order: N/A (criação parte do zero). O usuário deve estar autenticado e ter carrinho com itens.
- [x] Dependências externas esperadas: PostgreSQL (`orderingdb`), RabbitMQ (`eventbus`), Identity provider (issuer do JWT). A `OrderingContextSeed` popula `CardTypes` e statuses de exemplo.

## Resultado esperado (caminho feliz)

- [x] **Status code de sucesso:** `200 OK` (`TypedResults.Ok()` — `OrdersApi.cs:166`)
- [x] **Body de resposta:** vazio (resultado booleano `true` interno, sem retorno serializado)
- [x] Order muda de `(inexistente)` para `Submitted` (construtor `Order(...)` seta `OrderStatus = OrderStatus.Submitted`)
- [x] Alteração é persistida em `orderingdb` — tabelas `Ordering.orders`, `Ordering.orderitems`, `Ordering.buyers`, `Ordering.paymentmethods`, `Ordering.cardtypes`, `Ordering.clientrequest`, `IntegrationEventLog` (outbox)
- [x] Eventos `OrderStartedIntegrationEvent` e `OrderStatusChangedToSubmittedIntegrationEvent` são publicados no bus RabbitMQ (via outbox, após commit)
- [x] Interessados notificados: `Basket.API` (apaga carrinho do usuário) e `WebApp` (`OrderStatusNotificationService.NotifyOrderStatusChangedAsync`)

## Caminhos de erro e exceção

> Tratado como cidadão de primeira classe, não como rodapé do caminho feliz. O diagrama de erro correspondente fica em `diagrama-fluxo.md` (Etapa 6).

| Cenário | Status code | Onde é detectado (arquivo/método) | Efeito colateral / compensação |
|---------|-------------|-------------------------------------|----------------------------------|
| `x-requestid` ausente ou `Guid.Empty` | 400 BadRequest ("RequestId is missing.") | `OrdersApi.cs:132-136` | N/A — nada foi criado |
| Requisição duplicada (mesmo `x-requestid`) | 200 OK (idempotente) | `IdentifiedCommandHandler.cs:41-45` (`RequestManager.ExistAsync`) | N/A — o pedido original é mantido; resposta de sucesso repetida |
| Falha de validação (FluentValidation: cidade/rua/cartão/itens etc.) | 500 Problem Details (default) — sem handler de exceção dedicado | `ValidatorBehavior.cs:28-34` → lança `OrderingDomainException` | Transação não iniciada ou rollback via `TransactionBehavior` |
| Regra de domínio violada (ex.: `units <= 0`, desconto > total) | 500 (default) | `OrderItem.cs:25-33`, `Order.cs:182` | Rollback da transação via `TransactionBehavior`/`CommitTransactionAsync` |
| Falha ao publicar evento no bus após commit | N/A — cliente já recebeu 200 OK | `OrderingIntegrationEventService.cs:21-32` | Evento marcado `PublishedFailed` na tabela `IntegrationEventLog`; **sem retry automático** nesta versão (reconciliação manual) |
| Falha de concorrência/banco durante `SaveEntitiesAsync` | 500 (default) | `OrderingContext.cs:47-62`, `TransactionBehavior.cs:57-62` | Rollback/`ResetRetry` da execução strategy; exceção relançada |

## Regras de negócio e invariantes

- [x] Regra 1: Pedido só pode ser criado com ao menos um item (`CreateOrderCommandValidator.ContainOrderItems`, `OrderItem` exige `units > 0`).
- [x] Regra 2: Dados de cartão são obrigatórios e válidos: número com 12–19 dígitos, CVV com 3, data de expiração futura (`CreateOrderCommandValidator.cs:11-15`).
- [x] Regra 3: Itens de produto repetido são mesclados no agregado: aplica o maior desconto e soma as unidades (`Order.AddOrderItem` — `Order.cs:71-91`).
- [x] Regra 4: O agregado protege as invariantes de pagamento e endereço (Value Objects `Address`, `PaymentMethod` validam na construção).
- [x] Transições de estado permitidas: `(new) → Submitted`; a partir daí `Submitted → AwaitingValidation → StockConfirmed → Paid → Shipped`, ou `AwaitingValidation → Cancelled` (stock rejeitado), `Paid/Shipped → Cancelled` — ver `OrderStatus` enum e métodos `Set*Status` (`Order.cs:99-168`).

## Efeitos colaterais

- [x] Eventos publicados: `OrderStartedIntegrationEvent` (enfileirado no handler ANTES de persistir o pedido — `CreateOrderCommandHandler.cs:32-33`) e `OrderStatusChangedToSubmittedIntegrationEvent` (criado no handler do `OrderStartedDomainEvent` — `ValidateOrAddBuyerAggregateWhenOrderStartedDomainEventHandler.cs:50`).
- [x] Atualização de outras entidades: `Buyer` criado/atualizado + `PaymentMethod` adicionado (handler do `OrderStartedDomainEvent`).
- [x] Chamadas a outros serviços: nenhuma síncrona durante o Create; as trocas são assíncronas via bus.
- [x] Operações assíncronas disparadas: `Basket.API` apaga o carrinho do usuário; `WebApp` notifica mudança de status (SSE).

## Aspectos não-funcionais

> Frequentemente o que diferencia "entendi o CRUD" de "entendi o sistema em produção". Marque `N/A` apenas se confirmado no código que não se aplica — se não houver evidência nenhuma, vá para "Perguntas em aberto".

- [x] **Idempotência:** Sim. O header `x-requestid` é persistido em `ClientRequest` (`RequestManager.CreateRequestForCommandAsync`). Requisição repetida com o mesmo GUID retorna sucesso sem reprocessar (`CreateOrderIdentifiedCommandHandler.CreateResultForDuplicateRequest` → `true`).
- [x] **Controle de concorrência:** Otimista. `Order`/`OrderItem` herdam de `Entity` com `Id` gerado por HiLo (`EntityConfigurations`); `PaymentMethod` usa `ObjectVersion` e verificação de identidade/estado no repositório (`BuyerRepository`). Não há versão de linha explícita no `Order`.
- [x] **Observabilidade:** `OrderingApiTrace` (LoggerMessage source-gen: `OrderStatusUpdated`, `PaymentMethodUpdated`, `BuyerAndPaymentValidatedOrUpdated`), scopes de log com `TransactionId` e `IdentifiedCommandId`, OpenTelemetry via `AddServiceDefaults()` + `RabbitMQTelemetry` no bus.
- [x] **Rate limiting / throttling:** Não encontrado no código. (Em aberto — provavelmente N/A nesta stack.)
- [x] **Timeout e retry em dependências externas:** PostgreSQL com `EnrichNpgsqlDbContext` (resiliência de execução strategy); publicação no bus sem política de retry explícita — falha marca evento como `PublishedFailed`.
- [x] **Consistência:** Forte dentro do banco (transação única via `TransactionBehavior` + outbox na mesma transação); eventual entre microsserviços (eventos entregues via RabbitMQ após commit).

## Fora do escopo

- [x] Validação de estoque (`Catalog.API` processa `OrderStatusChangedToAwaitingValidationIntegrationEvent` e responde com stock confirmado/rejeitado) — pertence ao fluxo pós-criação.
- [x] Processamento de pagamento (`PaymentProcessor`, `OrderPaymentSucceeded/FailedIntegrationEvent`).
- [x] Cancelamento (`PUT /api/orders/cancel`), envio (`PUT /api/orders/ship`), consulta (`GET /api/orders`, `GET /api/orders/{id}`) e rascunho (`POST /api/orders/draft`).
- [x] Grace period / confirmação automática de pedidos pendentes (`OrderProcessor`).

## Perguntas em aberto

> Tudo que não pôde ser confirmado com confiança no código entra aqui — nunca preenchido por inferência nas seções acima.

| Pergunta | Por que não foi possível confirmar | Impacto se ficar sem resposta |
|----------|--------------------------------------|-------------------------------|
| Qual o status HTTP exato de falha de validação/regra de domínio? | Não há `IExceptionHandler`/`UseExceptionHandler` na `Ordering.API`; `OrderingDomainException` lançada pelo `ValidatorBehavior` cai no middleware default (500 Problem Details). O teste funcional só cobre o caso de `x-requestid` vazio (400). | Médio |
| Existe reconciliação/retry para eventos `PublishedFailed` no outbox? | Não foi encontrado `BackgroundService`/worker de redispatch na `Ordering.API` nesta versão. | Alto — eventos podem ser perdidos se o bus ficar indisponível no momento do publish |
| Por que `OrderStartedIntegrationEvent` é enfileirado antes de persistir o pedido (potencial de "evento órfão")? | O handler enfileira o evento na linha 32, antes de `SaveEntitiesAsync`/validação completa; se a persistência falhar, o evento pode já ter sido gravado no outbox. | Baixo (o outbox usa a mesma transação e só publica após commit) |
| O vínculo `Buyer`/`PaymentMethod` depende de o `Id` do pedido (HiLo) já estar atribuído no handler de domínio — é confiável? | Comentário `REVIEW` em `ValidateOrAddBuyerAggregateWhenOrderStartedDomainEventHandler.cs:31-32` admite que funciona "por coincidência" da ordem de geração do HiLo. | Médio |

## Critérios de aceite do estudo

- [x] Encontrei o arquivo que registra a rota/método do endpoint
- [x] Rastreei o handler/command que executa o caso de uso (tabela completa em `diagrama-fluxo.md`)
- [x] Identifiquei o repositório e a persistência
- [x] Mapeei eventos de domínio e integração
- [x] Confirmei status codes de sucesso e erro
- [x] Mapeei os aspectos não-funcionais ou registrei sua ausência como pergunta em aberto
- [x] Registrei decisões em `lista-adr.md`
- [x] Criei o `diagrama-fluxo.md` da Order
- [x] Preenchi a hipótese de `implement-reduz.md`
- [x] Adicionei a linha correspondente em `../../_INDEX.md`

## Observações

- [x] O endpoint mascara o número do cartão ANTES de criar o `CreateOrderCommand` e evita logar o request inteiro (comentário explícito em `OrdersApi.cs:124-130`).
- [x] A cadeia de eventos de integração é desacoplada do request HTTP: o cliente recebe `200 OK` assim que a transação é commitada; a publicação no bus acontece depois, no `TransactionBehavior`.
- [x] Débito técnico: a anotação `[DataContract]`/`[DataMember]` em `CreateOrderCommand` não é usada pelo serializador JSON padrão desta versão.
- [x] `CreateOrderDraftAsync` (`POST /api/orders/draft`) é um endpoint separado que NÃO persiste — apenas monta um `OrderDraftDTO` a partir do carrinho (fora do escopo deste estudo).
