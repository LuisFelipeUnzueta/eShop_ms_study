# Lista de Decisões (ADR) — {Entidade}

> **Instruções para a LLM**
>
> - [ ] Substitua os placeholders `{...}` pelos dados reais.
> - [ ] Registre decisões reais do projeto: problema, opções e motivo da escolha.
> - [ ] **Evite** listar tecnologias soltas ("Usa Kafka", "Usa DDD", "Usa repository").
> - [ ] **Prefira** explicar o porquê: "O Kafka foi usado para desacoplar a confirmação do pedido do processo de notificação, evitando que uma indisponibilidade do serviço de notificações impeça a confirmação."
> - [ ] Numere os ADRs sequencialmente: ADR-001, ADR-002, ...
> - [ ] Se, ao registrar uma decisão, você perceber que ela já apareceu em outro projeto estudado, adicione uma nota em "Decisões recorrentes" (fim deste arquivo) apontando para `../../licoes-aprendidas.md`.

## Etapa 1 — Identificar decisões durante a leitura

- [ ] Sempre que algo poderia ter sido implementado de outra forma, registre uma decisão.

- [ ] Método no agregado ou no handler?
- [ ] REST ou mensageria?
- [ ] Evento de domínio ou integração?
- [ ] Publicar diretamente ou usar Outbox?
- [ ] SSE ou polling?
- [ ] EF Core ou Dapper?
- [ ] Monólito ou microsserviço?
- [ ] Estado calculado ou persistido?
- [ ] Estratégia de idempotência escolhida?
- [ ] Estratégia de controle de concorrência escolhida?
- [ ] {Outra pergunta de decisão específica do projeto}

## Etapa 2 — Perguntas fixas (responda para cada decisão)

- [ ] 1. Qual problema precisava ser resolvido?
- [ ] 2. Qual decisão foi tomada?
- [ ] 3. Quais alternativas existiam?
- [ ] 4. Quais benefícios foram obtidos?
- [ ] 5. Quais custos ou limitações surgiram?
- [ ] 6. Em quais condições essa decisão deixaria de fazer sentido?

## Etapa 3 — Modelo ADR reduzido

- [ ] Duplique este modelo para cada decisão registrada.

## ADR-{N} — {Título da decisão}

### Contexto

- [ ] {Problema de negócio/técnico que motivou a decisão}

### Decisão

- [ ] {O que foi decidido e onde foi implementado (arquivo/método)}

### Motivo

- [ ] {Justificativa de por que esta opção venceu}

### Alternativas consideradas

- [ ] {Alternativa 1}
- [ ] {Alternativa 2}
- [ ] {Alternativa 3}

### Consequências positivas

- [ ] {Benefício 1}
- [ ] {Benefício 2}

### Consequências negativas

- [ ] {Custo/limitação 1}
- [ ] {Custo/limitação 2}

### Quando reconsiderar

- [ ] {Condição que invalidaria a decisão}

---

## Exemplos preenchidos (referência)

### ADR-001 — Regra de confirmação dentro do agregado

#### Contexto

- [ ] Um pedido só pode ser confirmado quando possui itens e está em rascunho.

#### Decisão

- [ ] A regra será implementada em `Order.Confirm()`, e não diretamente no endpoint ou no handler.

#### Motivo

- [ ] O agregado é responsável por proteger suas invariantes, independentemente do canal que iniciou a operação.

#### Alternativas consideradas

- [ ] Validar no endpoint.
- [ ] Validar no handler.
- [ ] Usar um serviço de domínio.

#### Consequências positivas

- [ ] Regra centralizada.
- [ ] Menor risco de bypass.
- [ ] Testes unitários simples.

#### Consequências negativas

- [ ] O agregado passa a concentrar mais comportamento.
- [ ] A persistência precisa reconstruir corretamente seu estado.

#### Quando reconsiderar

- [ ] Caso a confirmação dependa fortemente de múltiplos agregados ou de serviços externos.

---

### ADR-002 — SSE para atualização de status

#### Contexto

- [ ] O frontend precisa ser informado quando o status do pedido mudar; as atualizações são produzidas apenas pelo servidor.

#### Decisão

- [ ] Usar SSE para entregar notificações de mudança de status.

#### Alternativas consideradas

- [ ] Polling REST.
- [ ] WebSocket.
- [ ] SignalR.
- [ ] Push notification.

#### Motivo

- [ ] A comunicação é unidirecional, do servidor para o cliente; SSE possui menor complexidade operacional do que WebSocket.

#### Consequências positivas

- [ ] Reconexão automática no navegador.
- [ ] Uso do protocolo HTTP.
- [ ] Implementação simples.
- [ ] Menor tráfego que polling frequente.

#### Consequências negativas

- [ ] Conexões persistentes.
- [ ] Necessidade de heartbeat e timeout.
- [ ] `EventSource` não permite header Authorization personalizado.
- [ ] Infraestrutura precisa evitar buffering.

#### Quando reconsiderar

- [ ] Se o cliente também precisar enviar mensagens frequentes durante a mesma sessão.

---

## Decisões recorrentes (cross-project)

> Preencha apenas se, durante este estudo, você notou que uma decisão já havia aparecido em projeto anterior.

| ADR deste estudo | Já visto em | Nota |
|-------------------|-------------|------|
| {ADR-00N} | {Projeto anterior, referência} | {Ver `../../licoes-aprendidas.md#{ancora}`} |