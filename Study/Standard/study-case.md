# Caso de Estudo — {Entidade} ({Endpoint})

> **Instruções para a LLM**
>
> - [ ] Substitua todos os placeholders `{...}` pelos dados reais do projeto analisado.
> - [ ] Preencha os checklists apenas com informações encontradas no código; não invente.
> - [ ] Marque itens não aplicáveis como `N/A`, nunca deixe em branco.
> - [ ] Se não conseguir confirmar algo com certeza no código, **não preencha por inferência** — registre em "Perguntas em aberto" (Etapa 8).
> - [ ] Este arquivo é o sumário: deve permitir leitura isolada, sem abrir o código.
> - [ ] A tabela de rastreio de componentes vive em `diagrama-fluxo.md` — não a duplique aqui, apenas referencie.
> - [ ] Ao concluir o checklist final, adicione uma linha em `../../_INDEX.md` e, se aplicável, uma entrada em `../../licoes-aprendidas.md`.

## Metadados do estudo

- [ ] **Projeto:** {Projeto}
- [ ] **Repositório/commit analisado:** {URL ou nome do repo} @ {hash do commit}
- [ ] **Data do estudo:** {AAAA-MM-DD}
- [ ] **Stack:** {.NET / Flutter / outro}
- [ ] **Entidade:** {Entidade}
- [ ] **Endpoint:** {Endpoint}
- [ ] **Método HTTP:** {Método HTTP}
- [ ] **Rota:** {Rota}
- [ ] **Versão da API:** {Versão}

## Objetivo

- [ ] Descrever o objetivo do estudo: {O que o endpoint faz}
- [ ] Qual problema de negócio ele resolve?
- [ ] Quais outros componentes dependem dele?

## Entrada

- [ ] **Body:** {Estrutura do corpo da requisição}
- [ ] **Query params:** {Parâmetros de consulta}
- [ ] **Path params:** {Parâmetros de rota}
- [ ] **Headers obrigatórios:** {Headers}
- [ ] **Autenticação/Autorização:** {Tipo de auth e claims exigidas}

### Pré-condições

- [ ] Estado anterior exigido da {Entidade}: {Estado A}
- [ ] Dependências externas esperadas: {Serviços/recursos}

## Resultado esperado (caminho feliz)

- [ ] **Status code de sucesso:** {200/201/204...}
- [ ] **Body de resposta:** {Estrutura de retorno}
- [ ] {Entidade} muda de `{Estado A}` para `{Estado B}`
- [ ] Alteração é persistida em {Banco/Tabela}
- [ ] Evento `{NomeDoEvento}` é publicado em {Bus/Mensageria}
- [ ] Interessados notificados: {Lista de consumidores}

## Caminhos de erro e exceção

> Tratado como cidadão de primeira classe, não como rodapé do caminho feliz. O diagrama de erro correspondente fica em `diagrama-fluxo.md` (Etapa 6).

| Cenário | Status code | Onde é detectado (arquivo/método) | Efeito colateral / compensação |
|---------|-------------|-------------------------------------|----------------------------------|
| {ex.: Entidade não encontrada} | {404} | {Arquivo:linha} | {N/A ou rollback/compensação} |
| {ex.: Regra de negócio violada} | {409/422} | {Arquivo:linha} | {N/A ou notificação} |
| {ex.: Falha em serviço externo} | {502/503} | {Arquivo:linha} | {Retry? Circuit breaker? N/A} |
| {ex.: Falha ao publicar evento após persistir} | {N/A — já retornou sucesso ao cliente} | {Arquivo:linha} | {Outbox? Reconciliação manual? Perda aceita?} |

## Regras de negócio e invariantes

- [ ] Regra 1: {Regra}
- [ ] Regra 2: {Regra}
- [ ] Transições de estado permitidas: {Máquina de estados}

## Efeitos colaterais

- [ ] Eventos publicados: {Eventos}
- [ ] Atualização de outras entidades: {Entidades}
- [ ] Chamadas a outros serviços: {Serviços}
- [ ] Operações assíncronas disparadas: {Operações}

## Aspectos não-funcionais

> Frequentemente o que diferencia "entendi o CRUD" de "entendi o sistema em produção". Marque `N/A` apenas se confirmado no código que não se aplica — se não houver evidência nenhuma, vá para "Perguntas em aberto".

- [ ] **Idempotência:** {Como a requisição repetida é tratada — chave de idempotência, upsert natural, ou nenhuma proteção?}
- [ ] **Controle de concorrência:** {Otimista (version/ETag), pessimista (lock), ou nenhum?}
- [ ] **Observabilidade:** {Tracing/correlação distribuída, métricas, logs estruturados — o que existe de fato}
- [ ] **Rate limiting / throttling:** {Existe? Onde é aplicado?}
- [ ] **Timeout e retry em dependências externas:** {Política observada}
- [ ] **Consistência:** {Forte ou eventual? Onde a janela de inconsistência aparece?}

## Fora do escopo

- [ ] {Item que não é responsabilidade deste endpoint}
- [ ] {Item que não é responsabilidade deste endpoint}

## Perguntas em aberto

> Tudo que não pôde ser confirmado com confiança no código entra aqui — nunca preenchido por inferência nas seções acima.

| Pergunta | Por que não foi possível confirmar | Impacto se ficar sem resposta |
|----------|--------------------------------------|-------------------------------|
| {ex.: O que acontece se o broker estiver indisponível no momento da publicação?} | {ex.: Não há teste nem log cobrindo esse caminho} | {ex.: Baixo/Médio/Alto} |

## Critérios de aceite do estudo

- [ ] Encontrei o arquivo que registra a rota/método do endpoint
- [ ] Rastreei o handler/command que executa o caso de uso (tabela completa em `diagrama-fluxo.md`)
- [ ] Identifiquei o repositório e a persistência
- [ ] Mapeei eventos de domínio e integração
- [ ] Confirmei status codes de sucesso e erro
- [ ] Mapeei os aspectos não-funcionais ou registrei sua ausência como pergunta em aberto
- [ ] Registrei decisões em `lista-adr.md`
- [ ] Criei o `diagrama-fluxo.md` da {Entidade}
- [ ] Preenchi a hipótese de `implement-reduz.md`
- [ ] Adicionei a linha correspondente em `../../_INDEX.md`

## Observações

- [ ] {Anotações livres, caveats, débitos técnicos}