# Implementação Reduzida — Order (Create)

> **Instruções para a LLM**
>
> - [x] Substitua os placeholders `{...}` pelos dados reais.
> - [x] A implementação reduzida deve reproduzir o aprendizado com o menor número possível de componentes.
> - [x] Ela **não é**: uma cópia do eShop, um template genérico de produção, uma arquitetura completa.
> - [x] Ela **é**: um experimento executável que comprova que você entendeu o mecanismo.
> - [x] Este template é agnóstico de stack. Escolha a variante de estrutura mínima (Etapa 2) que corresponde à stack do projeto estudado; não force uma estrutura de outra stack.

## Etapa 1 — Hipótese de aprendizado

- [x] Formule a hipótese que direcionará o código.

**Exemplo (backend):**
- [x] "Consigo confirmar um pedido, persistir seu estado, publicar um evento e entregar a mudança ao frontend via SSE sem acoplar o domínio ao transporte HTTP."

**Exemplo (frontend):**
- [x] "Consigo disparar uma ação de confirmação, refletir o estado de loading/erro na UI e reagir a uma atualização assíncrona do servidor sem acoplar a lógica de estado ao widget."

**Sua hipótese:**
- [x] "Consigo criar um pedido, garantir idempotência por `request-id`, persistir pedido + evento de integração na mesma transação (outbox) e publicar o evento após o commit — sem acoplar o agregado `Order` ao handler, ao repositório ou ao bus."

## Etapa 2 — Mínimo necessário

- [x] Defina a estrutura mínima com base no fluxo mapeado no `diagrama-fluxo.md`. Escolha a variante correspondente à stack estudada e apague as demais.

### Variante — Backend orientado a domínio (.NET/DDD e equivalentes)

```text
OrderCreateStudy/
├── Domain/
│   ├── Order.cs
│   ├── OrderItem.cs
│   ├── Address.cs
│   └── OrderStatus.cs
│
├── Application/
│   ├── CreateOrderCommand.cs
│   ├── CreateOrderCommandHandler.cs
│   ├── IdentifiedCommand.cs
│   └── IdentifiedCommandHandler.cs
│
├── Infrastructure/
│   ├── InMemoryOrderRepository.cs
│   ├── InMemoryRequestManager.cs
│   └── InMemoryEventBroker.cs
│
├── Api/
│   └── Program.cs
│
└── Tests/
    └── OrderCreateTests.cs
```

- [x] Ajuste nomes/estrutura ao caso real; o objetivo é reproduzir o mecanismo, não copiar o projeto.

### Variante — Frontend com gerenciamento de estado (Flutter e equivalentes)

- [x] N/A — o mecanismo estudado é de backend (agregado + outbox). Não força estrutura de outra stack.

### Variante — Livre (outra stack)

- [x] N/A — variante .NET/DDD é suficiente para reproduzir o mecanismo.

## Etapa 3 — O que NÃO adicionar inicialmente

- [x] Framework de mensageria real (Kafka, RabbitMQ, ...) — usar um `InMemoryEventBroker` com lista de handlers inscritos.
- [x] Framework de mapeamento automático (AutoMapper e equivalentes)
- [x] Banco de dados real (PostgreSQL, DynamoDB, ...) — usar `InMemoryOrderRepository`/`InMemoryRequestManager`.
- [x] Containerização (Docker, Kubernetes)
- [x] Observabilidade real (OpenTelemetry e equivalentes)
- [x] Bibliotecas de injeção de dependência de terceiros, se o mecanismo nativo da stack já resolve
- [x] {Outra dependência pesada específica da stack} — MediatR, EF Core e FluentValidation podem ser substituídos por chamadas diretas para isolar o mecanismo.

- [x] Primeiro valide o fluxo completo; só depois decida por infraestrutura real.

## Etapa 4 — Roteiro de validação do fluxo

- [x] Criar Order no estado `Submitted` (construtor valida endereço/cartão)
- [x] Ponto de entrada `POST /orders (x-requestid + CreateOrderCommand)` recebe a solicitação
- [x] Camada de orquestração invoca `order.AddOrderItem(...)` (agregado aplica invariantes)
- [x] Persistência em memória guarda a mudança (repositório + unidade de trabalho simulada)
- [x] Notificação/evento em memória é emitido (OrderStartedIntegrationEvent no broker)
- [x] Consumidor (outro módulo, listener, ou widget) reage e produz `limpeza do carrinho do usuário`
- [x] Teste automatizado cobre o cenário de sucesso (pedido criado, evento emitido uma vez)
- [x] Teste automatizado cobre o cenário de regra violada / erro (item com `units <= 0` lança exceção de domínio)

## Etapa 5 — Guardrails de execução

- [x] **Orçamento de tempo/escopo:** a implementação reduzida deve ficar pronta em uma única sessão de trabalho focada; se estiver crescendo além do "mínimo necessário" da Etapa 2, é sinal de que o experimento perdeu o foco — pare e reavalie a hipótese.
- [x] Se, durante a implementação, surgir a necessidade de algo listado na Etapa 3, registre isso como uma decisão pendente em vez de simplesmente adicionar — a necessidade real é, em si, um aprendizado.

## Checklist final

- [x] Hipótese escrita e testável
- [x] Variante de estrutura escolhida corresponde à stack estudada
- [x] Fluxo completo reproduzido (entrada → persistência → evento/notificação → consumidor)
- [x] Sem dependências desnecessárias
- [x] Domínio/lógica desacoplado(a) do transporte ou da UI
- [x] Testes automatizados validam o mecanismo
