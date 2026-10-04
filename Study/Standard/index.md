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
| {AAAA-MM-DD} | {Projeto} | {.NET / Flutter / ...} | {Entidade} | {Método Rota} | `/studies/{projeto}/{entidade}-{endpoint}/` | {Completo / Parcial / Bloqueado} |

## Padrões observados até agora

> Preenchido incrementalmente conforme os estudos avançam. Cada linha aponta para o estudo onde o padrão foi identificado com detalhe — o detalhe fica em `licoes-aprendidas.md`, aqui é só o radar.

| Padrão/Decisão recorrente | Projetos onde aparece | Referência |
|---------------------------|------------------------|------------|
| {ex.: Outbox Pattern para publicação de eventos} | {Projeto A, Projeto B} | `licoes-aprendidas.md#outbox-pattern` |

## Perguntas em aberto (cross-project)

> Bloqueios ou dúvidas que não foram resolvidos durante um estudo específico e que valem revisitar.

| Origem (estudo) | Pergunta | Status |
|------------------|----------|--------|
| {projeto/entidade-endpoint} | {Pergunta não respondida com confiança} | {Em aberto / Resolvido em {data}} |

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