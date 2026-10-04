# Lições Aprendidas — Padrões e Antipadrões entre Projetos

> **Instruções para a LLM**
>
> - [ ] Este arquivo é atualizado ao final de cada estudo (`study-case.md` completo), não durante.
> - [ ] Só adicione uma entrada aqui se o padrão/decisão já apareceu em **mais de um projeto** ou se for suficientemente notável para servir de referência futura (ex.: uma decisão que deu errado e vale evitar).
> - [ ] Não duplique o conteúdo do ADR original — aponte para ele (`{projeto}/{entidade}-{endpoint}/lista-adr.md#adr-{N}`) e registre aqui apenas a generalização.
> - [ ] Separe padrões (algo que funcionou bem e vale repetir) de antipadrões (algo que gerou problema e vale evitar).

## Padrões (repetir)

### {Nome do padrão, ex.: "Outbox Pattern para publicação de eventos de integração"}

- **Onde apareceu:** {Projeto A `lista-adr.md#adr-003`, Projeto B `lista-adr.md#adr-001`}
- **Problema que resolve:** {Descrição geral do problema, sem detalhes de um projeto específico}
- **Quando aplicar:** {Condições em que esse padrão faz sentido}
- **Quando NÃO aplicar:** {Condições em que é overkill ou não se encaixa}
- **Trade-off aceito:** {O que se abre mão para ganhar o benefício}

## Antipadrões (evitar)

### {Nome do antipadrão}

- **Onde apareceu:** {Projeto, referência ao estudo}
- **O que foi feito:** {Descrição neutra da decisão}
- **Por que gerou problema:** {Consequência observada}
- **Alternativa recomendada:** {O que fazer da próxima vez}

## Glossário de decisões recorrentes

> Perguntas que se repetem entre estudos e como cada projeto respondeu — útil para comparação rápida sem reabrir os ADRs.

| Decisão | Projeto A | Projeto B | Projeto C |
|---------|-----------|-----------|-----------|
| REST ou mensageria? | | | |
| SSE, polling ou WebSocket? | | | |
| Evento de domínio ou integração publicado diretamente? | | | |
| Outbox ou publish direto? | | | |
| EF Core, Dapper ou outro? | | | |
| Estratégia de idempotência | | | |
| Estratégia de controle de concorrência | | | |