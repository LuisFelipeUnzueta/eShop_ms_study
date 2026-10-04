# Implementação Reduzida — {Entidade} ({Endpoint})

> **Instruções para a LLM**
>
> - [ ] Substitua os placeholders `{...}` pelos dados reais.
> - [ ] A implementação reduzida deve reproduzir o aprendizado com o menor número possível de componentes.
> - [ ] Ela **não é**: uma cópia do {Projeto}, um template genérico de produção, uma arquitetura completa.
> - [ ] Ela **é**: um experimento executável que comprova que você entendeu o mecanismo.
> - [ ] Este template é agnóstico de stack. Escolha a variante de estrutura mínima (Etapa 2) que corresponde à stack do projeto estudado; não force uma estrutura de outra stack.

## Etapa 1 — Hipótese de aprendizado

- [ ] Formule a hipótese que direcionará o código.

**Exemplo (backend):**
- [ ] "Consigo confirmar um pedido, persistir seu estado, publicar um evento e entregar a mudança ao frontend via SSE sem acoplar o domínio ao transporte HTTP."

**Exemplo (frontend):**
- [ ] "Consigo disparar uma ação de confirmação, refletir o estado de loading/erro na UI e reagir a uma atualização assíncrona do servidor sem acoplar a lógica de estado ao widget."

**Sua hipótese:**
- [ ] {Hipótese de aprendizado para {Entidade}/{Endpoint}}

## Etapa 2 — Mínimo necessário

- [ ] Defina a estrutura mínima com base no fluxo mapeado no `diagrama-fluxo.md`. Escolha a variante correspondente à stack estudada e apague as demais.

### Variante — Backend orientado a domínio (.NET/DDD e equivalentes)

```text
{Entidade}Study/
├── Domain/
│   ├── {Entidade}.cs
│   ├── {Entidade}Item.cs
│   └── {Estado}.cs
│
├── Application/
│   ├── {ConfirmarEntidade}Command.cs
│   └── {ConfirmarEntidade}Handler.cs
│
├── Infrastructure/
│   ├── InMemory{Entidade}Repository.cs
│   └── {Entidade}EventBroker.cs
│
├── Api/
│   └── Program.cs
│
└── Tests/
    └── {Entidade}Tests.cs
```

### Variante — Frontend com gerenciamento de estado (Flutter e equivalentes)

```text
{Entidade}Study/
├── domain/
│   └── {entidade}_model.dart
│
├── state/
│   └── {entidade}_notifier.dart
│
├── data/
│   └── in_memory_{entidade}_repository.dart
│
├── ui/
│   └── {entidade}_screen.dart
│
└── test/
    └── {entidade}_notifier_test.dart
```

### Variante — Livre (outra stack)

```text
{Entidade}Study/
├── {camada de domínio/modelo}/
├── {camada de orquestração/caso de uso}/
├── {camada de persistência/infra em memória}/
├── {camada de entrada — API, UI, CLI}/
└── {testes}/
```

- [ ] Ajuste nomes/estrutura ao caso real; o objetivo é reproduzir o mecanismo, não copiar o projeto.

## Etapa 3 — O que NÃO adicionar inicialmente

- [ ] Framework de mensageria real (Kafka, RabbitMQ, ...)
- [ ] Framework de mapeamento automático (AutoMapper e equivalentes)
- [ ] Banco de dados real (PostgreSQL, DynamoDB, ...)
- [ ] Containerização (Docker, Kubernetes)
- [ ] Observabilidade real (OpenTelemetry e equivalentes)
- [ ] Bibliotecas de injeção de dependência de terceiros, se o mecanismo nativo da stack já resolve
- [ ] {Outra dependência pesada específica da stack}

- [ ] Primeiro valide o fluxo completo; só depois decida por infraestrutura real.

## Etapa 4 — Roteiro de validação do fluxo

- [ ] Criar {Entidade} no estado `{Estado inicial}`
- [ ] Ponto de entrada `{Rota/Ação/Evento de UI}` recebe a solicitação
- [ ] Camada de orquestração invoca `{MetodoDoAgregado ou equivalente}()`
- [ ] Persistência em memória guarda a mudança
- [ ] Notificação/evento em memória é emitido
- [ ] Consumidor (outro módulo, listener, ou widget) reage e produz `{Efeito esperado}`
- [ ] Teste automatizado cobre o cenário de sucesso
- [ ] Teste automatizado cobre o cenário de regra violada / erro

## Etapa 5 — Guardrails de execução

- [ ] **Orçamento de tempo/escopo:** a implementação reduzida deve ficar pronta em uma única sessão de trabalho focada; se estiver crescendo além do "mínimo necessário" da Etapa 2, é sinal de que o experimento perdeu o foco — pare e reavalie a hipótese.
- [ ] Se, durante a implementação, surgir a necessidade de algo listado na Etapa 3, registre isso como uma decisão pendente em vez de simplesmente adicionar — a necessidade real é, em si, um aprendizado.

## Checklist final

- [ ] Hipótese escrita e testável
- [ ] Variante de estrutura escolhida corresponde à stack estudada
- [ ] Fluxo completo reproduzido (entrada → persistência → evento/notificação → consumidor)
- [ ] Sem dependências desnecessárias
- [ ] Domínio/lógica desacoplado(a) do transporte ou da UI
- [ ] Testes automatizados validam o mecanismo