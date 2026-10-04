# Índice de Estudos de Endpoints/Features

> **Instruções para a LLM**
>
> - [ ] Este arquivo é o catálogo mestre de todos os estudos realizados, em qualquer projeto.
> - [ ] Toda vez que um novo estudo for concluído (checklist final de `study-case.md` 100% marcado), adicione uma linha aqui.
> - [ ] Não reabra os arquivos dos estudos antigos para preencher esta tabela — use apenas os metadados do cabeçalho de cada `study-case.md`.
> - [ ] Mantenha ordenação por data decrescente (mais recente primeiro).

## Catálogo

| Data | Projeto | Stack | Entidade | Endpoint | Pasta do estudo | Status |
|------|---------|-------|----------|----------|------------------|--------|
| 2026-08-04 | eShop (eShop_ms_study) | .NET 10 (Minimal API + MediatR + EF/Npgsql + RabbitMQ) | Order | POST /api/orders | `Study/Order/Create/` | Completo |

## Padrões observados até agora

> Preenchido incrementalmente conforme os estudos avançam. Cada linha aponta para o estudo onde o padrão foi identificado com detalhe — o detalhe fica em `licoes-aprendidas.md`, aqui é só o radar.

| Padrão/Decisão recorrente | Projetos onde aparece | Referência |
|---------------------------|------------------------|------------|
| Outbox Pattern para publicação de eventos de integração | eShop (Order/Create) | `licoes-aprendidas.md#outbox-pattern` |
| Idempotência via request-id persistido (ClientRequest) | eShop (Order/Create) | `licoes-aprendidas.md#idempotencia-request-id` |
| Domain events dispatchados antes do SaveChanges (mesma transação) | eShop (Order/Create) | `licoes-aprendidas.md#domain-events-no-savechanges` |

## Perguntas em aberto (cross-project)

> Bloqueios ou dúvidas que não foram resolvidos durante um estudo específico e que valem revisitar.

| Origem (estudo) | Pergunta | Status |
|------------------|----------|--------|
| Order/Create | Não há retry/reconciliação automática para eventos `PublishedFailed` no outbox | Em aberto |
| Order/Create | Status HTTP exato (400 vs 500) para falhas de validação/regra de domínio — sem `IExceptionHandler` dedicado | Em aberto |

## Convenção de pastas

```text
/studies/
├── _INDEX.md                  ← este arquivo
├── licoes-aprendidas.md       ← padrões/antipadrões consolidados
└── {projeto}/
    └── {entidade}-{endpoint}/
        ├── study-case.md
        ├── diagrama-fluxo.md
        ├── lista-adr.md
        └── implement-reduz.md
```
