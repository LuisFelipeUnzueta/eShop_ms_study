# Lições Aprendidas — Padrões e Antipadrões entre Projetos

> **Instruções para a LLM**
>
> - [ ] Este arquivo é atualizado ao final de cada estudo (`study-case.md` completo), não durante.
> - [ ] Só adicione uma entrada aqui se o padrão/decisão já apareceu em **mais de um projeto** ou se for suficientemente notável para servir de referência futura (ex.: uma decisão que deu errado e vale evitar).
> - [ ] Não duplique o conteúdo do ADR original — aponte para ele (`{projeto}/{entidade}-{endpoint}/lista-adr.md#adr-{N}`) e registre aqui apenas a generalização.
> - [ ] Separe padrões (algo que funcionou bem e vale repetir) de antipadrões (algo que gerou problema e vale evitar).

## Padrões (repetir)

### Outbox Pattern para publicação de eventos de integração

- **Onde apareceu:** eShop `Order/Create/lista-adr.md#adr-002`
- **Problema que resolve:** Garantir que a persistência do estado e a intenção de publicar um evento sejam atômicas, evitando eventos fantasmas (publicado sem persistir) e eventos perdidos (persistir mas falhar ao publicar).
- **Quando aplicar:** Sempre que houver efeitos colaterais em outros serviços disparados por uma operação local persistente, e o bus não oferecer garantia transacional com o banco.
- **Quando NÃO aplicar:** Se o bus participar da transação com o banco, ou se a perda eventual de eventos for aceitável (ex.: telemetria de baixa criticidade).
- **Trade-off aceito:** Camada extra de tabela de log + processamento de publicação pós-commit; eventos só são publicados se a transação commitar.

### Idempotência via request-id persistido

- **Onde apareceu:** eShop `Order/Create/lista-adr.md#adr-001`
- **Problema que resolve:** Retries de rede e duplo clique podem repetir uma operação de escrita; sem proteção, o mesmo efeito colateral acontece duas vezes.
- **Quando aplicar:** Escritas com efeitos colaterais (eventos, mensageria) que não tenham chave natural de negócio única.
- **Quando NÃO aplicar:** Quando a própria operação é naturalmente idempotente (ex.: `UPDATE` com valor fixo, `DELETE`).
- **Trade-off aceito:** Cliente precisa fornecer o GUID; tabela de requisições precisa de política de retenção.

### Domain events dispatchados antes do SaveChanges (mesma transação)

- **Onde apareceu:** eShop `Order/Create/lista-adr.md#adr-004`
- **Problema que resolve:** Handlers de domínio que precisam alterar outras entidades da mesma agregação/borda (ex.: criar Buyer ao iniciar pedido) devem commitar junto com a entidade principal, sem compensação.
- **Quando aplicar:** Quando os side effects de domínio são locais (mesmo DbContext) e rápidos.
- **Quando NÃO aplicar:** Quando o handler de domínio fizer I/O externo lento/não-transacional — aí prefira eventos de integração após o commit.
- **Trade-off aceito:** Reentrância no SaveChanges e dependência da ordem de geração de IDs; frágil se a estratégia de geração de chaves mudar.

## Antipadrões (evitar)

### Publicar evento no bus sem registro em tabela (fire-and-forget pós-commit)

- **Onde apareceu:** Não foi observado no eShop (aqui foi usado outbox), mas é o desvio natural a evitar.
- **O que foi feito:** (referência) Publicar direto no bus após `SaveChanges`, sem armazenar a intenção.
- **Por que gerou problema:** Se o broker estiver indisponível no momento exato do publish, o evento é perdido silenciosamente e o estado dos serviços consumidores fica desatualizado sem rastro.
- **Alternativa recomendada:** Outbox (tabela de eventos + publish pós-commit com marcação de estado) ou retry com dead-letter.

### Cadeia de eventos dependendo da ordem de geração de IDs (HiLo) sem garantia formal

- **Onde apareceu:** eShop `Order/Create/study-case.md` (Pergunta em aberto) e `ValidateOrAddBuyerAggregateWhenOrderStartedDomainEventHandler.cs:31-32` (`REVIEW`)
- **O que foi feito:** O handler do `OrderStartedDomainEvent` cria um `OrderStatusChangedToSubmittedIntegrationEvent` usando o `Id` do pedido, que só existe depois do HiLo — funcionando "por coincidência" da ordem de execução.
- **Por que gerou problema:** Qualquer mudança no provider de chaves ou no pipeline de persistência pode gerar evento com `OrderId = 0` ou quebrar o vínculo Buyer/Pedido.
- **Alternativa recomendada:** Emitir o evento de integração somente após o commit (quando o ID está garantido), ou propagar o ID via retorno do repositório/transação.

## Glossário de decisões recorrentes

> Perguntas que se repetem entre estudos e como cada projeto respondeu — útil para comparação rápida sem reabrir os ADRs.

| Decisão | eShop (Order/Create) |
|---------|----------------------|
| REST ou mensageria? | REST para criar; mensageria para efeitos colaterais |
| SSE, polling ou WebSocket? | SSE (notificação de status na WebApp) |
| Evento de domínio ou integração publicado diretamente? | Domínio no processo; integração via outbox |
| Outbox ou publish direto? | Outbox (IntegrationEventLog na mesma transação) |
| EF Core, Dapper ou outro? | EF Core (escrita) + Dapper em `OrderQueries` (leitura) |
| Estratégia de idempotência | Request-id persistido em `ClientRequest` |
| Estratégia de controle de concorrência | Otimista (`PaymentMethod.ObjectVersion`) |
