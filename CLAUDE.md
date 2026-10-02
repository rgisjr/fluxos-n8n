# Fluxos n8n — regras do projeto

Este projeto existe para o Claude criar e manter fluxos no n8n sob demanda.
Instância: https://n8n.grupotoda.com.br (MCP `claude_ai_n8n` / `.mcp.json`).
Idioma: português em tudo (nomes de fluxo, nós, READMEs, commits).

## Estrutura

```
docs/specs/          designs aprovados
docs/plans/          planos de implementação
docs/referencias/    imagens e materiais de apoio
fluxos/<slug>/       um diretório por fluxo (slug em kebab-case)
  README.md          explicação em português
  workflow.json      cópia exata do fluxo no n8n (get_workflow_details)
  prompts/           prompts de IA, quando houver
```

README de cada fluxo, seções obrigatórias: Objetivo · Link e ID no n8n · Gatilho ·
Passo a passo · Credenciais usadas (nome e ID, nunca o segredo) · Configuração manual
necessária · Como testar · Limitações conhecidas · Histórico de decisões.

## Fonte da verdade

O fluxo vive no n8n; o repo guarda a cópia (`workflow.json`) e o README.

- **Antes de editar** um fluxo existente: `get_workflow_details` e comparar `versionId`
  com o do `workflow.json` local. Se diferir, alguém editou na interface: salvar a
  versão do n8n no repo e commitar **antes** de alterar.
- **Depois de qualquer mudança**: puxar o JSON atualizado, atualizar o README, commitar.

## Como construir (ordem obrigatória no MCP)

1. `get_workflow_sdk_reference` → 2. `get_workflow_best_practices` (por técnica) →
3. `search_nodes` → 4. `get_node_types` (com discriminadores) →
5. `explore_node_resources` para listas/IDs reais → 6. `validate_workflow` →
7. `create_workflow_from_code` (novo) ou `update_workflow` (edição; altera o rascunho) →
8. testar (`prepare_workflow_pin_data` / `test_workflow` / `execute_workflow`) →
9. `publish_workflow`.

Gatilhos executáveis pelo MCP: só Schedule, Webhook, Form e Chat. Para testar outros
gatilhos, fixar dados (pin data) ou usar um fluxo temporário "TESTE …" com Webhook,
arquivado ao final.

## Autonomia

Fluxo novo ou alterado: criar → validar → testar → **publicar se os testes passarem** →
salvar cópia e README → commit, sem pedir aprovação. Se um teste falhar sem correção
óbvia, deixar inativo e reportar.

## Regras fixas

- Todo fluxo publicado tem `errorWorkflow` = **Alerta WhatsApp** (ID: `ktpi4gsUslWWAWyG`).
- Avisos de negócio (ex.: "lead não encontrado") → Execute Workflow do Alerta WhatsApp
  com `{ titulo, mensagem, link }`.
- HTTP externo, Google e IA: `retryOnFail`, 3 tentativas, 2000 ms.
- Timezone dos fluxos: `America/Sao_Paulo`.
- Nenhum segredo no repo. Credenciais por nome/ID. O `.env` (fora do git) guarda tokens
  só para consultas locais do Claude.
- O MCP não cria credenciais: se faltar, dizer exatamente qual criar (tipo e nome).
- Commits: `<tipo>: <descrição>` (feat, fix, docs, chore, test, refactor).

## Credenciais conhecidas

| Uso | Nome | Tipo | ID |
|---|---|---|---|
| Google Drive (conta Sara) | `sara` | googleDriveOAuth2Api | `Xz7XkQuj0KDkVtW5` |
| Google Agenda (conta Sara) | `sara` | googleCalendarOAuth2Api | `cL4FeJhik5kcbnJB` |
| Clint CRM | `Header Clint` | httpHeaderAuth | `qiMe1q95EEa3L2HI` |
| OpenAI | `grupotoda` | openAiApi | `ynBZoBP7TIG7AJAZ` |
| Evolution API (alertas) | `evolution_valeng` — **chave errada (401), trocar** | httpHeaderAuth | `dpb9deNkG8TenXdV` |

## Clint CRM (API)

- Base `https://api.clint.digital`, header `api-token`. Docs: https://clint-api.readme.io
- Deal: `GET /v1/deals/{id}`, `GET /v1/deals?phone=&email=&status=OPEN`, `POST /v1/deals/{id}`.
- Campos personalizados: `GET /v1/account/fields`. No deal ficam planos em
  `fields.{chave}`; a chave é o rótulo em snake_case truncado em 20 caracteres.
- Tipo `RICH_TEXT` guarda HTML (`<p>`, `<strong>`, `<br>`, `<a>`).
- Atualizar campo: `POST /v1/deals/{id}` com `{"fields":{"<chave>":"..."}}` — altera só
  essa chave, os demais campos ficam intactos.
- **A API corrompe caracteres não-ASCII** (acentos, emoji viram `�`), mesmo com `\uXXXX`.
  Em campos RICH_TEXT, enviar tudo que não for ASCII como entidade HTML (`&#231;`).
- Para "apagar" um RICH_TEXT, enviar `<p></p>` (`""` e `null` são ignorados).
- Links de deal aparecem como `app.clint.digital/deal/{uuid}` **ou** `app.useclint.com/deal/{uuid}`.
- A API não cria campos nem envia anexos.
- Deal de teste: `40ab3b02-cadf-419d-a842-d8870204971e` (contato Régis Marques).

## Fluxos

| Fluxo | Pasta | ID n8n |
|---|---|---|
| Alerta WhatsApp | `fluxos/alerta-whatsapp` | `ktpi4gsUslWWAWyG` |
| Anotação Reuniao Meet | `fluxos/meet-para-clint` | `Owu9NI5ScaU5zZfA` |
